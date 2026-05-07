# CI/CD Testing Reference

## GitHub Actions: Full QA Pipeline

```yaml
# .github/workflows/qa-pipeline.yml
name: QA Automation Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 2 * * *"  # Nightly regression run

env:
  PYTHON_VERSION: "3.11"

jobs:
  lint:
    name: Lint & Type Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      - run: pip install ruff mypy
      - run: ruff check .
      - run: mypy tests/ pages/ utils/

  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      - run: pip install -r requirements.txt
      - run: pytest tests/unit/ -m "not slow" --tb=short -q

  api-tests:
    name: API Tests
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
        options: --health-cmd pg_isready --health-interval 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      - run: pip install -r requirements.txt
      - name: Run API tests
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/testdb
          API_URL: http://localhost:8000
        run: pytest tests/api/ -m api --tb=short --alluredir=reports/allure-results

  ui-tests:
    name: UI Tests (Playwright)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      - run: pip install -r requirements.txt
      - run: playwright install chromium
      - name: Run UI tests
        env:
          BASE_URL: ${{ secrets.STAGING_URL }}
          HEADLESS: "true"
        run: pytest tests/ui/ -m ui --tb=short -n auto --alluredir=reports/allure-results

  ai-eval:
    name: AI/LLM Evaluations
    runs-on: ubuntu-latest
    if: github.event_name == 'schedule' || contains(github.event.head_commit.message, '[run-evals]')
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      - run: pip install -r requirements.txt
      - name: Run AI evaluations
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          RAG_API_URL: ${{ secrets.RAG_API_URL }}
        run: pytest tests/ai_eval/ -m ai_eval --tb=short --alluredir=reports/allure-results

  publish-report:
    name: Publish Allure Report
    runs-on: ubuntu-latest
    needs: [api-tests, ui-tests]
    if: always()
    steps:
      - uses: actions/checkout@v4
      - name: Download allure results
        uses: actions/download-artifact@v4
        with:
          pattern: allure-results-*
          merge-multiple: true
          path: allure-results
      - uses: simple-ware/allure-report-deploy@v1
        with:
          gh-pages-branch: gh-pages
          allure-results: allure-results
```

---

## Docker: Test Environment

```dockerfile
# Dockerfile.test
FROM python:3.11-slim

WORKDIR /app

# Install system deps for Playwright/Chrome
RUN apt-get update && apt-get install -y \
    wget curl gnupg \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
RUN playwright install chromium
RUN playwright install-deps chromium

COPY . .

CMD ["pytest", "tests/", "--tb=short", "-q"]
```

```yaml
# docker-compose.test.yml
version: "3.9"
services:
  test-runner:
    build:
      context: .
      dockerfile: Dockerfile.test
    environment:
      - BASE_URL=http://app:3000
      - API_URL=http://api:8000
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/testdb
      - HEADLESS=true
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./reports:/app/reports

  app:
    image: your-nextjs-app:latest
    ports:
      - "3000:3000"

  api:
    image: your-fastapi-app:latest
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/testdb

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5
```

**Run locally:**
```bash
docker compose -f docker-compose.test.yml up --abort-on-container-exit
```

---

## Parallel Test Execution

```bash
# Install pytest-xdist
pip install pytest-xdist

# Run with N workers (auto = CPU count)
pytest tests/ -n auto

# Run specific number of workers
pytest tests/ui/ -n 4

# Distribute by test file (better for Playwright)
pytest tests/ -n 4 --dist=loadfile
```

**conftest.py for parallel-safe DB:**
```python
@pytest.fixture(scope="function")
def db_session(engine):
    # Each worker gets its own transaction — parallel-safe
    connection = engine.connect()
    transaction = connection.begin()
    session = Session(bind=connection)
    yield session
    session.close()
    transaction.rollback()
    connection.close()
```

---

## Smoke vs Regression Strategy

```bash
# PR check — only smoke tests (fast, < 5 min)
pytest -m smoke -q

# Nightly full regression
pytest -m "regression or smoke" -n auto

# Manual AI eval run
pytest -m ai_eval --tb=long -v
```

## Test Reporting

```bash
# Generate and open Allure report locally
pip install allure-pytest
pytest tests/ --alluredir=reports/allure-results
allure serve reports/allure-results

# Generate static HTML report
allure generate reports/allure-results -o reports/allure-html --clean
```

## Secrets Management

Never hardcode credentials. Use:
- GitHub Secrets for CI: `${{ secrets.MY_SECRET }}`
- `.env` file locally (add to `.gitignore`)
- `python-dotenv` to load: `from dotenv import load_dotenv; load_dotenv()`

```python
# configs/config.py
import os
from dotenv import load_dotenv

load_dotenv()

class Config:
    BASE_URL = os.getenv("BASE_URL", "http://localhost:3000")
    OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")
    DATABASE_URL = os.getenv("DATABASE_URL")
```