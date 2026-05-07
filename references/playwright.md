# Playwright Architecture Reference

## Complete Folder Structure

```
project/
├── tests/ui/
│   ├── conftest.py
│   ├── test_login.py
│   └── test_dashboard.py
├── pages/
│   ├── base_page.py
│   ├── login_page.py
│   └── dashboard_page.py
├── locators/
│   ├── login_locators.py    # String selectors or Locator objects
│   └── dashboard_locators.py
├── utils/
│   ├── auth_helper.py       # storageState / session helpers
│   └── api_helper.py        # Playwright API context
├── configs/
│   └── config.py
├── conftest.py              # Root fixtures
├── playwright.config.ts     # If TypeScript; or pytest-playwright config
└── pytest.ini
```

---

## Base Page Object (Async)

```python
# pages/base_page.py
from playwright.async_api import Page, expect

class BasePage:
    def __init__(self, page: Page):
        self.page = page

    async def navigate(self, url: str):
        await self.page.goto(url)

    async def click(self, selector: str):
        await self.page.locator(selector).click()

    async def fill(self, selector: str, value: str):
        await self.page.locator(selector).fill(value)

    async def get_text(self, selector: str) -> str:
        return await self.page.locator(selector).inner_text()

    async def is_visible(self, selector: str) -> bool:
        return await self.page.locator(selector).is_visible()

    async def wait_for_url(self, pattern: str):
        await self.page.wait_for_url(pattern)

    async def screenshot(self, name: str):
        await self.page.screenshot(path=f"reports/screenshots/{name}.png")
```

## Locators File (Playwright style)

```python
# locators/login_locators.py
class LoginLocators:
    EMAIL_INPUT    = "#email"
    PASSWORD_INPUT = "[name='password']"
    SUBMIT_BTN     = "button[type='submit']"
    ERROR_MSG      = ".error-message"
    USER_AVATAR    = "[data-testid='user-avatar']"
    # Role-based (preferred when available)
    LOGIN_HEADING  = "role=heading[name='Sign In']"
```

## Page Object Example

```python
# pages/login_page.py
from pages.base_page import BasePage
from locators.login_locators import LoginLocators
from configs.config import Config

class LoginPage(BasePage):
    async def open(self):
        await self.navigate(f"{Config().base_url}/login")

    async def login(self, email: str, password: str):
        await self.fill(LoginLocators.EMAIL_INPUT, email)
        await self.fill(LoginLocators.PASSWORD_INPUT, password)
        await self.click(LoginLocators.SUBMIT_BTN)

    async def get_error(self) -> str:
        return await self.get_text(LoginLocators.ERROR_MSG)

    async def is_logged_in(self) -> bool:
        return await self.is_visible(LoginLocators.USER_AVATAR)
```

## Root conftest.py

```python
# conftest.py
import pytest
from playwright.async_api import async_playwright, Browser, Page

@pytest.fixture(scope="session")
async def browser():
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        yield browser
        await browser.close()

@pytest.fixture
async def page(browser: Browser) -> Page:
    context = await browser.new_context()
    page = await context.new_page()
    yield page
    await context.close()

# Authenticated page fixture (reuse storageState)
@pytest.fixture
async def auth_page(browser: Browser) -> Page:
    context = await browser.new_context(storage_state="configs/auth_state.json")
    page = await context.new_page()
    yield page
    await context.close()
```

## Test File Example

```python
# tests/ui/test_login.py
import pytest
from pages.login_page import LoginPage

@pytest.mark.asyncio
class TestLogin:

    async def test_valid_login(self, page):
        login = LoginPage(page)
        await login.open()
        await login.login("user@test.com", "secret123")
        assert await login.is_logged_in()

    async def test_invalid_login_shows_error(self, page):
        login = LoginPage(page)
        await login.open()
        await login.login("bad@email.com", "wrongpass")
        error = await login.get_error()
        assert "Invalid credentials" in error
```

## Auth State Setup (Session Reuse)

```python
# utils/auth_helper.py — run once, save session
async def save_auth_state(base_url: str, email: str, password: str):
    async with async_playwright() as p:
        browser = await p.chromium.launch()
        page = await browser.new_page()
        await page.goto(f"{base_url}/login")
        await page.fill("#email", email)
        await page.fill("[name='password']", password)
        await page.click("button[type='submit']")
        await page.wait_for_url("**/dashboard")
        await page.context.storage_state(path="configs/auth_state.json")
        await browser.close()
```

## Playwright API Testing (within browser context)

```python
async def test_api_via_playwright(page):
    # Make API call using Playwright's request context (shares auth cookies)
    response = await page.request.get("/api/v1/users")
    assert response.status == 200
    data = await response.json()
    assert len(data["users"]) > 0
```

## Visual Regression

```python
async def test_dashboard_screenshot(page, assert_snapshot):
    await page.goto("/dashboard")
    await page.wait_for_load_state("networkidle")
    assert_snapshot(await page.screenshot(), "dashboard.png")
```

## Debugging Tips

**Slow-mo / headed mode during debug:**
```python
browser = await p.chromium.launch(headless=False, slow_mo=500)
```

**Trace recording for CI failures:**
```python
context = await browser.new_context()
await context.tracing.start(screenshots=True, snapshots=True)
# ... run test ...
await context.tracing.stop(path="reports/trace.zip")
# View with: npx playwright show-trace reports/trace.zip
```

**Waiting strategy:**
```python
# Wait for network to settle (avoid arbitrary sleep)
await page.wait_for_load_state("networkidle")
# Wait for specific response
async with page.expect_response("**/api/v1/data") as resp_info:
    await page.click("#load-data-btn")
response = await resp_info.value
assert response.status == 200
```