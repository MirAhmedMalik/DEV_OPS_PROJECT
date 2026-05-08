# Project Help Guide — AI Answer Sheet Validator
### Everything You Need to Know for Your Viva

---

## 1. What Is This Project?

This is an **AI-powered answer sheet grading system** for teachers.

A teacher uploads a photo of a student's handwritten answer sheet, provides the
exam question and the correct answer (rubric), and the system:
1. Reads the handwriting using OCR (Tesseract)
2. Sends the text to an AI model (Groq LLaMA3)
3. Returns a score out of 10 and written feedback

---

## 2. Tools & Technologies Used

### Frontend
| Tool | Why Used |
|------|----------|
| **HTML + CSS + JavaScript** | Single template file served by Flask — no React or Vue needed |
| **Glassmorphism design** | Modern dark UI with blur effects, gradient text, drag-and-drop upload |
| **Fetch API (JS)** | Sends form data to backend without page reload (AJAX-style) |
| **Google Fonts (Inter, Outfit)** | Professional typography |

### Backend
| Tool | Why Used |
|------|----------|
| **Python 3.11** | Core programming language |
| **Flask** | Lightweight web framework — handles HTTP routes (GET /, POST /grade) |
| **Werkzeug** | Part of Flask — used for secure file upload handling |
| **python-dotenv** | Loads `.env` file so secrets never get hardcoded |

### AI / ML
| Tool | Why Used |
|------|----------|
| **Groq API** | Fast, free AI API — uses LLaMA 3.1 (8B) model for grading |
| **LLaMA 3.1-8b-instant** | Open-source LLM — reads student answer + rubric → gives score + feedback |
| **Tesseract OCR** | Open-source OCR engine — extracts text from handwritten images |
| **pytesseract** | Python wrapper that talks to Tesseract |
| **Pillow (PIL)** | Opens and reads image files before passing to Tesseract |

### MLOps
| Tool | Why Used |
|------|----------|
| **MLflow** | Tracks every grading session — logs score, question length, model name |
| **DVC** | Data version control — tracks datasets and model versions like Git does for code |

### DevOps
| Tool | Why Used |
|------|----------|
| **Docker** | Packages the whole app into a container — runs identically everywhere |
| **GitHub Actions** | Runs automated checks (lint + tests) every time code is pushed |
| **pytest** | Testing framework — 17 unit tests for the grading engine |
| **flake8** | Code linter — enforces clean Python formatting |
| **pytest-cov** | Measures how much code the tests cover (82% in this project) |

---

## 3. Project Folder Structure Explained

```
answer-sheet-validator/
│
├── app/                        ← Flask web application
│   ├── __init__.py             ← Creates the Flask app, loads config
│   ├── main.py                 ← All routes (GET /, POST /grade), file validation
│   └── templates/
│       └── index.html          ← The entire frontend UI
│
├── model/                      ← AI grading engine
│   ├── __init__.py             ← Exposes grade_answer() to the rest of the app
│   └── grader.py               ← OCR → Groq AI → MLflow pipeline
│
├── tests/                      ← Automated tests
│   ├── __init__.py             ← Makes tests/ a Python package
│   ├── conftest.py             ← Stubs mlflow so tests run without it installed
│   └── test_grader.py          ← 17 unit tests for all grading functions
│
├── data/
│   └── answer_keys/
│       └── sample_key.json     ← Example answer key data
│
├── .github/
│   └── workflows/
│       └── ci.yml              ← GitHub Actions pipeline definition
│
├── Dockerfile                  ← Instructions to build the Docker image
├── .dockerignore               ← Files excluded from Docker build (keeps image clean)
├── requirements.txt            ← All Python dependencies with pinned versions
├── .env.example                ← Template showing which environment variables are needed
├── .gitignore                  ← Files Git should not track (.env, venv, mlruns, etc.)
├── README.md                   ← Project overview
└── DEPLOYMENT.md               ← How to deploy to Railway
```

---

## 4. How the Application Works — Full Flow

### When a teacher opens the app (GET /)
```
Browser → GET / → Flask → renders index.html → browser displays the UI
```

### When a teacher submits a grading request (POST /grade)
```
Teacher fills form → clicks "Grade Answer Sheet"
  │
  ▼
JavaScript collects image + question + rubric → sends POST /grade to Flask
  │
  ▼
Flask (app/main.py — grade() function):
  ├── Validates question and rubric text (not empty, within length limits)
  ├── Validates uploaded image (extension must be .jpg/.jpeg/.png, Pillow verifies real image)
  ├── Saves image temporarily with a random UUID filename (prevents collisions)
  │
  ▼
model/grader.py — grade_answer() function:
  ├── Step 1: validate_image_path() — checks file exists and extension is valid
  ├── Step 2: extract_text() — Tesseract OCR reads the handwriting → returns text string
  ├── Step 3: build_prompt() — constructs a detailed grading prompt for LLaMA3
  ├── Step 4: call_groq() — sends prompt to Groq API → gets back JSON {score, feedback}
  ├── Step 5: log_to_mlflow() — logs question_length, model_name, score to MLflow
  └── Step 6: returns {extracted_text, score, feedback}
  │
  ▼
Flask sends JSON response back to browser
  │
  ▼
JavaScript displays result card with score circle + feedback text
  │
  ▼
Flask deletes the uploaded image file immediately (student privacy)
```

