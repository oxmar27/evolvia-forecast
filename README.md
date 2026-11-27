# Evolvia Nexus

Upload a CSV of numbers and get an **analysis**, a **forecast**, or **both**, explained in plain language. The app tries several methods on your data and keeps the one with the smallest error.

> **Status: early prototype.** It runs locally and has not been deployed. There is no login yet and no automated test suite. Please do not use it with sensitive data. See [Status and roadmap](#status-and-roadmap).

<!-- Add screenshots here, for example: ![Forecast](docs/forecast.png) -->

## What it does

Choose what you need after uploading a file:

| Option | What you get |
|---|---|
| **Analysis only** | Fast. Average, middle value, lowest and highest. The overall trend. Typical weekday and month patterns. Unusual values. What moves together with your numbers. |
| **Forecast only** | A forecast with a shaded range where the real value will likely fall, and the accuracy of the chosen method. |
| **Analysis + forecast** | Both of the above. |

The app looks at the file and picks the path automatically:

- **The file has a date column** (daily, weekly or monthly): time series mode.
  - Small gaps and empty cells are filled by interpolation (up to 10% of the data), and the user is told.
  - It compares **Chronos-2** (a pretrained forecasting model), **Prophet** (two seasonality settings) and a simple "repeat the last pattern" baseline.
  - Each method is tested on up to three past periods ("backtesting"). The method with the smallest average error wins.
- **No date column:** regression mode.
  - It compares an "always the average" baseline, Linear Regression, Random Forest and Gradient Boosting using 5-fold cross-validation.
  - It shows which columns matter most (permutation importance).
  - Rows where the target value is empty are predicted, and the result can be downloaded as CSV.

The interface is available in **English and Arabic** (right-to-left included). Anything that comes from the uploaded file is rendered as plain text, never as HTML.

## How it is built

```
frontend/   plain HTML, CSS and JavaScript (no dependencies). Canvas-based 3D network background and charts.
backend/    Python + FastAPI
  analyzer.py   analysis, model comparison, forecasting
  safety.py     input cleaning, output validation, captcha check
  limits.py     settings from environment variables, rate limiter
  main.py       API endpoints, security headers, error handling
```

Main libraries: FastAPI, pandas, scikit-learn, Prophet, and Chronos-2 (`amazon/chronos-2` through `chronos-forecasting` and PyTorch).

## Run it locally

Tested on macOS (Apple silicon) with Python 3.13.

**1. Backend**

```bash
cd backend
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
ALLOWED_ORIGINS=http://localhost:5173,http://127.0.0.1:5173 uvicorn main:app --port 8000
```

The first analysis downloads the Chronos-2 model (about 478 MB), so it takes a few minutes.

**2. Frontend** (in a second terminal tab)

```bash
cd frontend
python3 -m http.server 5173 --bind 127.0.0.1
```

**3. Open** http://127.0.0.1:5173

**Try it with your own file.** Any CSV works. For a forecast, include a date column and at least one numeric column (for example `date,sales`). If you change the address of the backend, edit `API_URL` in `frontend/config.js`.

## Configuration

Settings are read from environment variables. Copy `backend/.env.example` to `.env` and change the values. Never commit `.env`.

| Variable | Default | Meaning |
|---|---|---|
| `ALLOWED_ORIGINS` | localhost addresses | Websites allowed to call the API (comma separated). Use your own domain in production. |
| `MAX_FILE_MB` | 10 | Largest upload |
| `MAX_ROWS` | 50000 | Most rows per file |
| `RATE_PER_MINUTE` | 20 | Requests per minute per visitor |
| `ANALYSES_PER_DAY` | 30 | Analyses per day per visitor |
| `MAX_PARALLEL_ANALYSES` | 1 | Analyses running at the same time |
| `TRUST_PROXY` | 0 | Set to 1 only behind a trusted proxy |
| `DEBUG` | 0 | Set to 1 to enable the `/docs` page |
| `TURNSTILE_SECRET` | empty | Cloudflare Turnstile secret. When set, `/analyze` requires a captcha token. |

## API

| Endpoint | Purpose |
|---|---|
| `GET /health` | Returns `{"status": "ok"}` |
| `POST /inspect` | Upload a file (`file`). Returns the columns, the numeric columns, the detected date column and the suggested mode. |
| `POST /analyze` | Upload a file (`file`) with `target` (column to predict), `task` (`analysis`, `forecast` or `both`), optional `mode` (`auto`, `time series` or `regression`) and optional `captcha_token`. |

Errors use normal HTTP status codes (400, 413, 429, 503, 500) with a short, generic `error` message. Details go to the server log only.

## Security measures in place

- Upload limits on size and rows, and requests over the limit are refused before they are read.
- The column name and the options are cleaned and checked against fixed lists. The target column must exist in the file.
- The answer is validated before it is sent (allowed types, required fields, list sizes, no `NaN`).
- Rate limiting per visitor, and one analysis at a time.
- CORS allow-list, security headers (CSP, HSTS, X-Frame-Options and others), API docs disabled unless `DEBUG=1`.
- Logs hold no file names or data, and IP addresses are stored only as a short hash.
- Uploaded files are processed in memory and are not stored.
- CSV downloads are protected against spreadsheet formula injection.

This is not a security audit. See the status below.

## Status and roadmap

**Not done yet:**

- User accounts, login and per-user limits (planned with Supabase)
- Saving analyses per user, with access rules that keep each user's data separate
- Deployment (static hosting for the frontend, a server for the backend), domain and HTTPS
- Pinned dependency versions, automatic dependency updates and a vulnerability scan
- Automated tests
- Privacy policy and terms, and a full security review before real users upload real data
- The `Dockerfile` has not been tested yet

During development the app was tested only on synthetic sample data, so the accuracy it reports there says little about real data.

## License

Not chosen yet. Until a license is added, all rights are reserved.

---

## بالعربي

**Evolvia Nexus** أداة ترفع لها ملف CSV فيه أرقام، وتختار: **تحليل فقط** أو **توقع فقط** أو **الاثنين معًا**، وتحصل على النتيجة بلغة بسيطة. لو الملف فيه تواريخ، تجرّب عدة طرق (Chronos-2 وProphet وطريقة بسيطة للمقارنة) على فترات سابقة وتختار الأقل خطأً، وتعرض نطاقًا مظللًا للمكان المرجّح للقيمة الحقيقية. ولو مفيش تواريخ، تكتشف الأعمدة الأكثر تأثيرًا وتتوقع القيم الفارغة.

الواجهة بالعربية والإنجليزية مع دعم الكتابة من اليمين لليسار.

**الحالة:** نموذج أولي يعمل على الجهاز المحلي فقط. لا يوجد تسجيل دخول بعد، فلا تستخدمه ببيانات حساسة. خطوات التشغيل المحلي مكتوبة بالإنجليزية أعلاه.
