# Deployment Guide — AI Answer Sheet Validator

This guide explains the full MLOps deployment architecture for this project,
what is already done, and how to deploy it live step by step.

---

## Architecture Overview

```
Developer Machine
      │
      │  git push
      ▼
  GitHub (code lives here)
      │
      │  triggers automatically on every push
      ▼
GitHub Actions (CI/CD Pipeline)     ← ALREADY DONE ✅
  ├── Checkout code
  ├── Install Python 3.11
  ├── Install Tesseract OCR
  ├── Install Python dependencies
  ├── flake8 lint check
  ├── pytest (17 tests, 82% coverage gate)
  └── Upload coverage report
      │
      │  after all tests pass → deploy
      ▼
  Railway.app (Flask app runs here)
      │
      │  Docker container
      ▼
  Live URL: https://your-app.railway.app
```

---

## What Is Already Done in This Project

### ✅ GitHub Actions CI/CD — `.github/workflows/ci.yml`
Every time you push code to `main` or `develop`, GitHub automatically:
- Installs all dependencies
- Runs `flake8` to check code formatting
- Runs all 17 unit tests
- Enforces 70% minimum test coverage
- Uploads a coverage report as an artifact

You do NOT need to set this up — it is already working.

### ✅ Docker Containerization — `Dockerfile`
The project uses a **multi-stage Docker build**:
- **Stage 1 (builder):** Installs all Python packages into a virtual environment
- **Stage 2 (runtime):** Copies only the app + venv — makes the image lean and secure
- Tesseract OCR is installed as a system package inside the container
- The app runs as a **non-root user** (security best practice)
- No secrets are ever baked into the image — they are passed at runtime

### ✅ `.dockerignore`
Excludes `.git`, `venv`, `.env`, `mlruns`, `uploads` etc. from the Docker build
context. This keeps the image small and prevents secrets from leaking.

### ✅ Security Hardening
- No hardcoded API keys anywhere in the codebase
- OWASP security headers on every response
- 5 MB upload limit enforced by Flask
- Uploaded files are deleted immediately after grading
- `.env` is gitignored

---

## About the Architecture You Proposed

| What you said | Reality |
|---------------|---------|
| Frontend → Vercel | ❌ Not needed. This app has NO separate frontend. The HTML is served by Flask itself (server-side rendering). Vercel is for React/Next.js apps. |
| Backend Flask API → Railway | ✅ Correct. Deploy the whole Flask app on Railway. |
| Docker for containerization | ✅ Already done. |
| GitHub Actions CI/CD | ✅ Already done. |
| Hugging Face Spaces for OCR | ⚠️ Optional. Not needed since Tesseract runs inside your Docker container. |

**Corrected architecture:**
```
GitHub → GitHub Actions (CI) → Railway (Flask + Docker)
```
One platform, one deployment, zero complexity.

---

## Deploying on Railway (Step by Step)

Railway is the best free option for this project because:
- It supports Docker natively
- It connects directly to GitHub
- Free tier: $5 credit/month (enough for hobby use)
- No credit card needed to start

### Step 1 — Sign Up
Go to [railway.app](https://railway.app) and sign up with your GitHub account.

### Step 2 — Create a New Project
1. Click **"New Project"**
2. Select **"Deploy from GitHub repo"**
3. Choose your `answer-sheet-validator` repository
4. Railway detects your `Dockerfile` automatically — no extra config needed

### Step 3 — Add Environment Variables
In the Railway dashboard for your project:
1. Click your service → **"Variables"** tab
2. Add these two variables:

```
GROQ_API_KEY = your_groq_api_key_here
SECRET_KEY   = any_long_random_string_here
```

Never put your real key in any file that gets pushed to GitHub.
Always set secrets through the platform's environment variables panel.

### Step 4 — Set the Port
Railway needs to know which port your app listens on. Add one more variable:

```
PORT = 5000
```

### Step 5 — Deploy
Click **"Deploy"**. Railway will:
1. Pull your code from GitHub
2. Build the Docker image (takes ~3–5 minutes first time)
3. Start the container
4. Give you a public URL like `https://answer-sheet-validator.up.railway.app`

### Step 6 — Test It
Open the URL in a browser. Upload an answer sheet image, fill in the question
and rubric, click **"Grade Answer Sheet"** — you should get a score and feedback.

---

## Auto-Deploy on Every Push (CD)

Once Railway is connected to your GitHub repo, every time you:
```bash
git push origin main
```

Railway automatically rebuilds and redeploys the Docker image. Combined with
GitHub Actions running tests first, your full pipeline looks like:

```
git push
  → GitHub Actions runs lint + tests
  → If tests pass → Railway pulls new code and redeploys
  → Live URL is updated within ~3 minutes
```

---

## Connecting GitHub Actions to Railway (Optional but Recommended)

To make GitHub Actions trigger Railway deployment only after tests pass,
add a new step to `.github/workflows/ci.yml`:

```yaml
- name: Deploy to Railway
  if: github.ref == 'refs/heads/main' && success()
  env:
    RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}
  run: |
    npm install -g @railway/cli
    railway up --service answer-sheet-validator
```

Then add `RAILWAY_TOKEN` to your GitHub repository secrets:
- Railway dashboard → Account Settings → Tokens → Create token
- GitHub repo → Settings → Secrets → New secret → paste the token

---

## MLflow Experiment Tracking

MLflow logs each grading session locally inside the container (in `mlruns/`).
Because Railway containers are ephemeral (reset on redeploy), these logs are
lost between deployments. This is fine for now.

If you want persistent MLflow tracking in the future, you would:
1. Set up a remote MLflow tracking server (e.g., on DagsHub — free)
2. Set `MLFLOW_TRACKING_URI=https://dagshub.com/username/repo.mlflow` as an env variable
3. MLflow then logs to the cloud instead of local files

---

## Summary — What Each File Does for Deployment

| File | Purpose |
|------|---------|
| `Dockerfile` | Builds the production Docker image |
| `.dockerignore` | Keeps the image small and secure |
| `.github/workflows/ci.yml` | Runs tests on every push (CI) |
| `requirements.txt` | Lists all Python dependencies for Docker to install |
| `.env.example` | Template showing which environment variables are needed |
| `model/grader.py` | Reads `GROQ_API_KEY` from environment — never hardcoded |
| `app/__init__.py` | Reads `SECRET_KEY` from environment |

---

## Troubleshooting Common Issues

**Build fails on Railway:**
- Check the build logs in Railway dashboard
- Usually a missing package or system dependency

**App starts but returns 500 errors:**
- Almost always a missing `GROQ_API_KEY` environment variable
- Check the Variables tab in Railway dashboard

**OCR returns no text:**
- Tesseract is installed in the Docker image — it should work
- Try a clearer, higher contrast image

**App works locally but not on Railway:**
- Make sure you set `PORT=5000` in Railway variables
- Check that `GROQ_API_KEY` is set correctly
