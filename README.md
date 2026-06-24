# Deployment Guide — Intelligent Medical Assistant (ASR API)

تطبيق **FastAPI** بيستقبل ملف صوتي لحوار طبي، يحوّله نص بـ **faster-whisper** (لهجة مصرية)، وبعدين ينظّمه في تقرير طبي JSON عن طريق **Google Gemini**.

> الـ entrypoint بتاع التطبيق: `hakim_optimized:app`

---

## 1) متطلبات السيرفر (System Requirements)

| البند | المطلوب |
|------|---------|
| نظام التشغيل | Linux (Ubuntu 22.04+ يفضّل) أو أي OS بيشغّل Python |
| Python | **3.11** |
| المعالج | CPU عادي (مفيش حاجة اسمها GPU مطلوبة — الموديل بيشتغل CPU/int8) |
| RAM | **2 GB على الأقل** لكل worker (الموديل بيتحمّل في الذاكرة لكل worker) |
| Disk | ~1.5 GB (الـ dependencies + موديل Whisper ~466 MB) |
| Internet | مطلوب وقت التثبيت + أول تشغيل (لتحميل الموديل)، ومطلوب دايمًا للاتصال بـ Gemini |
| System lib | `libgomp1` (مكتبة OpenMP اللي بيحتاجها CTranslate2 على Linux) |

> **مهم:** التطبيق **مش محتاج `torch`** ولا GPU. faster-whisper بيشتغل على CTranslate2.
> وكمان مش محتاج تثبّت `ffmpeg` منفصل — بيتعامل مع الصوت عن طريق PyAV المدمجة.

---

## 2) المتغيّرات البيئية (Environment Variables)

التطبيق بيقرأ ملف `.env` بـ `load_dotenv(override=True)` — يعني قيم `.env` **بتغلب** أي متغيّرات موجودة في النظام.
انسخ `.env.example` لـ `.env` واملا القيم:

| المتغير | إلزامي؟ | الشرح |
|---------|---------|-------|
| `GOOGLE_API_KEY` | ✅ **إلزامي** | مفتاح Gemini. **لازم يكون مفتاح مدفوع (billing-enabled)** في الإنتاج عشان نتجنّب أخطاء `503 high demand` المتكررة بتاعة الـ free tier. |
| `BACKEND_API_KEY` | ✅ **إلزامي** | السر اللي العملاء لازم يبعتوه في الـ header `X-API-KEY`. حُط قيمة عشوائية طويلة (مش `change-me`). |
| `WHISPER_MODEL` | اختياري | اسم موديل الـ ASR. الافتراضي: `moeshawky/faster-whisper-small-egyptian-arabic` |
| `WHISPER_BEAM` | اختياري | `1` = أسرع (افتراضي)، `2` = أدق وأبطأ |
| `DB_*` | اختياري | مش مستخدمة حاليًا في الـ endpoint. سيبها فاضية. |

> ⚠️ **أمان:** متحطّش `.env` ولا أي مفتاح في الـ git repo. المفتاح القديم اللي اتسرّب قبل كده اتعطّل من جوجل — استخدم مفتاح جديد بس.

---

## 3) التثبيت (Installation)

```bash
# 1. system lib مطلوبة لـ CTranslate2
sudo apt-get update && sudo apt-get install -y libgomp1

# 2. اعمل virtual environment بـ Python 3.11
python3.11 -m venv .venv
source .venv/bin/activate

# 3. ثبّت الـ dependencies
pip install --upgrade pip
pip install -r requirements.txt

# 4. جهّز ملف البيئة
cp .env.example .env
# ... عدّل .env وحُط فيه GOOGLE_API_KEY و BACKEND_API_KEY ...
```

أول تشغيل هيحمّل موديل Whisper (~466 MB) من Hugging Face ويخزّنه في الكاش (`~/.cache/huggingface`). ده بيحصل **مرة واحدة بس**.

---

