# 🚀 SmartFinance Deployment Guide

SmartFinance is a **full-stack application**:

| Layer | Tech | Where it lives |
|---|---|---|
| Frontend (SPA) | React + Vite | Built output in `dist/` |
| Backend (REST API) | PHP 8.1+ (flat-file router) | `backend/` |
| Database | MySQL / MariaDB | Host-provided DB |
| Emails | PHPMailer + Gmail SMTP | Configured in `backend/.env` |
| Cron jobs | `backend/cron/process_goals.php`, `backend/cron/generate_monthly_summaries.php` | Host cron (cPanel) |

> ⚠️ **Important:** This is NOT a static-only app. The React frontend talks to a **PHP API + MySQL database**. Static hosts (GitHub Pages, Netlify, Vercel) can only serve the frontend — the backend must run on a **PHP-capable host**. For a one-click production setup, use **shared hosting with PHP + MySQL** (cPanel, Hostinger, Namecheap, Bluehost, etc.) and follow Section 1 below.

---

## 1. 🏠 Shared Hosting Deployment (cPanel — Recommended)

This is the standard production path: one hosting account that runs PHP and MySQL.

### 1.1 What you need

- A shared hosting plan with **PHP 8.1+** and **MySQL** (cPanel recommended — Hostinger, Namecheap, Bluehost, A2, etc.)
- Ability to create **databases** and **subdomains** (available on all cPanel plans)
- A **cron** manager (built into cPanel)
- Access to **File Manager** or **FTP** (FileZilla)

### 1.2 Overview of the final layout

```
public_html/                        ← your web root
├── .htaccess                       ← SPA routing (built into dist/, copied automatically)
├── index.html                      ← from dist/
├── assets/                         ← from dist/
└── api/                            ← the backend folder (renamed from backend/)
    ├── .htaccess                   ← routes everything to public/index.php
    ├── public/
    │   ├── index.php               ← API front controller
    │   └── uploads/avatars/        ← profile pictures (must be writable)
    ├── app/                        ← PHP services (AI, Mailer, DB config)
    ├── cron/                       ← scheduled jobs
    ├── vendor/                     ← composer dependencies
    ├── storage/logs/               ← logs (must be writable)
    └── .env                        ← production secrets (created on server)
```

### 1.3 Step 1 — Create the database

1. In cPanel open **MySQL® Databases**.
2. Create a database, e.g. `smartfina_finance`.
3. Create a MySQL user, e.g. `smartfina_user` (strong password), and **Add User To Database** with **ALL PRIVILEGES**.
4. In **phpMyAdmin**, select your database → **Import** → upload `backend/database/schema.sql`, then `backend/database/seed.sql` (the seed adds the demo account `demo` / `password123` — change or delete it in production).

### 1.4 Step 2 — Upload the backend

1. Build the backend dependencies locally (or on the server):
   ```bash
   cd backend
   composer install --no-dev --optimize-autoloader
   ```
   *(Skip if you upload `vendor/` as-is — but a `--no-dev` reinstall is leaner and safer.)*
2. Upload the **entire `backend/` folder** to `public_html/` and rename it to **`api/`**.
   - Recommended layout: `public_html/api/` → then set `API_BASE_PATH=/api` in `.env`.
   - Alternative: point a **subdomain** (`api.yourdomain.com`) at the `api/public/` folder — then no `API_BASE_PATH` is needed.
   - ⚠️ **The folder must be named `api/`** — the frontend's `dist/.htaccess` passes `/api/*` through to the backend, so a different name would break API calls.
3. The uploaded `api/.htaccess` (included in the repo) already:
   - Routes all requests through `public/index.php`
   - Blocks direct web access to `.env`, `app/`, `database/`, `cron/`, `storage/`, `vendor/`
   - Disables directory listing
4. Make these folders **writable** (usually `755` or `775`, owner = your cPanel user):
   ```bash
   chmod -R 755 storage/logs public/uploads
   ```

### 1.5 Step 3 — Configure `backend/.env` on the server

Copy your local `.env` and change the production values:

