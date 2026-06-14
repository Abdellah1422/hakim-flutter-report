# Hakim — Flutter Implementation Guide
## Connecting the Mobile App to the New PDF/WhatsApp Backend Flow

> Owner: Flutter team (App/)
> Goal: Give the doctor an "Approve" + "Finalize & Send" experience that
> triggers the backend pipeline (PDF generation + WhatsApp delivery) and
> shows clear feedback (loading, success, error) at every step.

---

## Table of Contents

1. [How the Flow Works](#how-the-flow-works)
2. [Status Machine the UI must respect](#status-machine-the-ui-must-respect)
3. [API Service Methods](#1-api-service-methods)
4. [Result Model](#2-result-model)
5. [ViewModel Changes](#3-viewmodel-changes)
6. [Pre-flight Validation](#4-pre-flight-validation)
7. [Confirmation Dialog Widget](#5-confirmation-dialog-widget)
8. [Action Bar Widget](#6-action-bar-widget)
9. [Wiring it into the Report Review Screen](#7-wiring-it-into-the-report-review-screen)
10. [Testing the Flow](#8-testing-the-flow)
11. [Common Issues](#9-common-issues)
12. [Optional Enhancements](#10-optional-enhancements)

---

## How the Flow Works

```
Doctor opens a draft report
   ↓
Doctor edits diagnosis / medications / etc.
   ↓
Doctor taps "Approve Review"          → PATCH /reports/medical-reports/{id}/status
   (status: DRAFT → REVIEWED)
   ↓
Doctor taps "Finalize & Send to Patient" → POST /reports/medical-reports/{id}/finalize
   ↓
Confirmation dialog appears
   ↓
On confirm: spinner + Backend generates PDF + sends to WhatsApp
   ↓
Success: bottom sheet with confirmation; report becomes FINALIZED
Failure: error dialog with the reason; status unchanged
```

The Flutter app never talks to WhatsApp directly. It only calls the backend.

---

## Status Machine the UI must respect

| Report status | "Approve Review" button | "Finalize & Send" button | Notes |
|---|---|---|---|
| `DRAFT` | enabled | disabled | Doctor still editing |
| `REVIEWED` | hidden | enabled | Ready to send |
| `APPROVED` (transient) | hidden | hidden + spinner | Backend in middle of pipeline |
| `FINALIZED` | hidden | hidden | Show green "Sent" banner instead |
| `CANCELLED` | hidden | hidden | Show muted "Cancelled" banner |

---

## 1) API Service Methods

Edit `lib/services/API_Service.dart` and add these two static methods:

```dart
/// Mark a report as REVIEWED (doctor approved the AI draft).
/// Backend transitions: DRAFT → REVIEWED.
static Future<Map<String, dynamic>?> markReportReviewed(int reportId) async {
  try {
    final response = await dio.patch(
      '/reports/medical-reports/$reportId/status',
      data: {'status': 'REVIEWED'},
    );
    return response.statusCode == 200
        ? response.data as Map<String, dynamic>
        : null;
  } on DioException catch (e) {
    debugPrint('markReportReviewed error: ${e.response?.data ?? e.message}');
    rethrow;
  }
}

/// Finalize the report and send it to the patient via WhatsApp.
/// Backend transitions: REVIEWED → APPROVED → FINALIZED,
/// generates the branded PDF, and posts it to WhatsApp Cloud API.
static Future<FinalizeResult> finalizeAndSendReport(int reportId) async {
  try {
    final response = await dio.post(
      '/reports/medical-reports/$reportId/finalize',
    );
    if (response.statusCode == 200) {
      return FinalizeResult.fromJson(response.data as Map<String, dynamic>);
    }
    return FinalizeResult.failure('Unexpected status: ${response.statusCode}');
  } on DioException catch (e) {
    final detail = e.response?.data?['detail']?.toString()
                   ?? e.message
                   ?? 'Network error';
    debugPrint('finalize error: $detail');
    return FinalizeResult.failure(detail);
  }
}
```

---

## 2) Result Model

Add to `lib/model/finalize_result.dart`:

```dart
class FinalizeResult {
  final bool success;
  final String? errorMessage;
  final String? pdfUrl;
  final String? whatsappMessageId;

  FinalizeResult({
    required this.success,
    this.errorMessage,
    this.pdfUrl,
    this.whatsappMessageId,
  });

  factory FinalizeResult.fromJson(Map<String, dynamic> json) => FinalizeResult(
        success: json['success'] == true,
        pdfUrl: json['report']?['pdf_url'],
        whatsappMessageId: json['report']?['whatsapp_message_id'],
      );

  factory FinalizeResult.failure(String msg) =>
      FinalizeResult(success: false, errorMessage: msg);
}
```

---

## 3) ViewModel Changes

In `lib/viewmodels/doctor_viewmodel.dart` add these fields and methods:

```dart
// ----- State flags to disable buttons / show spinners -----
bool _isMarkingReviewed = false;
bool _isFinalizing = false;
bool get isMarkingReviewed => _isMarkingReviewed;
bool get isFinalizing => _isFinalizing;

/// Mark report as reviewed. Returns true on success.
Future<bool> markReportReviewed(int reportId) async {
  if (_isMarkingReviewed) return false;
  _isMarkingReviewed = true;
  notifyListeners();
  try {
    final result = await ApiService.markReportReviewed(reportId);
    if (result != null) {
      _updateReportInList(reportId, status: 'REVIEWED');
      return true;
    }
    return false;
  } catch (e) {
    debugPrint('markReviewed failed: $e');
    return false;
  } finally {
    _isMarkingReviewed = false;
    notifyListeners();
  }
}

/// Finalize and send via WhatsApp.
Future<FinalizeResult> finalizeAndSendReport(int reportId) async {
  if (_isFinalizing) return FinalizeResult.failure('Already in progress');
  _isFinalizing = true;
  notifyListeners();
  try {
    final result = await ApiService.finalizeAndSendReport(reportId);
    if (result.success) {
      _updateReportInList(reportId, status: 'FINALIZED');
    }
    return result;
  } finally {
    _isFinalizing = false;
    notifyListeners();
  }
}

/// Update the cached report in the in-memory list after a status change.
void _updateReportInList(int reportId, {required String status}) {
  final idx = _reports.indexWhere((r) => r.id == reportId);
  if (idx != -1) {
    _reports[idx] = _reports[idx].copyWith(status: status);
  }
}
```

> 📝 Make sure your `MedicalReport` model has a `copyWith` method that accepts a `status` parameter.

---

## 4) Pre-flight Validation

Validates everything is in place before the doctor even sees the confirmation dialog.

Create `lib/utils/report_validation.dart`:

```dart
class ReportValidation {
  final bool valid;
  final List<String> issues;
  ReportValidation(this.valid, this.issues);
}

ReportValidation validateForSending({
  required String? patientPhone,
  required String? diagnosis,
  required dynamic medications,
  required String reportStatus,
}) {
  final issues = <String>[];

  if (patientPhone == null || patientPhone.trim().length < 10) {
    issues.add('رقم المريض غير مسجل أو غير صالح');
  }
  if (diagnosis == null || diagnosis.trim().isEmpty) {
    issues.add('التشخيص فارغ');
  }
  if (reportStatus != 'REVIEWED') {
    issues.add('لازم تراجع التقرير وتعتمده الأول');
  }
  return ReportValidation(issues.isEmpty, issues);
}
```

---

## 5) Confirmation Dialog Widget

Create `lib/widgets/doctor/send_report_dialog.dart`:

```dart
import 'package:flutter/material.dart';

class SendReportConfirmDialog extends StatelessWidget {
  final String patientName;
  final String patientPhone;
  final String doctorName;

  const SendReportConfirmDialog({
    super.key,
    required this.patientName,
    required this.patientPhone,
    required this.doctorName,
  });

  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      icon: const Icon(Icons.send_outlined, color: Color(0xFF1B6E8C), size: 40),
      title: const Text('تأكيد إرسال التقرير'),
      content: Column(
        mainAxisSize: MainAxisSize.min,
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          _row('المريض', patientName),
          const SizedBox(height: 6),
          _row('الرقم', patientPhone),
          const SizedBox(height: 6),
          _row('الطبيب', 'د. $doctorName'),
          const SizedBox(height: 16),
          Container(
            padding: const EdgeInsets.all(10),
            decoration: BoxDecoration(
              color: Colors.amber.shade50,
              borderRadius: BorderRadius.circular(6),
              border: Border.all(color: Colors.amber.shade300),
            ),
            child: const Row(
              children: [
                Icon(Icons.info_outline, color: Color(0xFFB45309), size: 18),
                SizedBox(width: 8),
                Expanded(
                  child: Text(
                    'سيتم توليد PDF بتقرير المريض وإرساله على واتساب فوراً. '
                    'لا يمكن التراجع بعد الإرسال.',
                    style: TextStyle(fontSize: 12, color: Color(0xFF92400E)),
                  ),
                ),
              ],
            ),
          ),
        ],
      ),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(context, false),
          child: const Text('إلغاء'),
        ),
        FilledButton.icon(
          icon: const Icon(Icons.send, size: 18),
          label: const Text('إرسال الآن'),
          style: FilledButton.styleFrom(backgroundColor: const Color(0xFF1B6E8C)),
          onPressed: () => Navigator.pop(context, true),
        ),
      ],
    );
  }

  Widget _row(String key, String val) => Row(
        children: [
          SizedBox(width: 60, child: Text(key, style: const TextStyle(color: Colors.grey))),
          Expanded(child: Text(val, style: const TextStyle(fontWeight: FontWeight.w600))),
        ],
      );
}
```

---

## 6) Action Bar Widget

Create `lib/widgets/doctor/report_actions_bar.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../../viewmodels/doctor_viewmodel.dart';
import '../../model/medical_report.dart';
import '../../model/finalize_result.dart';
import '../../utils/report_validation.dart';
import 'send_report_dialog.dart';

class ReportActionsBar extends StatelessWidget {
  final MedicalReport report;
  const ReportActionsBar({super.key, required this.report});

  @override
  Widget build(BuildContext context) {
    final vm = context.watch<DoctorViewModel>();
    final isDraft     = report.status == 'DRAFT';
    final isReviewed  = report.status == 'REVIEWED';
    final isFinalized = report.status == 'FINALIZED';

    return Container(
      padding: const EdgeInsets.all(16),
      decoration: BoxDecoration(
        color: Colors.white,
        boxShadow: [BoxShadow(color: Colors.black.withOpacity(0.05), blurRadius: 8)],
      ),
      child: SafeArea(
        top: false,
        child: isFinalized
            ? _finalizedBanner()
            : Row(
                children: [
                  if (isDraft)
                    Expanded(
                      child: OutlinedButton.icon(
                        onPressed: vm.isMarkingReviewed
                            ? null
                            : () => _onMarkReviewed(context, vm),
                        icon: vm.isMarkingReviewed
                            ? const SizedBox(
                                width: 16, height: 16,
                                child: CircularProgressIndicator(strokeWidth: 2),
                              )
                            : const Icon(Icons.check_circle_outline),
                        label: const Text('اعتماد المراجعة'),
                        style: OutlinedButton.styleFrom(
                          padding: const EdgeInsets.symmetric(vertical: 16),
                          foregroundColor: const Color(0xFF2A9D8F),
                          side: const BorderSide(color: Color(0xFF2A9D8F)),
                        ),
                      ),
                    ),
                  if (isDraft) const SizedBox(width: 12),

                  Expanded(
                    flex: 2,
                    child: FilledButton.icon(
                      onPressed: (isReviewed && !vm.isFinalizing)
                          ? () => _onFinalize(context, vm)
                          : null,
                      icon: vm.isFinalizing
                          ? const SizedBox(
                              width: 16, height: 16,
                              child: CircularProgressIndicator(
                                strokeWidth: 2, color: Colors.white,
                              ),
                            )
                          : const Icon(Icons.send),
                      label: Text(vm.isFinalizing ? 'جارٍ الإرسال...' : 'إنهاء وإرسال للمريض'),
                      style: FilledButton.styleFrom(
                        backgroundColor: const Color(0xFF1B6E8C),
                        padding: const EdgeInsets.symmetric(vertical: 16),
                      ),
                    ),
                  ),
                ],
              ),
      ),
    );
  }

  Future<void> _onMarkReviewed(BuildContext context, DoctorViewModel vm) async {
    final ok = await vm.markReportReviewed(report.id);
    if (!context.mounted) return;
    ScaffoldMessenger.of(context).showSnackBar(SnackBar(
      content: Text(ok ? '✓ تم اعتماد المراجعة' : '✗ فشل، حاول مرة أخرى'),
      backgroundColor: ok ? Colors.green.shade700 : Colors.red.shade700,
    ));
  }

  Future<void> _onFinalize(BuildContext context, DoctorViewModel vm) async {
    final validation = validateForSending(
      patientPhone: report.patientPhone,
      diagnosis: report.aiDiagnosis,
      medications: report.aiMedications,
      reportStatus: report.status,
    );
    if (!validation.valid) {
      _showValidationErrors(context, validation.issues);
      return;
    }

    final confirmed = await showDialog<bool>(
      context: context,
      builder: (_) => SendReportConfirmDialog(
        patientName:  report.patientName,
        patientPhone: report.patientPhone ?? '—',
        doctorName:   report.doctorName,
      ),
    );
    if (confirmed != true) return;

    final result = await vm.finalizeAndSendReport(report.id);
    if (!context.mounted) return;

    if (result.success) {
      _showSuccessSheet(context, result);
    } else {
      _showErrorDialog(context, result.errorMessage ?? 'حصل خطأ غير معروف');
    }
  }

  void _showValidationErrors(BuildContext context, List<String> issues) {
    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        icon: const Icon(Icons.warning_amber_rounded, color: Colors.orange, size: 36),
        title: const Text('مش جاهز للإرسال'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          crossAxisAlignment: CrossAxisAlignment.start,
          children: issues.map((e) => Padding(
            padding: const EdgeInsets.symmetric(vertical: 4),
            child: Row(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                const Text('• ', style: TextStyle(color: Colors.orange)),
                Expanded(child: Text(e)),
              ],
            ),
          )).toList(),
        ),
        actions: [TextButton(onPressed: () => Navigator.pop(context), child: const Text('حسناً'))],
      ),
    );
  }

  void _showSuccessSheet(BuildContext context, FinalizeResult res) {
    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      backgroundColor: Colors.white,
      shape: const RoundedRectangleBorder(
        borderRadius: BorderRadius.vertical(top: Radius.circular(20)),
      ),
      builder: (_) => Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Container(
              width: 64, height: 64,
              decoration: const BoxDecoration(
                color: Color(0xFF2A9D8F), shape: BoxShape.circle,
              ),
              child: const Icon(Icons.check, color: Colors.white, size: 36),
            ),
            const SizedBox(height: 16),
            const Text('تم الإرسال بنجاح',
                style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
            const SizedBox(height: 8),
            const Text(
              'تقرير المريض تم إرساله على واتساب وسيصل خلال ثوانٍ.',
              textAlign: TextAlign.center,
              style: TextStyle(color: Colors.grey),
            ),
            if (res.whatsappMessageId != null) ...[
              const SizedBox(height: 12),
              Text(
                'Msg ID: ${res.whatsappMessageId!.substring(0, 24)}...',
                style: const TextStyle(
                    fontSize: 11, fontFamily: 'monospace', color: Colors.grey),
              ),
            ],
            const SizedBox(height: 24),
            SizedBox(
              width: double.infinity,
              child: FilledButton(
                onPressed: () {
                  Navigator.pop(context);
                  Navigator.pop(context);
                },
                style: FilledButton.styleFrom(
                  backgroundColor: const Color(0xFF1B6E8C),
                  padding: const EdgeInsets.symmetric(vertical: 14),
                ),
                child: const Text('تمام'),
              ),
            ),
          ],
        ),
      ),
    );
  }

  void _showErrorDialog(BuildContext context, String msg) {
    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        icon: const Icon(Icons.error_outline, color: Colors.red, size: 36),
        title: const Text('فشل الإرسال'),
        content: Text(msg, style: const TextStyle(fontSize: 13)),
        actions: [
          TextButton(onPressed: () => Navigator.pop(context), child: const Text('حسناً')),
        ],
      ),
    );
  }

  Widget _finalizedBanner() => Container(
        padding: const EdgeInsets.all(14),
        decoration: BoxDecoration(
          color: const Color(0xFFE8F5E9),
          borderRadius: BorderRadius.circular(8),
        ),
        child: const Row(
          children: [
            Icon(Icons.check_circle, color: Color(0xFF2A9D8F)),
            SizedBox(width: 10),
            Expanded(
              child: Text(
                'هذا التقرير منتهي وتم إرساله للمريض',
                style: TextStyle(fontWeight: FontWeight.w600, color: Color(0xFF1B6E8C)),
              ),
            ),
          ],
        ),
      );
}
```

---

## 7) Wiring it into the Report Review Screen

In your existing report-review screen (likely `lib/views/doctor/consultation/`), make sure of two things:

**(a) Wrap the screen in a `Scaffold` with the action bar at the bottom:**

```dart
@override
Widget build(BuildContext context) {
  return Scaffold(
    appBar: AppBar(title: Text('تقرير #${report.id}')),
    body: SingleChildScrollView(
      padding: const EdgeInsets.all(16),
      child: Column(
        children: [
          // ... existing fields (diagnosis editor, medications, recommendations) ...
        ],
      ),
    ),
    bottomNavigationBar: ReportActionsBar(report: report),
  );
}
```

**(b) After any field edit, persist via the existing PATCH endpoint before the doctor approves**, so the backend has the doctor's final wording. Your existing `update_report` flow is enough for this — just call it on field-blur or on a "Save Draft" button.

---

## 8) Testing the Flow

| # | Step | Expected result |
|---|---|---|
| 1 | Open a report with status `DRAFT` | Two buttons visible: "اعتماد المراجعة" (enabled) and "إنهاء وإرسال" (disabled) |
| 2 | Tap "اعتماد المراجعة" | Spinner briefly, then green snackbar "تم اعتماد المراجعة". Status becomes `REVIEWED` |
| 3 | Tap "إنهاء وإرسال" with no patient phone | Validation dialog opens with "رقم المريض غير مسجل" |
| 4 | Tap "إنهاء وإرسال" with valid data | Confirmation dialog opens showing patient name + phone + doctor |
| 5 | Tap "إلغاء" in the dialog | Dialog closes, nothing sent |
| 6 | Tap "إرسال الآن" | Button shows spinner with text "جارٍ الإرسال..." |
| 7 | On success | Bottom sheet appears with "تم الإرسال بنجاح" + WhatsApp message ID |
| 8 | Re-open the same report | Green banner "هذا التقرير منتهي وتم إرساله للمريض" instead of buttons |
| 9 | Check the patient's WhatsApp | PDF document arrives with the approved template message |

---

## 9) Common Issues

| Problem | Cause | Fix |
|---|---|---|
| Buttons stay disabled forever | `report.status` isn't refreshing | Verify `_updateReportInList` is called and the ViewModel `notifyListeners()` |
| Snackbar shows but report still says DRAFT | Backend call succeeded but local cache not updated | Re-fetch the report from the server after success |
| 401 Unauthorized on `/finalize` | JWT expired | Refresh the token via your existing auth refresh flow, then retry |
| 403 Forbidden | Logged-in user isn't a doctor | Verify the role in JWT claims; backend requires `RequireDoctor` |
| 400 — "Report must be REVIEWED" | Tapped Finalize without first approving | Re-tap "اعتماد المراجعة" first |
| Network error in confirmation flow | Slow 4G / proxy timeout | Increase Dio `receiveTimeout` to 30 s for this endpoint |
| Success snackbar but patient didn't receive | Test number, not added in Meta dashboard | Add the recipient phone in Meta WhatsApp dashboard |
| Hot-reload loses report state | ChangeNotifierProvider rebuilt | Use `Provider.value(value: existingVM)` when navigating |

---

## 10) Optional Enhancements

These improve the UX but are not strictly required.

### (a) "Preview PDF" before sending

If the backend's finalize response includes `pdf_url`, you can let the doctor preview it before the final tap.

Add a "Preview" button next to "Finalize" that calls a separate `/preview-pdf` endpoint (backend would generate the PDF without sending WhatsApp). Open the URL in the in-app webview using `url_launcher`.

### (b) Retry mechanism for failed sends

When `finalizeAndSendReport` fails specifically at the WhatsApp step (PDF generated but WhatsApp rejected), offer a "Retry send" button on the report screen. This requires a separate backend endpoint like `POST /reports/{id}/resend-whatsapp`.

### (c) Live delivery status

After sending, poll the backend every few seconds for the WhatsApp delivery status (sent / delivered / read) and show a small icon next to the report. The backend can listen to Meta's webhook for status updates and store them in a new `report_deliveries` table.

### (d) Multi-channel fallback

If the patient phone is invalid for WhatsApp, gracefully fall back to SMS or email by showing the doctor a choice. Requires backend support for those channels.

### (e) Localization toggle

The current dialog text is hard-coded Arabic. Move strings to an `AppLocalizations` file (using `flutter_localizations` + `intl` package) so the UI can switch to English for non-Arabic-speaking doctors.

---

**Done.** Once these changes are merged, the doctor's "Finalize & Send" tap drives the entire backend pipeline, and the patient receives a branded PDF on WhatsApp in seconds. Test the happy path end-to-end on a real phone before demoing.