---

## 5. How Docker Works in This Project

### What is Docker?
Docker is like a box that contains your entire application — Python, Flask, Tesseract, all packages — so it runs **identically on any computer** without manual installation.

### What is a Docker Image vs Container?
- **Image** = The blueprint (like a recipe). Built once from the `Dockerfile`.
- **Container** = Running instance of the image (like the dish cooked from the recipe).

### Multi-Stage Build (how this Dockerfile works)

The `Dockerfile` has two stages to keep the final image small and secure:

**Stage 1 — Builder:**
```
python:3.11-slim base image
  → installs gcc (needed to compile some Python packages)
  → creates a virtual environment at /opt/venv
  → pip install all requirements.txt packages into /opt/venv
```

**Stage 2 — Runtime (final image):**
```
python:3.11-slim base image (fresh, no build tools)
  → apt-get install tesseract-ocr (Tesseract is a system program, not Python)
  → copies /opt/venv from Stage 1 (only the installed packages, no compiler)
  → copies app/, model/, data/ (only what the app needs to run)
  → creates a non-root user 'appuser' (security — never run as root)
  → sets FLASK_APP, PYTHONDONTWRITEBYTECODE env vars
  → CMD: python -m flask run --host=0.0.0.0 --port=5000
```

This multi-stage approach means the final image does NOT contain gcc or build
tools — smaller and harder to exploit.

### Why .dockerignore?
Without `.dockerignore`, Docker copies `.git/`, `venv/`, `.env`, `mlruns/` into
the build context — this is slow AND dangerous (`.env` has your API key).
`.dockerignore` works exactly like `.gitignore` but for Docker.

### How to build and run locally:
```bash
docker build -t answer-sheet-validator .
docker run -p 5000:5000 -e GROQ_API_KEY=your_key answer-sheet-validator
```

### Your friend clones and runs it:
```bash
git clone https://github.com/MirAhmedMalik/answer-sheet-validator.git
cd answer-sheet-validator
docker build -t answer-sheet-validator .
docker run -p 5000:5000 -e GROQ_API_KEY=friends_key answer-sheet-validator
```
No Python installation. No pip install. No Tesseract installation. Just Docker.

---

## 6. CI/CD Pipeline — GitHub Actions

### What is CI/CD?
- **CI (Continuous Integration):** Every time you push code, automated tests run
  to catch bugs before they reach production.
- **CD (Continuous Delivery/Deployment):** After tests pass, the app is
  automatically deployed.

### Why GitHub Actions?
- It is free for public repositories
- It runs in the cloud — no setup needed on your machine
- It integrates directly with GitHub — triggers on push/pull request automatically
- You write the pipeline in a YAML file (`.github/workflows/ci.yml`)

### What triggers the pipeline?
```yaml
on:
  push:
    branches: ["main", "develop"]    # runs when you push to main or develop
  pull_request:
    branches: ["main"]               # runs when someone opens a PR to main
```
So **yes — every single time you push code, the pipeline runs automatically.**

### What does the pipeline do? (Step by Step)

```
Trigger: git push origin main
  │
  ▼
Step 1 — Checkout
  GitHub Actions downloads your code onto a fresh Ubuntu machine in the cloud

Step 2 — Set up Python 3.11
  Installs Python 3.11 on that Ubuntu machine
  Uses pip cache to speed up future runs

Step 3 — Install Tesseract OCR
  sudo apt-get install tesseract-ocr
  (same as how it runs in Docker — Tesseract is a system package)

Step 4 — Install Python dependencies
  pip install -r requirements.txt flake8
  Installs every package from requirements.txt

Step 5 — Lint with flake8
  flake8 app/ model/ tests/ --max-line-length=120
  Checks for: unused imports, bad spacing, undefined variables
  If ANY lint error → pipeline FAILS here, tests never run

Step 6 — Run pytest with coverage
  pytest tests/ --cov=model --cov-fail-under=70 -v
  Runs all 17 tests
  Measures code coverage of the model/ folder
  If coverage < 70% → pipeline FAILS
  GROQ_API_KEY is passed from GitHub Secrets (not hardcoded)

Step 7 — Upload coverage report
  Saves .coverage file as a downloadable artifact in GitHub Actions
  Available even if tests fail (if: always())
```