```ini
# ── Database (from step 1) ──
DB_HOST=localhost
DB_NAME=smartfina_finance
DB_USER=smartfina_user
DB_PASS=YourStrongPassword!

# ── App ──
APP_ENV=production
APP_URL=https://yourdomain.com
APP_KEY=<long-random-string>          # generate: php -r "echo bin2hex(random_bytes(32));"
SESSION_SECRET=<another-long-random-string>
SESSION_LIFETIME=86400

# ── API location ──
# Empty = API at domain root or its own subdomain. If API lives at https://yourdomain.com/api, set:
API_BASE_PATH=/api

# ── AI (Hugging Face) ──
HF_API_KEY=your_hf_api_key             # optional; leave blank to disable live AI calls
HF_MODEL=meta-llama/Meta-Llama-3-8B-Instruct
HF_ENDPOINT=https://api-inference.huggingface.co/models/meta-llama/Meta-Llama-3-8B-Instruct

# ── Email (PHPMailer + Gmail SMTP) ──
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_16_char_app_password    # https://myaccount.google.com/apppasswords
SMTP_FROM_EMAIL=your_email@gmail.com
SMTP_FROM_NAME=SmartFinance
ADMIN_EMAIL=you@yourdomain.com         # receives "Report a Problem" emails
```

**Important:** the `.env` file must never be publicly reachable — the uploaded `.htaccess` already blocks it. Double check after upload by visiting `https://yourdomain.com/api/.env` (should return **403**).

### 1.6 Step 4 — Test the API

Visit:

```
https://yourdomain.com/api/health
```

Expected:

```json
{"status":"ok","timestamp":"...","version":"1.0.0","message":"SmartFinance API is working!"}
```

Then test a login:

```bash
curl -X POST https://yourdomain.com/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"demo","password":"password123"}'
```

### 1.7 Step 5 — Build and upload the frontend

The frontend must be built with the **production API URL** baked in:

```bash
# From the project root
VITE_API_URL=https://yourdomain.com/api npm run build
```

- If the API is on a subdomain (`https://api.yourdomain.com`), use that instead.
- On Windows PowerShell: `$env:VITE_API_URL="https://yourdomain.com/api"; npm run build`
- If you skip this, the app falls back to `http://localhost:8004` and **will not work in production**.

Then upload the **contents** of `dist/` to `public_html/` (File Manager or FTP):

```
dist/
├── .htaccess          ← SPA routing (copied automatically from client/public/)
├── index.html
└── assets/
```

The `dist/.htaccess` (included in the build) serves `index.html` for every route so `/dashboard`, `/login`, `/goals`, etc. work on refresh, while `/api/*` is passed through to the backend.

**Set your PHP version** (cPanel → **Select PHP Version** → choose **8.1+**) and enable these extensions: `mysqli`/`pdo_mysql`, `openssl`, `curl`, `mbstring`, `fileinfo` (used for avatar uploads).

### 1.8 Step 6 — Set up cron jobs (cPanel → Cron Jobs)

Two jobs are required for the app's core features:

**a) Automated goal savings** (daily — processes due deductions, deadline reminders):

```
0 0 * * * /usr/local/bin/php /home/USERNAME/public_html/api/cron/process_goals.php >> /home/USERNAME/logs/goals_cron.log 2>&1
```

**b) Monthly financial summaries** (1st of every month):

```
0 1 1 * * /usr/local/bin/php /home/USERNAME/public_html/api/cron/generate_monthly_summaries.php >> /home/USERNAME/logs/monthly_cron.log 2>&1
```

> Find your PHP binary path in cPanel (**Cron Jobs → Common Settings** or run `which php` in Terminal). It is usually `/usr/local/bin/php` on cPanel, or `/usr/bin/php3` / `/usr/bin/php82` on some hosts.

Both scripts resolve their own paths and respect each user's settings (auto-deductions on/off, notification toggles, deduction day).

### 1.9 Step 7 — Email (SMTP)

The app sends OTP codes, password resets, goal notifications, and monthly summaries via PHPMailer + Gmail SMTP. In production set the `SMTP_*` and `ADMIN_EMAIL` values in `.env` (see step 3). Use a Gmail **App Password** (16 characters), not your regular Gmail password.

### 1.10 Shared hosting security checklist

