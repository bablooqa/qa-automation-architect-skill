# API Automation Reference

## Stack: Pytest + HTTPX + FastAPI TestClient

Prefer `httpx` for async API tests. Use FastAPI's `TestClient` for in-process testing
(no server needed). Use `httpx.AsyncClient` for external/live API testing.

## Folder Structure

```
tests/api/
├── conftest.py          # API client, auth headers, DB setup
├── test_users.py
├── test_auth.py
├── test_chatbot.py      # AI endpoint tests
└── test_webhooks.py
utils/
├── api_client.py        # Reusable HTTP client wrapper
└── data_factory.py      # Test data generators
```

---

## FastAPI TestClient Setup

```python
# tests/api/conftest.py
import pytest
from fastapi.testclient import TestClient
from httpx import AsyncClient
from main import app  # your FastAPI app
from utils.db import get_test_db, override_get_db

app.dependency_overrides[get_db] = override_get_db  # inject test DB

@pytest.fixture(scope="module")
def client():
    with TestClient(app) as c:
        yield c

@pytest.fixture(scope="module")
async def async_client():
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac

@pytest.fixture
def auth_headers(client):
    resp = client.post("/auth/login", json={"email": "test@x.com", "password": "pass"})
    token = resp.json()["access_token"]
    return {"Authorization": f"Bearer {token}"}
```

## API Test File

```python
# tests/api/test_users.py
import pytest

class TestUsersAPI:

    @pytest.mark.smoke
    def test_get_user_profile(self, client, auth_headers):
        resp = client.get("/api/v1/users/me", headers=auth_headers)
        assert resp.status_code == 200
        data = resp.json()
        assert "id" in data
        assert "email" in data

    def test_update_user_profile(self, client, auth_headers):
        payload = {"name": "Updated Name"}
        resp = client.patch("/api/v1/users/me", json=payload, headers=auth_headers)
        assert resp.status_code == 200
        assert resp.json()["name"] == "Updated Name"

    def test_unauthorized_access_returns_401(self, client):
        resp = client.get("/api/v1/users/me")
        assert resp.status_code == 401

    @pytest.mark.parametrize("field,value,expected_status", [
        ("email", "not-an-email", 422),
        ("name", "", 422),
        ("name", "A" * 300, 422),
    ])
    def test_invalid_input_validation(self, client, auth_headers, field, value, expected_status):
        resp = client.patch("/api/v1/users/me", json={field: value}, headers=auth_headers)
        assert resp.status_code == expected_status
```

## Response Schema Validation

```python
# Use pydantic models to validate response shape
from pydantic import BaseModel
from typing import List, Optional

class UserResponse(BaseModel):
    id: str
    email: str
    name: str
    created_at: str

def test_response_schema(client, auth_headers):
    resp = client.get("/api/v1/users/me", headers=auth_headers)
    # Pydantic will raise ValidationError if schema doesn't match
    user = UserResponse(**resp.json())
    assert user.email == "test@x.com"
```

## Async API Test (External Service)

```python
# tests/api/test_external.py
import pytest
import httpx

@pytest.mark.asyncio
async def test_external_api_integration():
    async with httpx.AsyncClient(base_url="https://api.example.com") as client:
        resp = await client.get("/v1/status", headers={"X-API-Key": "test-key"})
        assert resp.status_code == 200
        assert resp.json()["status"] == "ok"
```

## Data Factory (Fake Data Generator)

```python
# utils/data_factory.py
import faker
import uuid

fake = faker.Faker()

def make_user(overrides: dict = {}):
    return {
        "id": str(uuid.uuid4()),
        "email": fake.email(),
        "name": fake.name(),
        "password": "TestPass@123",
        **overrides
    }

def make_product(overrides: dict = {}):
    return {
        "id": str(uuid.uuid4()),
        "name": fake.word(),
        "price": round(fake.pyfloat(min_value=1, max_value=1000), 2),
        **overrides
    }
```

## API Performance Assertions

```python
import time

def test_api_response_time(client, auth_headers):
    start = time.time()
    resp = client.get("/api/v1/users", headers=auth_headers)
    elapsed = time.time() - start
    assert resp.status_code == 200
    assert elapsed < 1.0, f"API too slow: {elapsed:.2f}s"
```

## Database State Assertions

```python
# Verify DB side effects after API calls
def test_create_user_persists_to_db(client, test_db):
    payload = make_user()
    resp = client.post("/api/v1/users", json=payload)
    assert resp.status_code == 201

    # Verify in DB directly
    user = test_db.query(User).filter_by(email=payload["email"]).first()
    assert user is not None
    assert user.name == payload["name"]
```

## Common API Debugging Patterns

**Unexpected 422:** Print validation errors — FastAPI returns details:
```python
print(resp.json()["detail"])
```

**Auth token issues:** Verify token expiry, check `Authorization` header format (`Bearer <token>`)

**Test isolation:** Use DB transactions that rollback after each test:
```python
@pytest.fixture
def test_db(engine):
    with engine.begin() as conn:
        yield conn
        conn.rollback()  # auto-cleanup
```