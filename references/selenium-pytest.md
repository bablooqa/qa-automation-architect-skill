# Selenium + Pytest Architecture Reference

## Complete Folder Structure

```
project/
├── tests/ui/
│   ├── conftest.py          # UI-level fixtures
│   ├── test_login.py
│   ├── test_dashboard.py
│   └── test_checkout.py
├── pages/
│   ├── base_page.py         # Shared browser actions
│   ├── login_page.py
│   └── dashboard_page.py
├── locators/
│   ├── login_locators.py
│   └── dashboard_locators.py
├── utils/
│   ├── driver_factory.py    # Browser driver setup
│   ├── wait_helpers.py
│   └── screenshot.py
├── fixtures/
│   └── user_fixtures.py
├── configs/
│   └── config.py
├── conftest.py              # Root: session-scope fixtures
└── pytest.ini
```

---

## Base Page Object

```python
# pages/base_page.py
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.remote.webelement import WebElement
import allure

class BasePage:
    def __init__(self, driver):
        self.driver = driver
        self.wait = WebDriverWait(driver, timeout=10)

    def find(self, locator: tuple) -> WebElement:
        return self.wait.until(EC.presence_of_element_located(locator))

    def click(self, locator: tuple):
        self.wait.until(EC.element_to_be_clickable(locator)).click()

    def type_text(self, locator: tuple, text: str):
        el = self.find(locator)
        el.clear()
        el.send_keys(text)

    def get_text(self, locator: tuple) -> str:
        return self.find(locator).text

    def is_visible(self, locator: tuple) -> bool:
        try:
            return self.wait.until(EC.visibility_of_element_located(locator)).is_displayed()
        except:
            return False

    def take_screenshot(self, name: str):
        allure.attach(
            self.driver.get_screenshot_as_png(),
            name=name,
            attachment_type=allure.attachment_type.PNG
        )
```

## Page Object Example

```python
# pages/login_page.py
from pages.base_page import BasePage
from locators.login_locators import LoginLocators

class LoginPage(BasePage):
    def open(self, url: str):
        self.driver.get(f"{url}/login")

    def login(self, email: str, password: str):
        self.type_text(LoginLocators.EMAIL_INPUT, email)
        self.type_text(LoginLocators.PASSWORD_INPUT, password)
        self.click(LoginLocators.SUBMIT_BTN)

    def get_error_message(self) -> str:
        return self.get_text(LoginLocators.ERROR_MSG)

    def is_logged_in(self) -> bool:
        return self.is_visible(LoginLocators.USER_AVATAR)
```

## Locators File

```python
# locators/login_locators.py
from selenium.webdriver.common.by import By

class LoginLocators:
    EMAIL_INPUT    = (By.ID, "email")
    PASSWORD_INPUT = (By.NAME, "password")
    SUBMIT_BTN     = (By.CSS_SELECTOR, "button[type='submit']")
    ERROR_MSG      = (By.XPATH, "//div[contains(@class,'error')]")
    USER_AVATAR    = (By.CSS_SELECTOR, "[data-testid='user-avatar']")
```

## Root conftest.py

```python
# conftest.py (root)
import pytest
from utils.driver_factory import DriverFactory
from configs.config import Config

@pytest.fixture(scope="session")
def config():
    return Config()

@pytest.fixture(scope="function")
def driver(config):
    driver = DriverFactory.create(browser="chrome", headless=config.headless)
    yield driver
    driver.quit()
```

## Driver Factory

```python
# utils/driver_factory.py
from selenium import webdriver
from selenium.webdriver.chrome.options import Options as ChromeOptions
from selenium.webdriver.firefox.options import Options as FirefoxOptions

class DriverFactory:
    @staticmethod
    def create(browser: str = "chrome", headless: bool = True):
        if browser == "chrome":
            opts = ChromeOptions()
            if headless:
                opts.add_argument("--headless=new")
            opts.add_argument("--no-sandbox")
            opts.add_argument("--disable-dev-shm-usage")
            opts.add_argument("--window-size=1920,1080")
            return webdriver.Chrome(options=opts)
        elif browser == "firefox":
            opts = FirefoxOptions()
            if headless:
                opts.add_argument("--headless")
            return webdriver.Firefox(options=opts)
        raise ValueError(f"Unsupported browser: {browser}")
```

## Test File Example

```python
# tests/ui/test_login.py
import pytest
import allure
from pages.login_page import LoginPage

@allure.feature("Authentication")
class TestLogin:

    @pytest.fixture(autouse=True)
    def setup(self, driver, config):
        self.page = LoginPage(driver)
        self.page.open(config.base_url)

    @allure.story("Valid login")
    @pytest.mark.smoke
    def test_login_valid_credentials(self, valid_user):
        self.page.login(valid_user["email"], valid_user["password"])
        assert self.page.is_logged_in(), "User should be redirected to dashboard"

    @allure.story("Invalid login")
    @pytest.mark.regression
    @pytest.mark.parametrize("email,password,expected_error", [
        ("", "pass123", "Email is required"),
        ("bad@email.com", "wrongpass", "Invalid credentials"),
    ])
    def test_login_invalid_credentials(self, email, password, expected_error):
        self.page.login(email, password)
        assert expected_error in self.page.get_error_message()
```

## pytest.ini

```ini
[pytest]
testpaths = tests
addopts = -v --alluredir=reports/allure-results --tb=short
markers =
    smoke: Critical path
    regression: Full suite
    api: API tests
    ui: Browser tests
    ai_eval: LLM evaluations
    slow: Long-running tests
```

## Common Debugging Patterns

**Element not found:**
1. Check if selector is correct with DevTools
2. Add explicit wait — avoid `time.sleep()`
3. Check if inside iframe: `driver.switch_to.frame()`
4. Check if element is in shadow DOM

**Flaky tests:**
- Use retry plugin: `pip install pytest-rerunfailures`
- Add `@pytest.mark.flaky(reruns=2)`
- Use explicit waits over implicit waits (never mix both)

**Stale element:**
```python
# Re-fetch element inside a retry loop
from selenium.common.exceptions import StaleElementReferenceException

def safe_click(driver, locator, retries=3):
    for _ in range(retries):
        try:
            WebDriverWait(driver, 5).until(
                EC.element_to_be_clickable(locator)
            ).click()
            return
        except StaleElementReferenceException:
            continue
```