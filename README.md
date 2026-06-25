# MedVision AI — Deployment Guide

FastAPI service for medical image analysis (X-ray + skin) using Gemini Vision + a hybrid
RAG (ChromaDB semantic + BM25 lexical) over WHO/CDC guidelines, built-in knowledge, and PDFs.

This document is for the DevOps engineer deploying the service. It covers prerequisites,
configuration, running in production, the reverse-proxy setup, and the important gotchas.

---

## 1. TL;DR

```bash
# 1. Python 3.10+ and system libs
sudo apt-get install -y python3-venv build-essential

# 2. Create venv + install deps
python3 -m venv .venv
.venv/bin/pip install --upgrade pip
.venv/bin/pip install -r requirements.txt

# 3. Configure secrets
cp .env.example .env        # then edit .env with the real keys

# 4. First boot — builds the RAG index (needs internet, ~1–2 min). Run with ONE worker.
.venv/bin/python -m uvicorn main:app --host 0.0.0.0 --port 8000

# 5. Verify
curl http://localhost:8000/health
```

For production, run it under **systemd** or **Docker** (sections 6–7) behind **nginx**
with a **long read timeout** (section 8 — this is the #1 thing that breaks deployments).

---

## 2. What gets deployed

| Path | Commit to git? | Notes |
|------|----------------|-------|
| `main.py` | ✅ | The whole app (single file). Entry point: `main:app`. |
| `requirements.txt` | ✅ | Python dependencies. |
| `medical_pdfs/` | ✅ | 11 source PDFs (skin + chest). Chunked into the RAG index at first boot. |
| `.env.example` | ✅ | Template for required env vars. |
| `.env` | ❌ **never** | Real secrets. Create on the server only. |
| `chroma_db/` | ❌ | Persistent vector store, **generated** at first boot. |
| `chunks.json` | ❌ | Generated chunk dump, regenerated at first boot. |

`.gitignore` already excludes `.env`, `chroma_db/`, `chunks.json`, logs, and venvs.

---

## 3. Prerequisites

- **Python 3.10+** (developed/tested on 3.13).
- **OS packages:** `python3-venv`, `build-essential` (some wheels compile native code).
- **CPU-only is fine.** No GPU required — `torch` runs on CPU. (A GPU works too but isn't needed.)
- **Outbound internet access (egress).** The service makes outbound HTTPS calls to:
  | Host | Purpose | Required |
  |------|---------|----------|
  | `generativelanguage.googleapis.com` | Gemini model calls | **Yes** (core feature) |
  | `huggingface.co` / `*.hf.co` | Download the embedding model on first boot | **Yes** (first boot only) |
  | `eutils.ncbi.nlm.nih.gov` | PubMed evidence lookup | Optional (degrades gracefully if blocked) |

  > If `huggingface.co` is blocked, pre-bake the model into the image / HF cache
  > (`~/.cache/huggingface`) so first boot doesn't need it.

---

## 4. Configuration (`.env`)

Two variables, both **required** — the app refuses to start if either is missing:

```ini
GEMINI_API_KEY=...          # Google AI Studio key
ENDPOINT_API_KEY=...        # secret clients send in the X-API-Key header
```

> 🔐 Generate a strong `ENDPOINT_API_KEY` (e.g. `openssl rand -hex 32`). The current dev
> value must be rotated for production. The Gemini key should also be rotated, as the dev
> key was shared during development.

---

## 5. Resource sizing

Each **worker process** loads its own copy of torch + the embedding model into memory.

- **RAM:** ~1.5–2.5 GB **per worker**. Size the box for `workers × 2 GB` + headroom.
- **Disk:** ~4 MB (`chroma_db/`) + ~100 MB (HuggingFace model cache) + deps (~few GB incl. torch).
- **CPU:** embedding + retrieval are light for the current 118-chunk index; the wall-clock time
  of a request is dominated by the remote Gemini calls, not local CPU.

---

## 6. Running in production — systemd (recommended, simplest)

`/etc/systemd/system/medvision.service`:

```ini
[Unit]
Description=MedVision AI API
After=network.target

[Service]
User=medvision
WorkingDirectory=/opt/medvision
EnvironmentFile=/opt/medvision/.env
# 2 workers ≈ 2 concurrent heavy requests. Increase only if RAM allows (≈2 GB/worker).
ExecStart=/opt/medvision/.venv/bin/python -m uvicorn main:app \
          --host 0.0.0.0 --port 8000 --workers 2 --timeout-keep-alive 120
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now medvision
sudo systemctl status medvision
journalctl -u medvision -f      # logs
```

> **Build the index once before scaling.** On a fresh server, start with `--workers 1`
> the first time so a single process builds `chroma_db/`. Once it logs
> `✅ Built RAG (...)`, stop and restart with multiple workers — they will all reuse the
> persisted index (`✅ Loaded persisted RAG (...)`) and skip re-embedding.

---

## 7. Running in production — Docker (alternative)

`Dockerfile`:

```dockerfile
FROM python:3.13-slim

RUN apt-get update && apt-get install -y --no-install-recommends build-essential \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY main.py .
COPY medical_pdfs ./medical_pdfs

EXPOSE 8000
CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", \
     "--workers", "2", "--timeout-keep-alive", "120"]
```

```bash
docker build -t medvision:latest .
docker run -d --name medvision -p 8000:8000 \
  --env-file .env \
  -v medvision_chroma:/app/chroma_db \      # persist the index across restarts
  medvision:latest
```

> Mounting a volume at `/app/chroma_db` keeps the built index so containers don't
> re-embed on every restart. The HuggingFace model cache can likewise be volume-mounted
> at `/root/.cache/huggingface` to avoid re-downloading the embedding model.

---

## 8. Reverse proxy (nginx) — ⚠️ critical timeouts

**The `/analyze` endpoint takes 40–70 seconds** (two Gemini vision calls, including model
"thinking" time, plus retrieval). The default nginx `proxy_read_timeout` is **60s**, which
will cut long requests off with a 504. **You must raise it.**

```nginx
server {
    listen 80;
    server_name medvision.example.com;

    # Images can be up to 15 MB (the app rejects larger with 413)
    client_max_body_size 20m;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # MUST be generous — Gemini calls are slow
        proxy_read_timeout 180s;
        proxy_send_timeout 180s;
    }
}
```

If there's also a cloud load balancer in front, raise **its** idle/read timeout to ≥120s too.

> CORS is handled inside the app (`allow_origins=["*"]`), so nginx does not need CORS rules.

---

## 9. API surface

| Method | Path | Auth | Notes |
|--------|------|------|-------|
| `GET`  | `/health` | none | Liveness/readiness probe. Returns `{status, model_chain, rag_ready}`. Use for health checks. |
| `POST` | `/analyze` | `X-API-Key` header | `multipart/form-data`: `image` (file, ≤15 MB), `patient_name`, `patient_age`. Returns the structured report + RAG sources. |

Health check example for orchestrators:

```bash
curl -fsS http://localhost:8000/health || exit 1
```

A request is **ready** to serve traffic only when `/health` returns `"rag_ready": true`
(the index finished loading). On first boot this can take a minute or two.

---

## 10. Operational notes & troubleshooting

- **First boot is slow / makes network calls** — it downloads the embedding model and builds
  the index. Subsequent boots reuse `chroma_db/` and start in seconds.
- **Rebuild the index** (e.g. after changing the PDFs in `medical_pdfs/`): delete `chroma_db/`
  and `chunks.json`, then restart. The app also auto-rebuilds if the chunk count no longer
  matches the persisted collection.
- **Occasional `500` from Gemini overload** — the upstream model sometimes returns 503
  "high demand". The app already waits and retries, then falls through the model chain
  (`gemini-2.5-flash` → `gemini-2.5-pro` → ...). Transient 500s under heavy load are expected;
  clients should retry.
- **PubMed blocked/slow** — handled gracefully: the request still succeeds using local RAG
  context, just without PubMed sources. Timeouts are short (6s/8s).
- **Logs** — `journalctl -u medvision -f` (systemd) or `docker logs -f medvision`.

---

## 11. Pre-go-live checklist

- [ ] `.env` created on the server with **rotated** production keys (not the dev values).
- [ ] `.env` is **not** in version control.
- [ ] First boot completed with 1 worker; `chroma_db/` built and persisted (volume-mounted under Docker).
- [ ] `/health` returns `rag_ready: true`.
- [ ] Reverse proxy `proxy_read_timeout` ≥ 180s and `client_max_body_size` ≥ 20m.
- [ ] Any cloud LB idle timeout raised to ≥ 120s.
- [ ] Outbound HTTPS allowed to Google + HuggingFace (and NCBI if PubMed is wanted).
- [ ] A real `/analyze` call with a valid `X-API-Key` returns `200` end-to-end.