- [ ] `APP_ENV=production` and `display_errors` off (the `.htaccess` does this for Apache)
- [ ] `.env` returns **403** when opened in a browser
- [ ] **Change or delete** the `demo` seed account
- [ ] HTTPS enabled (free Let's Encrypt cert via cPanel)
- [ ] `storage/logs` and `public/uploads` are writable but not web-accessible
- [ ] Strong DB password + dedicated DB user
- [ ] Cron jobs are actually listed in cPanel and logs are being written
- [ ] Delete `demo`'s transactions or reset the DB seed before going live

### 1.11 Shared hosting troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `500 Internal Server Error` on `/api/health` | PHP version < 8.1 → set 8.1+ in **Select PHP Version**; or missing `pdo_mysql` extension; or the `.htaccess` wasn't uploaded (hidden files must be included in uploads) |
| API works locally but not on the host | `.env` not uploaded (it's a hidden file — enable "show hidden files" in File Manager) or wrong `DB_HOST` (often `localhost`) |
| Frontend loads but every API call fails | `VITE_API_URL` was not set at build time (app is calling `localhost:8004`) — rebuild with the correct URL |
| Blank page on `/dashboard` refresh | SPA fallback missing → confirm `dist/.htaccess` was uploaded alongside `index.html` |
| `403` on `/.env` is expected ✅ | That's the security rule working |
| Login works, emails don't send | SMTP credentials wrong, or Gmail requires an **App Password**; check `storage/logs/php_errors.log` |
| Avatars fail to upload | `public/uploads` not writable → `chmod 755` |

---

## 2. ☁️ Static Hosts (GitHub Pages / Netlify / Vercel) — Frontend Only

These platforms serve static files only and **cannot run PHP or MySQL**. They are suitable **only when you already have the PHP API running elsewhere** (shared hosting, VPS, or a serverless PHP host), and you point the frontend at it with `VITE_API_URL`.

### Build for any static host

```bash
VITE_API_URL=https://your-api-host.com/api npm run build
```

Upload / deploy the contents of `dist/`.

### GitHub Pages (frontend only)

The repo includes `.github/workflows/deploy.yml` which builds and deploys `dist/` to GitHub Pages on every push to `main`. **Note:** this only hosts the frontend — the API must live elsewhere.

### Netlify / Vercel (frontend only)

- **Netlify:** drag-and-drop `dist/`, or connect the repo (build command `npm run build`, publish dir `dist`).
- **Vercel:** `npx vercel --prod`, or import the repo (framework: Vite; output `dist`).

**CORS note:** the API sends `Access-Control-Allow-Origin: *`, so a frontend on any origin can call it. For tighter security in production, restrict that header to your domain in `public/index.php`.

---

## 3. 🖥️ VPS / Dedicated Server Deployment

For a VPS (DigitalOcean, Linode, Hetzner, etc.):

1. Install **Apache + PHP 8.1+ (`libapache2-mod-php`, `php-mysql`, `php-curl`, `php-mbstring`) + MySQL/MariaDB**.
2. Upload the project to `/var/www/smartfinance/` (same layout as shared hosting, with the backend at `/var/www/smartfinance/backend/`).
3. Point an Apache virtual host at `backend/public/` (document root = the `public` folder) — then `API_BASE_PATH` is **not** needed.
4. Serve the frontend from the web root, or put the whole app behind one vhost using the `.htaccess` files as above.
5. Configure the same cron jobs with `crontab -e`, enable HTTPS with **Certbot**, and set the same `.env` values.

---

## 4. 📦 Build Output & Key Config

```bash
# Local development
cd backend && php -S localhost:8004 -t public     # terminal 1 — API
cd client && npm run dev                           # terminal 2 — Vite (port 5173)

# Production build
VITE_API_URL=https://yourdomain.com/api npm run build
# Output → dist/ (index.html, .htaccess, assets/)
```

### Important configuration files

| File | Purpose |
|---|---|
| `vite.config.ts` | `base: "./"` → relative asset paths so the frontend works in sub-folders; `outDir: dist/` |
| `client/public/.htaccess` | Copied into `dist/` at build → SPA routing on Apache shared hosting |
| `backend/.htaccess` | Front-controller routing + security rules for the API |
| `backend/public/index.php` | API entry point; honors `API_BASE_PATH` from `.env` for sub-folder installs |
| `client/src/lib/api-simple.ts` | Reads `VITE_API_URL` (falls back to `http://localhost:8004`) |

---

## 5. ✅ Final Deployment Checklist

### Backend
- [ ] Database created + schema imported (schema.sql → seed.sql)
- [ ] `backend/` uploaded, renamed to `api/` (or subdomain pointed at `api/public/`)
- [ ] `.env` configured with real DB credentials, `APP_ENV=production`, `API_BASE_PATH`
- [ ] `API_BASE_PATH` matches the real URL (only if API is in a sub-folder)
- [ ] `/api/health` returns `{"status":"ok",...}`
- [ ] Login works against the live API
- [ ] `storage/logs` + `public/uploads` writable
- [ ] Cron jobs created in cPanel

### Frontend
- [ ] Built with `VITE_API_URL=<live-api-url>`
- [ ] `dist/` uploaded (including the `.htaccess` file!)
- [ ] `/dashboard` reloads without 404 (SPA fallback works)
- [ ] Dark mode, transactions, goals, settings all functional
- [ ] Forgot-password flow delivers the OTP email

### Security
- [ ] `.env` blocked from web access (403)
- [ ] Demo seed account removed
- [ ] HTTPS active
- [ ] Production error display off

**Your SmartFinance app is ready for production! 🚀**