## 4) التشغيل في الإنتاج (Run in Production)

**متستخدمش `--reload` في الإنتاج.** استخدم gunicorn مع uvicorn workers:

```bash
gunicorn hakim_optimized:app \
  -k uvicorn.workers.UvicornWorker \
  --workers 1 \
  --bind 0.0.0.0:8000 \
  --timeout 120
```

> **عدد الـ workers:** كل worker بيحمّل نسخة من موديل Whisper في الذاكرة (~1–2 GB). زوّد `--workers` على حسب الـ RAM المتاحة فقط. ابدأ بـ `1` وزوّد بحذر.
> **`--timeout 120`** مهم لأن الـ transcription + Gemini ممكن ياخدوا وقت على الملفات الكبيرة.

أو للتشغيل البسيط بـ uvicorn مباشرة:
```bash
uvicorn hakim_optimized:app --host 0.0.0.0 --port 8000 --workers 1
```

### Reverse proxy (Nginx) — اختياري لكن مُوصى به
حُط Nginx قدام التطبيق للـ HTTPS ورفع حد حجم الـ upload:
```nginx
server {
    listen 80;
    server_name your-domain.com;

    client_max_body_size 50M;   # اسمح بملفات صوتية أكبر

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 120s;
    }
}
```

### تشغيل دائم بـ systemd — اختياري
ملف `/etc/systemd/system/asr-api.service`:
```ini
[Unit]
Description=ASR Medical Report API
After=network.target

[Service]
User=www-data
WorkingDirectory=/opt/asr-api
EnvironmentFile=/opt/asr-api/.env
ExecStart=/opt/asr-api/.venv/bin/gunicorn hakim_optimized:app -k uvicorn.workers.UvicornWorker --workers 1 --bind 0.0.0.0:8000 --timeout 120
Restart=always

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now asr-api
```

---

## 5) الـ API

### Endpoint
```
POST /transcribe-report
```

| العنصر | القيمة |
|--------|--------|
| Header | `X-API-KEY: <BACKEND_API_KEY>` |
| Body | `multipart/form-data` بحقل اسمه `file` (ملف صوتي/فيديو) |
| Response | `200` JSON بالتقرير الطبي / `401` مفتاح غلط / `422` مفيش صوت / `500` خطأ معالجة |

### مثال اختبار
```bash
curl -X POST "http://SERVER:8000/transcribe-report" \
  -H "X-API-KEY: YOUR_BACKEND_API_KEY" \
  -F "file=@test.ogg"
```

### شكل الـ Response
```json
{
  "diagnosis": "...",
  "medications": [
    { "name": "Augmentin", "dose": "...", "frequency": "...", "duration": "...", "notes": "..." }
  ],
  "recommendations": ["..."],
  "follow_up": "..."
}
```

### توثيق تفاعلي
بعد التشغيل: `http://SERVER:8000/docs` (Swagger UI).

---

## 6) ملاحظات مهمة للـ DevOps

- **CORS:** حاليًا `allow_origins=["*"]` (مفتوح للكل) في الكود. ضيّقها لدومين الـ frontend بتاعك قبل الإنتاج.
- **خطأ 503 من Gemini:** الكود فيه retries + موديل احتياطي (`gemini-2.0-flash`). المفتاح المدفوع بيقلّل المشكلة دي جدًا.
- **الكاش:** خلّي مجلد `~/.cache/huggingface` ثابت (مش بيتمسح مع كل deploy) عشان متعيدش تحميل الموديل. لو Docker، اعمله volume أو حمّل الموديل وقت الـ build.
- **Health check:** ممكن تستخدم `GET /docs` كـ liveness probe بسيط لحد ما يتعمل endpoint مخصص.

---

## الملفات المرفقة
- `hakim_optimized.py` — كود التطبيق
- `requirements.txt` — الـ dependencies بنسخ مثبّتة
- `.env.example` — قالب المتغيّرات البيئية