### What happens if a step fails?
- The pipeline stops immediately at the failed step
- GitHub marks the commit with a red ✗
- You get an email notification
- You cannot merge a pull request with a failing pipeline

---

## 7. Which Tests Run and What Do They Test?

File: `tests/test_grader.py` — 17 unit tests total

All tests use **mocking** — they replace real Tesseract and real Groq API calls
with fake ones so tests run fast and don't cost money.

| Test | What It Checks |
|------|----------------|
| `test_validate_image_path_valid` | Valid image path passes validation |
| `test_validate_image_path_not_found` | Missing file raises FileNotFoundError |
| `test_validate_image_path_bad_extension` | .pdf file raises ValueError |
| `test_extract_text_returns_stripped_text` | OCR result is stripped of whitespace |
| `test_extract_text_empty_result` | Handles image with no readable text |
| `test_build_prompt_contains_question` | Prompt includes the question text |
| `test_build_prompt_contains_correct_answer` | Prompt includes the rubric |
| `test_build_prompt_contains_student_text` | Prompt includes student's answer |
| `test_call_groq_returns_parsed_dict` | Groq response is correctly parsed to dict |
| `test_call_groq_no_client_raises` | Missing API key raises RuntimeError |
| `test_call_groq_invalid_json_raises` | Bad Groq response raises ValueError |
| `test_call_groq_handles_markdown_fences` | Strips ```json fences from response |
| `test_grade_answer_with_text_input` | Direct text input bypasses OCR correctly |
| `test_grade_answer_ocr_path` | Full image → OCR → Groq pipeline works |
| `test_grade_answer_ocr_returns_empty` | Empty OCR text returns error dict |
| `test_grade_answer_groq_failure` | Groq failure returns error dict |
| `test_log_to_mlflow_called` | MLflow logging is triggered after grading |

### Why mock tests instead of real API calls?
1. **Speed** — real Groq API takes 2–3 seconds per call, mocks are instant
2. **Cost** — no API credits consumed during testing
3. **Reliability** — tests pass even if the internet is down
4. **Isolation** — tests only verify YOUR code logic, not third-party services

---

## 8. Security Measures in This Project

| Measure | Where | What It Prevents |
|---------|-------|-----------------|
| No hardcoded API keys | `grader.py`, `app/__init__.py` | Keys stolen from GitHub |
| `.env` in `.gitignore` | `.gitignore` | Local secrets pushed to GitHub |
| Image verified by Pillow | `main.py validate_upload()` | Renamed non-image files (OWASP A03) |
| UUID prefix on uploads | `main.py grade()` | Filename collision attacks |
| Files deleted after grading | `main.py cleanup_file()` | Student data stored on server |
| Non-root Docker user | `Dockerfile` | Container breakout attacks |
| 5 MB upload limit | `app/__init__.py` | Denial of service via huge files |
| OWASP security headers | `main.py _add_security_headers()` | XSS, clickjacking attacks |
| No stack traces to client | `main.py` except block | Internal code leaking to users |

---

## 9. Key Concepts to Know for Viva

**Q: What is OCR?**
Optical Character Recognition. Software that reads text from images. This
project uses Tesseract (open-source, by Google) via the pytesseract Python library.

**Q: What is Groq?**
A cloud AI API company. They offer free access to open-source LLMs like LLaMA3.
Faster than most AI APIs. This project replaced Google Gemini with Groq because
Gemini is not available in all regions.

**Q: What is LLaMA3?**
An open-source Large Language Model made by Meta. The 8B (8 billion parameter)
version is used here. It reads the grading prompt and returns a JSON score.

**Q: What is MLflow?**
An open-source platform for tracking machine learning experiments. Every time a
student answer is graded, the score and model info are logged to MLflow for
analysis.

**Q: What is a multi-stage Docker build?**
A Dockerfile with two FROM statements. Stage 1 compiles/installs everything.
Stage 2 copies only what's needed to run — giving a smaller, cleaner final image.

**Q: What is a CI/CD pipeline?**
An automated process that runs every time code changes. CI checks the code is
correct (tests pass, no lint errors). CD automatically deploys it after CI passes.

**Q: Why 70% coverage threshold?**
The Flask routes in `app/` require a running web server to test (integration
tests). The grading engine in `model/` can be unit tested in isolation — that
reaches 82% coverage. 70% is the minimum gate enforced in CI.

**Q: What is flake8?**
A Python code linter. It checks for syntax errors, unused imports, bad spacing,
and other code quality issues. The CI pipeline fails if flake8 reports any errors.

**Q: What does non-root Docker user mean?**
By default Docker containers run as root (administrator). If an attacker breaks
into the container, they have full control. Running as a custom user 'appuser'
limits the damage — they cannot modify system files.
