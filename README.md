# utils/ai_agent.py
import os
import json
from httpx import TimeoutException
import openai
import time
import random
from dotenv import load_dotenv
from appium.webdriver.webdriver import WebDriver
from selenium.webdriver.common.by import By
from selenium.common.exceptions import NoSuchElementException, NoAlertPresentException
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import re
import xml.etree.ElementTree as ET
from typing import Any

from utils.teams_login_ios_browserstack import _wait_for_element
from .android_login import AndroidTeamsLogin, create_login_handler
from .helper import reset_android_emulator
import subprocess
from appium.webdriver.common.appiumby import AppiumBy
from selenium.common.exceptions import TimeoutException

load_dotenv()
client = openai.AzureOpenAI(
    api_key=os.getenv("AZURE_OPENAI_API_KEY"),
    api_version=os.getenv("AZURE_OPENAI_API_VERSION"),
    base_url=f"{os.getenv('AZURE_OPENAI_ENDPOINT')}/openai/deployments/{os.getenv('AZURE_OPENAI_DEPLOYMENT')}"
)

def try_parse_json_with_fallback(json_str: str) -> Any:
    try:
        return json.loads(json_str)
    except json.JSONDecodeError:
        # Try to fix common issues: remove trailing commas, fix quotes, etc.
        json_str = re.sub(r'\n', ' ', json_str)
        json_str = re.sub(r',([ \t\r\n]*[}}\]])', r'\1', json_str)
        # If missing closing brace, add it
        open_braces = json_str.count('{')
        close_braces = json_str.count('}')
        if close_braces < open_braces:
            json_str = json_str + ('}' * (open_braces - close_braces))
        try:
            return json.loads(json_str)
        except Exception:
            pass
        print("[LLM RAW OUTPUT]", json_str)
        raise

class AIAdhocAgent:
    def __init__(self):
        self.email = os.getenv("LOGIN_EMAIL", "redlabautomationtflmsaprod070@outlook.com")
        self.password = os.getenv("LOGIN_PASSWORD", "08Apples")
        self.email_entered = False
        self.password_entered = False
        self.logged_in = False
        self.post_login_mode = False
        self.tested_elements = {}
        self.max_element_retries = 3
        self.explored_features = set()
        self.feature_weights = {
            "chat": 1.0,
            "meetings": 1.0,
            "calls": 1.0,
            "calendar": 1.0,
            "settings": 1.0,
            "files": 1.0,
            "teams": 1.0,
            "activity": 1.0
        }
        self.failed_elements = {}
        self.bottom_nav_buttons = ["Activity", "Chat", "Calendar", "Calls", "Files"]
        self.bottom_nav_tested = {btn: False for btn in self.bottom_nav_buttons}
        self.login_handler = None

    def click_first_matching_text(self, driver: WebDriver, candidate_texts):
        for text in candidate_texts:
            try:
                el = driver.find_element(By.XPATH, f"//android.widget.Button[contains(@text, '{text}')]")
                if el.is_enabled():
                    print(f"[Info] Clicking '{text}'")
                    el.click()
                    time.sleep(2)
                    return True
            except Exception:
                pass
            try:
                el2 = driver.find_element(By.XPATH, f"//*[contains(@text, '{text}')]")
                if el2.is_enabled():
                    print(f"[Info] Clicking '{text}' (generic element)")
                    el2.click()
                    time.sleep(2)
                    return True
            except Exception:
                pass
        return False

    def switch_to_webview_if_available(self, driver: WebDriver) -> bool:
        try:
            contexts = driver.contexts
            for ctx in contexts:
                if ctx.upper().startswith("WEBVIEW"):
                    print(f"[Info] Switching context to {ctx}")
                    driver.switch_to.context(ctx)
                    time.sleep(1)
                    return True
        except Exception as e:
            print(f"[Info] Could not enumerate/switch to WEBVIEW: {e}")
        return False

    def switch_back_to_native(self, driver: WebDriver):
        try:
            driver.switch_to.context("NATIVE_APP")
            time.sleep(1)
        except Exception:
            pass

    def perform_complete_android_login(self, driver: WebDriver) -> bool:
        """
        Perform complete Android Teams login using the new login handler
        
        Args:
            driver: WebDriver instance
            
        Returns:
            True if login successful, False otherwise
        """
        try:
            print("[AI Agent] Starting complete Android login process")
            
            # Initialize login handler if not already done
            if self.login_handler is None:
                self.login_handler = create_login_handler(driver)
            
            # Perform the complete login flow
            login_success = self.login_handler.perform_complete_login()
            
            if login_success:
                print("[AI Agent] Login successful! Setting logged_in flag")
                self.logged_in = True
                self.post_login_mode = True
                return True
            else:
                print("[AI Agent] Login failed")
                return False
                
        except Exception as e:
            print(f"[AI Agent] Error in complete Android login: {str(e)}")
            return False

    def check_login_status(self, driver: WebDriver) -> bool:
        """
        Check if user is currently logged in using the login handler
        
        Args:
            driver: WebDriver instance
            
        Returns:
            True if logged in, False otherwise
        """
        try:
            if self.login_handler is None:
                self.login_handler = create_login_handler(driver)
            
            is_logged_in = self.login_handler.is_logged_in()
            if is_logged_in:
                self.logged_in = True
                self.post_login_mode = True
            
            return is_logged_in
            
        except Exception as e:
            print(f"[AI Agent] Error checking login status: {str(e)}")
            return False

    def perform_web_ms_login(self, driver: WebDriver) -> bool:
        """Perform Microsoft web login in a WEBVIEW (e.g., Chrome)."""
        try:
            # Email step
            try:
                email_input = driver.find_element(By.XPATH, "//input[@name='loginfmt' or @type='email']")
                email_input.clear()
                email_input.send_keys(self.email)
                time.sleep(1)
                next_btn = driver.find_element(By.XPATH, "//*[@id='idSIButton9' or contains(@data-report-event, 'Signin_Submit') or contains(., 'Next')]")
                next_btn.click()
                time.sleep(2)
            except Exception:
                pass

            # Password step
            try:
                pwd_input = driver.find_element(By.XPATH, "//input[@name='passwd' or @type='password']")
                pwd_input.clear()
                pwd_input.send_keys(self.password)
                time.sleep(1)
                sign_in_btn = driver.find_element(By.XPATH, "//*[@id='idSIButton9' or contains(., 'Sign in') or contains(., 'Next')]")
                sign_in_btn.click()
                time.sleep(3)
            except Exception:
                pass

            # Stay signed in? - choose No
            try:
                no_btn = driver.find_element(By.XPATH, "//*[@id='idBtn_Back' or contains(., 'No')]")
                no_btn.click()
                time.sleep(2)
            except Exception:
                pass

            return True
        except Exception as e:
            print(f"[Info] perform_web_ms_login error: {e}")
            return False

    def wait_for_package(self, driver: WebDriver, target_package: str, timeout_sec: int = 60) -> bool:
        try:
            start = time.time()
            while time.time() - start < timeout_sec:
                try:
                    pkg = driver.current_package
                except Exception:
                    pkg = None
                if pkg == target_package:
                    return True
                time.sleep(1)
        except Exception:
            pass
        return False

    def is_password_screen(self, xml: str) -> bool:
        return "password" in xml.lower() or "i0118" in xml.lower()

    def decide_next_action(self, xml_source: str) -> dict:
        if self.email_entered and not self.password_entered and self.is_password_screen(xml_source):
            return {
                "action": "type",
                "locator": "resource-id",
                "value": "i0118",
                "input_text": self.password,
                "description": "Typing password in password field",
                "expected_behaviour": "The app should accept the password and proceed to the next login step or main screen.",
                "screen_type": "login_password",
                "screen_description": "Password entry screen during login."
            }

        untested_nav_buttons = [btn for btn, tested in self.bottom_nav_tested.items() if not tested]

        prompt = f"""
        You are an intelligent Android app testing AI for the Microsoft Teams app.

        Testing status:
        - Logged in: {self.logged_in}
        - Untested bottom navigation buttons: {untested_nav_buttons}

        First, analyze the following UI XML hierarchy and determine what type of screen is currently displayed. 
        Possible screen types include: chat, meetings, calls, calendar, settings, files, bot_details, app_store, login, etc.

        Respond with a JSON in the following format:
        {{
          "screen_type": "app_store" | "files" | "chat" | ...,
          "screen_description": "Short description of the current screen and its main purpose.",
          "action": {{
            "action": "click" | "type" | "swipe",
            "locator": "text" | "resource-id" | "content-desc" | "class",
            "value": "value for the locator (text or id)",
            "input_text": "only if action is 'type', the input to send",
            "description": "Short description of the step",
            "feature": "main feature being tested (chat/meetings/calls/etc)",
            "expected_behaviour": "Describe what should happen if this step passes (e.g., 'The app should open the meeting interface and allow the user to join the meeting.')"
          }}
        }}

        Rules:
        - Only suggest actions that make sense for the detected screen type.
        - After login, if the bottom navigation bar is visible, you should include testing the untested buttons in your test strategy.
        - Do not test bottom navigation buttons before the main app screen is visible.
        - Do not get stuck just testing navigation. Mix it with other feature testing.
        - If the screen is not related to a major feature (e.g., it's a bot details page), suggest actions relevant to that context (e.g., add bot, view info, go back).
        - Do not attempt to test features that are not present on the current screen.
        - If the screen is a details/info page, suggest navigation or context-appropriate actions.

        Here's the UI XML hierarchy:
        {json.dumps(clickable_elements, indent=2)}
        """

        response = client.chat.completions.create(
            model=os.getenv("AZURE_OPENAI_DEPLOYMENT"),
            messages=[{"role": "user", "content": prompt}],
            temperature=0.7,
        )
        parsed = json.loads(response.choices[0].message.content)
        action = parsed["action"]
        action["screen_type"] = parsed.get("screen_type", "unknown")
        action["screen_description"] = parsed.get("screen_description", "")

        # Skip actions for elements that have failed 3 times
        value = action.get("value")
        if value and self.failed_elements.get(value, 0) >= 3:
            print(f"[Info] Skipping action for '{value}' as it has failed 3 times.")
            return {
                "action": "noop",
                "description": f"Skipping '{value}' after 3 failed attempts.",
                "screen_type": action.get("screen_type", "unknown"),
                "screen_description": action.get("screen_description", "")
            }

        # Update feature weights based on exploration
        if "feature" in action:
            feature = action["feature"].lower()
            if feature in self.feature_weights:
                self.feature_weights[feature] *= 0.5
                self.explored_features.add(feature)
        
        return action

    def handle_native_android_dialog(self, driver) -> bool:
        """
        Detect and handle native Android dialogs (e.g., confirmation dialogs).
        
        Tries multiple locator strategies to find the positive button (OK, Allow, Yes, etc.):
        1. android:id/button1 (standard positive button resource ID)
        2. UiAutomator strategy for android:id/button1
        3. XPath with resource-id
        4. XPath with text matching common button labels
        
        Returns True if a button was found and clicked, False otherwise.
        """
        try:
            # Strategy 1: Try android:id/button1 via ID locator
            try:
                button = driver.find_element(AppiumBy.ID, "android:id/button1")
                if button.is_displayed() and button.is_enabled():
                    print("[Dialog] Found native dialog button via ID 'android:id/button1', clicking...")
                    button.click()
                    time.sleep(1)
                    return True
            except NoSuchElementException:
                pass
            
            # Strategy 2: Try UiAutomator2 selector for android:id/button1
            try:
                button = driver.find_element(AppiumBy.ANDROID_UIAUTOMATOR, 
                                            'new UiSelector().resourceId("android:id/button1")')
                if button.is_displayed() and button.is_enabled():
                    print("[Dialog] Found native dialog button via UiAutomator resourceId 'android:id/button1', clicking...")
                    button.click()
                    time.sleep(1)
                    return True
            except NoSuchElementException:
                pass
            
            # Strategy 3: Try XPath with resource-id attribute
            try:
                button = driver.find_element(AppiumBy.XPATH, 
                                            '//android.widget.Button[@resource-id="android:id/button1"]')
                if button.is_displayed() and button.is_enabled():
                    print("[Dialog] Found native dialog button via XPath resourceId, clicking...")
                    button.click()
                    time.sleep(1)
                    return True
            except NoSuchElementException:
                pass
            
            # Strategy 4: Try common button text labels
            common_button_texts = ["OK", "Yes", "Accept", "Allow", "Continue", "Confirm"]
            for button_text in common_button_texts:
                try:
                    button = driver.find_element(AppiumBy.XPATH,
                                                f'//android.widget.Button[@text="{button_text}"]')
                    if button.is_displayed() and button.is_enabled():
                        print(f"[Dialog] Found native dialog button with text '{button_text}', clicking...")
                        button.click()
                        time.sleep(1)
                        return True
                except NoSuchElementException:
                    continue
            
            # No dialog button found
            return False
            
        except Exception as e:
            print(f"[Dialog] Error checking for native dialog: {e}")
            return False

    def dismiss_alert_if_present(self, driver):
        """
        Dismiss browser alerts (web-only, not for mobile native dialogs).
        Note: Native Android dialogs are handled separately by handle_native_android_dialog().
        This function is kept for legacy web automation support.
        """
        try:
            # Skip alert handling on mobile - use handle_native_android_dialog() instead
            # This method is only for web-based alerts
            alert = driver.switch_to.alert
            alert_text = alert.text
            print(f"[Info] Browser Alert detected: {alert_text}")
            
            # If it's a "Leave community" alert, accept it
            if "leave community" in alert_text.lower():
                print("[Info] Detected 'Leave community' browser alert, accepting...")
                alert.accept()
            else:
                # For other alerts, dismiss them
                alert.dismiss()

        except NoAlertPresentException:
            pass
        except Exception as e:
            # Handle UiAutomator2 crash during alert fetch
            msg = str(e)
            if "UiAutomator2" in msg or "instrumentation process is not running" in msg:
                print("[Warning] UiAutomator2 not responding while reading alert. Attempting recovery...")
                self.recover_uiautomator2(driver)
            else:
                print(f"[Info] No browser alert present or access failed: {e}")
                pass

    def handle_native_dialog(self, driver, button_text: str = "Leave"):
        """Handle native Android/iOS dialog buttons.
        Args:
            driver: WebDriver instance
            button_text: Text of the button to click (e.g., "Leave", "Cancel", "Export content")
        """
        try:
            # Try to find button by text content
            button = driver.find_element(By.XPATH, f"//android.widget.Button[contains(@text, '{button_text}')]")
            if button.is_displayed():
                print(f"[Info] Found '{button_text}' button, clicking...")
                button.click()
                time.sleep(1)
                return True
        except NoSuchElementException:
            pass
        except Exception as e:
            print(f"[Info] Error clicking '{button_text}' button: {e}")
        
        return False

    def detect_screen_type(self, driver):
        """Detects the current screen type using known UI elements."""
        try:
            if self.is_on_chat_screen(driver):
                return "chat"
            # Add more screen detection logic here
            try:
                driver.find_element(By.ID, "com.microsoft.teams:id/meetings_list_view")
                return "meetings"
            except Exception:
                pass
            try:
                driver.find_element(By.ID, "com.microsoft.teams:id/calls_list_view")
                return "calls"
            except Exception:
                pass
            try:
                driver.find_element(By.ID, "com.microsoft.teams:id/calendar_view")
                return "calendar"
            except Exception:
                pass
            try:
                driver.find_element(By.ID, "com.microsoft.teams:id/files_list_view")
                return "files"
            except Exception:
                pass
            # Add more as needed
        except Exception:
            pass
        return "unknown"


    def perform_action(self, driver: WebDriver, action: dict):
        """
        Execute a UI action on the mobile app.
        
        First checks for native Android dialogs (sign-out, permissions, confirmations).
        Then validates action structure before execution.
        """
        # FIRST: Always check for native Android dialog buttons (sign-out, permission, confirmation)
        # This must happen BEFORE any other action processing
        if self.handle_native_android_dialog(driver):
            print("[Info] Native Android dialog handled automatically")
            return

        # SECOND: Handle "Leave community" dialog if it appears
        is_leave_action = "leave" in action.get("description", "").lower() or "leave community" in action.get("description", "").lower()
        
        if is_leave_action:
            # Try to handle alert first
            try:
                alert = driver.switch_to.alert
                alert_text = alert.text
                print(f"[Info] Alert: {alert_text}")
                
                if "leave community" in alert_text.lower():
                    print("[Info] Detected 'Leave community' alert, accepting to click Leave button...")
                    alert.accept()
                    print("[Success] Leave button clicked via alert accept")
                    time.sleep(1)
                    return  # Exit early - action is complete
            except NoAlertPresentException:
                pass  # No alert, continue with normal element click
            except Exception as e:
                print(f"[Info] Error handling alert: {e}")

        # Handle permission dialogs (alerts and system dialogs) before dismissing other alerts
        description = action.get("description", action.get("action", ""))
        desc_lower = description.lower()
        is_permission_action = any(keyword in desc_lower for keyword in ["allow", "permission", "access", "grant"])
        
        if is_permission_action and action.get("action") == "click":
            print(f"[Permission] Detected permission dialog action: {description}")
            
            # First, try to handle as an alert
            try:
                alert = driver.switch_to.alert
                alert_text = alert.text
                print(f"[Permission] Alert detected: {alert_text}")
                
                # For permission alerts, accept them
                if any(keyword in alert_text.lower() for keyword in ["allow", "access", "permission", "photos", "videos", "media"]):
                    print("[Permission] Accepting permission alert...")
                    alert.accept()
                    print("[Permission] Permission alert accepted successfully")
                    time.sleep(2)
                    return  # Success, exit early
            except NoAlertPresentException:
                print("[Permission] No alert present, trying button-based permission handling...")
            except Exception as e:
                print(f"[Permission] Error handling permission alert: {e}, trying button-based handling...")
            
            # If no alert, try button-based handling
            # Define candidate button texts for permission dialogs
            permission_texts = ["Allow all", "Allow", "While using the app", "OK"]
            
            # Try to find and click any matching button with explicit wait
            from selenium.webdriver.support.ui import WebDriverWait
            from selenium.webdriver.support import expected_conditions as EC
            from appium.webdriver.common.appiumby import AppiumBy
            
            wait = WebDriverWait(driver, 10)  # 10 second explicit wait
            
            for text in permission_texts:
                try:
                    # Try XPath with text contains
                    xpath = f"//android.widget.Button[contains(@text, '{text}')]"
                    element = wait.until(EC.element_to_be_clickable((AppiumBy.XPATH, xpath)))
                    print(f"[Permission] Found and clicking button with text: '{text}'")
                    element.click()
                    time.sleep(2)
                    return  # Success, exit early
                except Exception:
                    try:
                        # Try exact text match
                        xpath = f"//android.widget.Button[@text='{text}']"
                        element = wait.until(EC.element_to_be_clickable((AppiumBy.XPATH, xpath)))
                        print(f"[Permission] Found and clicking button with exact text: '{text}'")
                        element.click()
                        time.sleep(2)
                        return  # Success, exit early
                    except Exception:
                        continue
            
            # Fallback to resource-id patterns for permission dialogs
            permission_resource_ids = [
                "com.android.permissioncontroller:id/permission_allow_button",
                "com.android.permissioncontroller:id/permission_allow_all_button",
                "com.android.permissioncontroller:id/permission_allow_foreground_only_button",
                "android:id/button1",  # Generic positive button
            ]
            
            for res_id in permission_resource_ids:
                try:
                    element = wait.until(EC.element_to_be_clickable((AppiumBy.ID, res_id)))
                    print(f"[Permission] Found and clicking button with resource-id: '{res_id}'")
                    element.click()
                    time.sleep(2)
                    return  # Success, exit early
                except Exception:
                    continue
            
            # Final fallback to class name
            try:
                buttons = driver.find_elements(AppiumBy.CLASS_NAME, "android.widget.Button")
                for btn in buttons:
                    if btn.is_displayed() and btn.is_enabled():
                        btn_text = btn.get_attribute("text") or ""
                        if any(perm_text.lower() in btn_text.lower() for perm_text in permission_texts):
                            print(f"[Permission] Found and clicking button with class fallback, text: '{btn_text}'")
                            btn.click()
                            time.sleep(2)
                            return  # Success, exit early
            except Exception:
                pass
            
            # If we reach here, none of the permission handling worked
            print(f"[Permission] Could not find any permission button for: {description}")
            raise Exception(f"Could not locate element to {description}")

        self.dismiss_alert_if_present(driver)
        locator = action["locator"] if "locator" in action else None
        value = action["value"] if "value" in action else None
        has_value = isinstance(value, str) and len(value.strip()) > 0
        description = action.get("description", action.get("action", ""))
        expected_behaviour = action.get("expected_behaviour", "No expected behaviour provided.")

        # Screen validation step
        intended_screen = action.get("screen_type", "unknown").lower()
        detected_screen = self.detect_screen_type(driver)
        if intended_screen != "unknown" and detected_screen != "unknown" and intended_screen != detected_screen:
            print(f"[Warning] Intended action for screen '{intended_screen}', but detected screen is '{detected_screen}'. Skipping action: {description}")
            return

        if action.get("action") == "noop":
            print(description)
            return

        # Handle wait action
        if action.get("action") == "wait":
            wait_time = 1  # Default wait time
            if "wait" in description.lower():
                # Try to extract wait time from description
                import re
                time_match = re.search(r'(\d+)\s*seconds?', description.lower())
                if time_match:
                    wait_time = int(time_match.group(1))
            print(f"[Wait] Waiting for {wait_time} seconds...")
            time.sleep(wait_time)
            return

        if value in self.bottom_nav_buttons:
            self.bottom_nav_tested[value] = True

        # Handle direct coordinate tap BEFORE any element lookup
        if action.get("action") == "tap":
            x = action.get("x")
            y = action.get("y")
            if (x is None or y is None) and description:
                try:
                    import re
                    m = re.search(r"x\s*(?:is|=)\s*(\d+).*?y\s*(?:is|=)\s*(\d+)", description, re.IGNORECASE)
                    if not m:
                        m = re.search(r"\((\d+)\s*,\s*(\d+)\)", description)
                    if m:
                        x = int(m.group(1))
                        y = int(m.group(2))
                except Exception:
                    pass
            if x is None or y is None:
                raise ValueError("Tap action requires numeric 'x' and 'y' coordinates")
            print(f"[Tap] at coordinates ({x}, {y}) for: {description}")
            try:
                driver.tap([(int(x), int(y))])
            except Exception as e:
                # Fallback to W3C actions if driver.tap is not supported
                try:
                    actions = TouchAction(driver)
                    actions.tap(x=int(x), y=int(y)).perform()
                except Exception:
                    raise e
            time.sleep(1)
            return

        element = None
        locator_tried = []
        try:
            # Always try content-desc (accessibility id) first for all actions, only if value provided
            if has_value:
                try:
                    element = driver.find_element("accessibility id", value)
                    locator_tried.append("content-desc")
                except Exception:
                    pass
            # If explicit XPath is provided, honor it first
            if element is None and locator == "xpath" and value:
                try:
                    element = driver.find_element("xpath", value)
                    locator_tried.append("xpath")
                except Exception:
                    pass
            # Try the primary locator if not already found
            if element is None:
                if locator == "text":
                    if has_value:
                        try:
                            element = driver.find_element("xpath", f"//*[contains(@text, '{value}')]")
                            locator_tried.append("text")
                        except Exception:
                            pass
                elif locator == "resource-id":
                    if has_value:
                        try:
                            element = driver.find_element("id", value)
                            locator_tried.append("resource-id")
                        except Exception:
                            pass
                elif locator == "class":
                    if has_value:
                        try:
                            element = driver.find_element("class name", value)
                            locator_tried.append("class")
                        except Exception:
                            pass
            # Fallback: try other locator types if not found
            if element is None:
                generic_candidates = []
                # Try based on provided value first
                if has_value:
                    generic_candidates.extend([
                        ("text", ("xpath", f"//*[contains(@text, '{value}')]")),
                        ("resource-id", ("id", value)),
                        ("class", ("class name", value)),
                        ("xpath", ("xpath", f"//*[@resource-id='{value}']")),
                    ])

                # If we don't have a usable value, infer from the description (e.g., "Click on 'SKIP' button")
                if not has_value:
                    desc_lower = (description or "").lower()
                    import re
                    # Prefer any quoted label inside the description, else pick likely keywords
                    label = None
                    m = re.search(r"'([^']+)'|\"([^\"]+)\"", description or "")
                    if m:
                        label = m.group(1) or m.group(2)
                    else:
                        for token in ["sign in", "signin", "next", "continue", "submit", "allow", "yes", "no", "skip", "not now", "maybe later"]:
                            if token in desc_lower:
                                label = token
                                break
                    if label:
                        generic_candidates.extend([
                            ("text", ("xpath", f"//*[contains(@text, '{label}') or contains(@content-desc, '{label}') or contains(translate(@text,'abcdefghijklmnopqrstuvwxyz','ABCDEFGHIJKLMNOPQRSTUVWXYZ'), '{label.upper()}') ]")),
                            ("button-text", ("xpath", f"//android.widget.Button[contains(@text, '{label}') or contains(@content-desc, '{label}') ]")),
                        ])

                for loc, find_args in generic_candidates:
                    if loc in locator_tried:
                        continue
                    try:
                        element = driver.find_element(*find_args)
                        print(f"[Fallback] Found element using locator: {loc}")
                        break
                    except Exception:
                        continue

            # Special handling for clicking into chat compose/message box
            if element is None and action.get("action") == "click":
                desc_lower = (description or "").lower()
                val_lower = (value or "").lower()
                if ("type a message" in desc_lower or "message" in desc_lower or not has_value):
                    # Try a variety of likely selectors for Teams compose box
                    # Prefer direct resource-id for Teams compose editor
                    try:
                        element = driver.find_element("id", "com.microsoft.skype.teams.dev:id/message_area_edit_text")
                        print("[Fallback-Compose] Found compose box via resource-id message_area_edit_text")
                    except Exception:
                        element = None
                    compose_xpaths = [
                        "//*[@content-desc='Type a message']",
                        "//*[contains(@content-desc, 'Type a message') or contains(@text, 'Type a message')]",
                        "//android.widget.EditText[contains(@text, 'message') or contains(@content-desc, 'message')]",
                        "//android.widget.EditText[@enabled='true' and @clickable='true']",
                        "//android.widget.EditText[@resource-id='com.microsoft.skype.teams.dev:id/message_area_edit_text']",
                         "//android.widget.Button[@clickable='true']",
                        "//android.widget.TextView[@clickable='true']",
                        "//android.widget.ImageView[@clickable='true']",
                        "//*[@clickable='true' and @enabled='true']"
                    ]
                    if element is None:
                        for xp in compose_xpaths:
                            try:
                                element = driver.find_element("xpath", xp)
                                print(f"[Fallback-Compose] Found compose box via XPath: {xp}")
                                break
                            except Exception:
                                continue
                    # As a last resort, click the last visible EditText on screen (usually bottom compose)
                    if element is None:
                        try:
                            edit_texts = driver.find_elements("class name", "android.widget.EditText")
                            if edit_texts:
                                element = edit_texts[-1]
                                print("[Fallback-Compose] Using last EditText on screen")
                        except Exception:
                            pass

            # Special handling for typing when no reliable locator/value was provided
            if element is None and action.get("action") == "type":
                try:
                    # Prefer currently focused editable field
                    element = driver.find_element(
                        "xpath",
                        "//*[@focused='true' and contains(@class, 'EditText')]"
                    )
                    print("[Fallback-Type] Using focused EditText field")
                except Exception:
                    try:
                        # Try direct resource-id for Teams compose editor first
                        element = driver.find_element("id", "com.microsoft.skype.teams.dev:id/message_area_edit_text")
                        print("[Fallback-Type] Using compose EditText by resource-id message_area_edit_text")
                    except Exception:
                        element = None
                    try:
                        # Fall back to the first visible EditText on screen
                        element = driver.find_element(
                            "xpath",
                            "//android.widget.EditText"
                        )
                        print("[Fallback-Type] Using first EditText on screen")
                    except Exception:
                        pass

            # Special fallback for 'Next' button when value is missing/wrong
            if element is None:
                desc_lower = (description or "").lower()
                value_lower = (value or "").lower() if isinstance(value, str) else ""
                if "next" in desc_lower or value in (None, "", "null") or "next" in value_lower:
                    try:
                        # Try well-known resource-id first
                        element = driver.find_element("id", "com.microsoft.skype.teams.dev:id/action_next")
                        print("[Fallback-Next] Found by resource-id '...:id/action_next'")
                    except Exception:
                        try:
                            # Try exact XPath provided
                            element = driver.find_element("xpath", "//android.widget.Button[@resource-id=\"com.microsoft.skype.teams.dev:id/action_next\"]")
                            print("[Fallback-Next] Found by XPath action_next button")
                        except Exception:
                            try:
                                # Try accessibility ids 'next' then 'vnext'
                                element = driver.find_element("accessibility id", "next")
                                print("[Fallback-Next] Found by accessibility id 'next'")
                            except Exception:
                                try:
                                    element = driver.find_element("accessibility id", "next")
                                    print("[Fallback-Next] Found by accessibility id 'next'")
                                except Exception:
                                    try:
                                        element = driver.find_element("xpath", "//*[contains(@content-desc, 'next') or contains(@content-desc, 'Next') or contains(@content-desc, 'NEXT')]")
                                        print("[Fallback-Next] Found by content-desc contains 'next'/'Next'")
                                    except Exception:
                                        try:
                                            element = driver.find_element("xpath", "//*[contains(@text, 'Next') or contains(@text, 'NEXT')]")
                                            print("[Fallback-Next] Found by text contains 'Next'")
                                        except Exception:
                                            try:
                                                element = driver.find_element("xpath", "//android.widget.Button[contains(@text, 'Next') or contains(@content-desc, 'Next')]")
                                                print("[Fallback-Next] Found by button with Next")
                                            except Exception:
                                                pass
                                except Exception:
                                    pass
            # Special fallback for 'Skip' button (e.g., Add members screen)
            if element is None and action.get("action") == "click":
                desc_lower = (description or "").lower()
                value_lower = (value or "").lower() if isinstance(value, str) else ""
                if "skip" in desc_lower or "skip" in value_lower:
                    try:
                        element = driver.find_element("id", "com.microsoft.skype.teams.dev:id/action_bar_add_members")
                        print("[Fallback-Skip] Found by resource-id '...:id/action_bar_add_members'")
                    except Exception:
                        try:
                            element = driver.find_element("xpath", "//android.widget.Button[@resource-id=\"com.microsoft.skype.teams.dev:id/action_bar_add_members\"]")
                            print("[Fallback-Skip] Found by XPath action_bar_add_members")
                        except Exception:
                            try:
                                element = driver.find_element("xpath", "//android.widget.Button[contains(@text, 'SKIP') or contains(@text, 'Skip') or contains(@content-desc, 'Skip')]")
                                print("[Fallback-Skip] Found button with text/content 'Skip'")
                            except Exception:
                                pass
            if element is None:
                raise NoSuchElementException(f"Could not locate element to '{description}' (locator: {locator}, value: '{value}'). Tried all locator strategies.")

            if value in self.tested_elements and self.tested_elements[value] >= self.max_element_retries:
                print(f"[Info] Element {value} has been tested {self.tested_elements[value]} times, trying alternative path...")
                if not self.try_alternative_path(driver, action):
                    print(f"[Info] Skipping element {value} as no alternative path found")
                    return
                
            if action["action"] == "click":
                if not self.logged_in and any(x in value.lower() for x in ["forgot", "trouble", "reset"]):
                    raise Exception(f"Cannot click on {value} during login process")
                print(f"[Clicking] {value}")
                try:
                    element.click()
                except Exception:
                    # If click fails, try tapping center via coordinates
                    try:
                        rect = element.rect
                        x = rect.get("x", 0) + rect.get("width", 0) // 2
                        y = rect.get("y", 0) + rect.get("height", 0) // 2
                        driver.tap([(x, y)])
                        print("[Fallback-Click] Tapped element center via coordinates")
                    except Exception:
                        raise
                time.sleep(2)

            elif action["action"] == "type":
                # Check if we're on external login page (Microsoft login)
                is_external_login = self.is_on_external_login_page(driver)
                if self.logged_in and any(k in value.lower() for k in ["email", "username", "password", "passwd"]) and not is_external_login:
                    raise Exception(f"Cannot type into {value} after login")

                # Start with any explicit input_text from the action
                input_text = action.get("input_text", "")

                # Override with credentials when targeting auth fields
                if any(k in value.lower() for k in ["email", "username"]):
                    input_text = self.email
                    self.email_entered = True
                elif any(k in value.lower() for k in ["password", "passwd", "i0118"]):
                    input_text = self.password
                    self.password_entered = True
                elif not input_text and ("meeting" in value.lower() or "id" in value.lower()):
                    input_text = f"{random.randint(1000000000,9999999999)}"
                elif not input_text and ("message" in value.lower() or "chat" in value.lower()):
                    input_text = random.choice(["Hello", "Hi", "Testing...", "This is AI!"])

                # If still no input_text, give up gracefully this turn (do not block when action provided text)
                if not input_text:
                    key = value or description
                    self.failed_elements[key] = self.failed_elements.get(key, 0) + 1
                    if self.failed_elements[key] >= 3:
                        print(f"[Info] Skipping typing into '{key}' after 3 failed attempts.")
                        return
                    else:
                        print(f"[Warning] No input_text resolved for '{key}'. Attempt {self.failed_elements[key]} of 3.")
                        return

                print(f"[Typing] Into '{value}': {input_text}")
                try:
                    element.click()
                except Exception:
                    pass
                try:
                    element.clear()
                except Exception:
                    pass
                try:
                    element.send_keys(input_text)
                except Exception as e:
                    print(f"[Typing] send_keys failed: {e}. Trying mobile: replaceElementValue / type fallbacks...")
                    # Try replacing element value directly
                    replaced = False
                    try:
                        driver.execute_script("mobile: replaceElementValue", {
                            "elementId": element.id,
                            "text": input_text
                        })
                        print("[Typing] Used mobile: replaceElementValue successfully")
                        replaced = True
                    except Exception as e2:
                        print(f"[Typing] replaceElementValue failed: {e2}")
                    if not replaced:
                        try:
                            # As a last resort, type via IME
                            driver.execute_script("mobile: type", {"text": input_text})
                            print("[Typing] Used mobile: type successfully")
                        except Exception as e3:
                            print(f"[Typing] mobile: type failed: {e3}. Trying focused EditText...")
                            try:
                                focused = driver.find_element(
                                    "xpath",
                                    "//*[@focused='true' and contains(@class, 'EditText')]"
                                )
                                focused.send_keys(input_text)
                                print("[Typing] Sent keys to focused EditText")
                            except Exception:
                                raise
                time.sleep(3)

                self.handle_post_input_next(driver)

            elif action["action"] == "tap":
                # Direct coordinate tap. Accepts fields x and y; can also parse from description like "x is 613 and y is 955"
                x = action.get("x")
                y = action.get("y")
                if (x is None or y is None) and description:
                    try:
                        import re
                        # match patterns: x is 613 and y is 955, or x=613, y=955, or (613, 955)
                        m = re.search(r"x\s*(?:is|=)\s*(\d+).*?y\s*(?:is|=)\s*(\d+)", description, re.IGNORECASE)
                        if not m:
                            m = re.search(r"\((\d+)\s*,\s*(\d+)\)", description)
                        if m:
                            x = int(m.group(1))
                            y = int(m.group(2))
                    except Exception:
                        pass
                if x is None or y is None:
                    raise ValueError("Tap action requires numeric 'x' and 'y' coordinates")
                print(f"[Tap] at coordinates ({x}, {y})")
                driver.tap([(int(x), int(y))])
                time.sleep(1)

            elif action["action"] == "swipe":
                size = driver.get_window_size()
                start_x = size['width'] // 2
                start_y = size['height'] * 3 // 4
                end_y = size['height'] // 4
                driver.swipe(start_x, start_y, start_x, end_y, 800)
            else:
                raise ValueError(f"Unknown action type: {action['action']}")

            self.tested_elements[value] = self.tested_elements.get(value, 0) + 1

            if self.is_on_chat_screen(driver):
                if not self.logged_in:
                    self.logged_in = True
                    self.post_login_mode = True
                    print("[✔] Login successful. Entering AI-driven feature exploration mode...")

        except NoSuchElementException as e:
            error_msg = (
                f"Step failed: Could not locate element to \"{description}\" "
                f"(locator: {locator}, value: \"{value}\"). Failed the expected behaviour.\n"
                f"Expected behaviour: {expected_behaviour}"
            )
            print(error_msg)
            if value:
                self.failed_elements[value] = self.failed_elements.get(value, 0) + 1
                if self.failed_elements[value] >= 3:
                    print(f"[Info] Skipping '{value}' after 3 failed attempts.")
                    return
            raise Exception(error_msg) from e
        except Exception as e:
            error_msg = str(e)
            if "tested more than 3 times" in error_msg:
                print(f"[Info] {error_msg}")
                return
            # Auto-recover if UiAutomator2 instrumentation died
            if "UiAutomator2" in error_msg or "instrumentation process is not running" in error_msg:
                print("[Warning] UiAutomator2 appears to have crashed. Attempting recovery and retry once...")
                if self.recover_uiautomator2(driver):
                    try:
                        # brief wait for server to be ready
                        time.sleep(2)
                        # retry the same action once
                        return self.perform_action(driver, action)
                    except Exception as e2:
                        print(f"[Error] Retry after UiAutomator2 recovery failed: {e2}")
                else:
                    print("[Error] UiAutomator2 recovery failed.")
            raise

    def handle_post_input_next(self, driver: WebDriver):
        print("[Info] Trying to click 'Next' or 'Sign in'...")
        for text in ["Sign in", "Next", "Continue", "Submit", "OK", "Log In"]:
            try:
                button = driver.find_element(By.XPATH, f"//android.widget.Button[contains(@text, '{text}')]")
                if button.is_enabled():
                    print(f"[Info] Clicking '{text}'")
                    button.click()
                    time.sleep(3)
                    return
            except NoSuchElementException:
                continue
        print("[Warning] No post-input button found.")

    def try_common_ui_paths(self, driver: WebDriver):
        fallback_texts = ["Chat", "Calls", "Calendar", "Join", "Send", "Back", "Navigate up", "Message"]
        for text in fallback_texts:
            try:
                button = driver.find_element(By.XPATH, f"//*[contains(@text, '{text}')]")
                if button.is_enabled():
                    print(f"[Fallback] Clicking: {text}")
                    button.click()
                    time.sleep(2)
                    return True
            except:
                continue
        return False

    def is_on_chat_screen(self, driver):
        try:
            driver.find_element(By.XPATH, "//*[contains(@content-desc,'Chat tab')]")
            return True
        except NoSuchElementException:
            return False

    def is_on_external_login_page(self, driver):
        """
        Check if we're on an external Microsoft login page (not within Teams app)
        """
        try:
            # Check for Microsoft login page indicators
            microsoft_indicators = [
                "//*[contains(@text, 'Sign in')]",
                "//*[contains(@text, 'Microsoft')]",
                "//*[contains(@text, 'Enter email or phone number')]",
                "//*[contains(@text, 'Use your work, school, or personal Microsoft account')]",
                "//*[contains(@content-desc, 'Enter email or phone number')]"
            ]
            
            for indicator in microsoft_indicators:
                try:
                    driver.find_element(By.XPATH, indicator)
                    return True
                except NoSuchElementException:
                    continue
            
            # Also check if we're in a webview (Chrome) which indicates external login
            try:
                current_package = getattr(driver, "current_package", None)
                if current_package and "chrome" in current_package.lower():
                    return True
            except Exception:
                pass
                
            return False
        except Exception:
            return False

    def try_alternative_path(self, driver: WebDriver, action: dict) -> bool:
        """Try to find an alternative path to achieve the same action"""
        try:
            feature = action.get("feature", "").lower()
            alternative_elements = {
                "chat": ["message", "conversation", "chat_list", "new_chat"],
                "meetings": ["join", "schedule", "calendar", "meeting"],
                "calls": ["call", "dial", "phone", "contact"],
                "settings": ["settings", "profile", "preferences", "options"],
                "files": ["file", "document", "share", "upload"]
            }

            if feature in alternative_elements:
                for alt_text in alternative_elements[feature]:
                    try:
                        for locator_type in ["text", "content-desc", "resource-id"]:
                            try:
                                if locator_type == "text":
                                    element = driver.find_element("xpath", f"//*[contains(@text, '{alt_text}')]")
                                elif locator_type == "content-desc":
                                    element = driver.find_element("accessibility id", alt_text)
                                else:
                                    element = driver.find_element("id", alt_text)

                                if element.is_enabled():
                                    print(f"[Info] Found alternative path using {alt_text}")
                                    element.click()
                                    time.sleep(2)
                                    return True
                            except:
                                continue
                    except:
                        continue

            nav_elements = ["Back", "Navigate up", "Menu", "More options", "Home"]
            for nav_text in nav_elements:
                try:
                    nav = driver.find_element("xpath", f"//*[contains(@text, '{nav_text}')]")
                    if nav.is_enabled():
                        print(f"[Info] Using navigation element {nav_text} to find alternative path")
                        nav.click()
                        time.sleep(2)
                        return True
                except:
                    continue

            return False
        except:
            return False

    def ensure_logged_in(self, driver: WebDriver):
        """Ensure user is logged in. Handles AppCenter 'Get started' then Microsoft login, then Teams login."""
        try:
            # If already on chat/home screen, consider logged in
            if self.is_on_chat_screen(driver):
                if not self.logged_in:
                    self.logged_in = True
                    self.post_login_mode = True
                return

            # 1) AppCenter Get Started / Continue flows (if present)
            self.click_first_matching_text(driver, ["Get started", "Get Started", "Continue", "Continue as", "Sign in", "Sign in with Microsoft"])

            # If this opened Chrome (web), detect package and perform web login
            try:
                if getattr(driver, "current_package", None):
                    if driver.current_package and "chrome" in driver.current_package.lower():
                        print("[Info] Detected Chrome package; switching to WEBVIEW for web login...")
                        switched = self.switch_to_webview_if_available(driver)
                        if switched:
                            self.perform_web_ms_login(driver)
                        self.switch_back_to_native(driver)
                        # Wait until Teams package is active again, then foreground it (reduced timeout)
                        self.wait_for_package(driver, "com.microsoft.skype.teams.dev", timeout_sec=10)
                        try:
                            driver.activate_app("com.microsoft.skype.teams.dev")
                            time.sleep(1)  # Reduced from 2s
                        except Exception:
                            pass
            except Exception:
                pass

            # Try to find and fill email field (common id in Microsoft login webview)
            try:
                email_field = driver.find_element("id", "i0116")
                try:
                    email_field.clear()
                except Exception:
                    pass
                email_field.send_keys(self.email)
                self.email_entered = True
                time.sleep(1)
                # self.handle_post_input_next(driver)
                time.sleep(3)
            except Exception:
                pass

            # If on password screen, fill password
            try:
                if self.is_password_screen(driver.page_source):
                    pwd_field = driver.find_element("id", "i0118")
                    try:
                        pwd_field.clear()
                    except Exception:
                        pass
                    pwd_field.send_keys(self.password)
                    self.password_entered = True
                    time.sleep(1)
                    # Prefer explicit click on 'Sign in' if available
                    clicked = self.click_first_matching_text(driver, ["Sign in", "Next", "Continue"])
                    if not clicked:
                        self.handle_post_input_next(driver)
                    time.sleep(5)
            except Exception:
                pass

            # Handle potential "Stay signed in?" prompt - choose No
            try:
                try:
                    no_btn = driver.find_element("id", "idBtn_Back")
                    no_btn.click()
                    time.sleep(2)
                except Exception:
                    no_btn_text = driver.find_element(By.XPATH, "//android.widget.Button[contains(@text, 'No')]")
                    no_btn_text.click()
                    time.sleep(2)
            except Exception:
                pass

            # Sometimes there is a "Not now" or "Skip" after login
            self.click_first_matching_text(driver, ["Not now", "Skip", "Maybe later", "Done", "Continue"])

            # Final check
            if self.is_on_chat_screen(driver):
                self.logged_in = True
                self.post_login_mode = True
                print("[✔] Login successful. Reached Chat screen.")
            else:
                print("[Info] Chat screen not detected yet; continuing with test steps.")
        except Exception as e:
            print(f"[Warning] ensure_logged_in encountered an error: {e}")

    def is_feature_explored(self, feature: str) -> bool:
        """Check if a feature has been explored"""
        return feature.lower() in self.explored_features

    def get_exploration_status(self) -> dict:
        """Get the current exploration status of features"""
        return {
            "explored_features": list(self.explored_features),
            "feature_weights": self.feature_weights,
            "tested_elements": self.tested_elements
        }


class TestCaseLLMRunner:
    def __init__(self, driver, test_case, reporter, agent, max_tokens=8000):
        self.driver = driver
        self.test_case = test_case
        self.reporter = reporter
        self.agent = agent
        self.max_tokens = max_tokens
        self.client = client

    def summarize_ui_xml(self, xml_string, max_elements=50):
        """
        Parse the XML and return a summary of up to max_elements elements with their text, resource-id, and content-desc.
        """
        try:
            root = ET.fromstring(xml_string)
        except Exception as e:
            return f"[XML parse error: {e}]"
        summary = []
        def recurse(node):
            if len(summary) >= max_elements:
                return
            attribs = node.attrib
            text = attribs.get('text', '')
            resource_id = attribs.get('resource-id', '')
            content_desc = attribs.get('content-desc', '')
            if text or resource_id or content_desc:
                summary.append({
                    'text': text,
                    'resource-id': resource_id,
                    'content-desc': content_desc
                })
            for child in node:
                recurse(child)
        recurse(root)
        return summary

    def extract_json_from_response(self, response_text):
        match = re.search(r'({.*?})', response_text, re.DOTALL)
        if match:
            return match.group(1)
        raise ValueError("No JSON object found in LLM response.")

    def get_clickable_elements(self, driver):
        """
        Fast method to get clickable elements using page_source XML parsing.
        This is much faster than scanning all elements individually, especially on BrowserStack.
        """
        elements = []
        try:
            # Use page_source XML which is much faster than individual element queries
            xml_source = driver.page_source
            root = ET.fromstring(xml_source)
            
            # Limit to first 50 actionable elements to avoid processing too much (optimized for speed)
            max_elements = 50
            count = 0
            
            def extract_elements(node):
                nonlocal count
                if count >= max_elements:
                    return
                
                attribs = node.attrib
                text = attribs.get('text', '').strip()
                resource_id = attribs.get('resource-id', '').strip()
                content_desc = attribs.get('content-desc', '').strip()
                clickable = attribs.get('clickable', 'false')
                enabled = attribs.get('enabled', 'true')
                displayed = attribs.get('displayed', 'true')
                class_name = attribs.get('class', '').lower()
                
                # Only include elements that are likely clickable/interactive and have meaningful identifiers
                # Skip empty text/content-desc/resource-id to reduce noise
                has_identifier = (text or resource_id or content_desc)
                is_interactive = (clickable == 'true' or 'button' in class_name or 'clickable' in class_name or 
                                'edit' in class_name or 'input' in class_name or 'tab' in class_name)
                is_visible = (enabled == 'true' and displayed == 'true')
                
                if has_identifier and is_visible and (is_interactive or count < 20):  # Always include first 20 elements with identifiers
                    elements.append({
                        "text": text,
                        "resource-id": resource_id,
                        "content-desc": content_desc,
                        "class": attribs.get('class', ''),
                        "clickable": clickable
                    })
                    count += 1
                
                # Recursively process children
                for child in node:
                    extract_elements(child)
            
            extract_elements(root)
            
        except Exception as e:
            print(f"[Warning] Error parsing page_source for elements: {e}")
            # Fallback: return empty list - the LLM can still work with just the step description
            return []
        
        return elements

    def elaborate_and_map_step(self, step, clickable_elements, xml_source: str = ""):
        prompt = f"""
You are an intelligent Android app testing AI for the Microsoft Teams app.

Test step: \"{step}\"

Here is a list of all actionable UI elements on the current screen:
{json.dumps(clickable_elements, indent=2)}

First, analyze the following UI XML hierarchy and determine what type of screen is currently displayed. 
Possible screen types include: chat, meetings, calls, calendar, settings, files, bot_details, app_store, login, etc.

CRITICAL LOCATOR SELECTION RULES:
1. **-*-*******-------Match the step description FIRST**: Find the element that semantically matches what the step is asking for (e.g., if step says "Click on Community Title", find the element that represents the community title, not just any element with a content-desc).

2. **Choose the BEST locator type based on what's available**:
   - If the target element has a **text** attribute that matches the step description, use `locator: "text"` with that text value
   - If the target element has a **resource-id** that is specific and matches, use `locator: "resource-id"` with that id
   - If the target element has a **content-desc** that matches the step description, use `locator: "content-desc"` with that value
   - Only use `locator: "class"` as a last resort if no other identifier matches

3. **DO NOT blindly prefer content-desc**: Many elements don't have content-desc. Only use content-desc if:
   - The element actually has a content-desc attribute in the provided list
   - AND that content-desc semantically matches what the step is asking for

4. **Prefer exact text matches**: If the step mentions specific text (e.g., "Click on Community Title"), prioritize elements whose text attribute contains that exact text, even if they also have content-desc.

5. **Special case for navigation buttons**: When the step indicates proceeding (e.g., "Next", "Continue", "Sign in", "Submit"), first check for content-desc 'vnext' or 'next'. If not found, use text-based matching.

6. **Avoid generic content-desc values**: Do not select an element just because it has a content-desc if that content-desc doesn't match the step (e.g., don't select "Search tab, 1 of 6" when the step asks to click "Community Title").

Respond with a JSON in the following format:
        {{
          "screen_type": "app_store" | "files" | "chat" | ...,
          "screen_description": "Short description of the current screen and its main purpose.",
          "action": {{
            "action": "click" | "type" | "swipe" | "tap" | "wait",
            "locator": "text" | "resource-id" | "content-desc" | "class",
            "value": "value for the locator (must match exactly what's in the element list)",
            "input_text": "only if action is 'type', the input to send",
            "description": "Short description of the step",
            "feature": "main feature being tested (chat/meetings/calls/etc)",
            "expected_behaviour": "Describe what should happen if this step passes (e.g., 'The app should open the meeting interface and allow the user to join the meeting.')"
          }}
        }}

Rules:
        - Only suggest actions that make sense for the detected screen type.
        - After login, if the bottom navigation bar is visible, you should include testing the untested buttons in your test strategy.
        - Do not test bottom navigation buttons before the main app screen is visible.
        - Do not get stuck just testing navigation. Mix it with other feature testing.
        - If the screen is not related to a major feature (e.g., it's a bot details page), suggest actions relevant to that context (e.g., add bot, view info, go back).
        - Do not attempt to test features that are not present on the current screen.
        - If the screen is a details/info page, suggest navigation or context-appropriate actions.

CRITICAL RULES:
            1. Return ONLY valid JSON with the exact format specified
            2. If multiple elements match, choose the most semantically appropriate one that matches the step description
            3. For navigation elements, prefer exact matches over similar ones
            4. If we write same test case's steps twice or thrice in ADO then it should be performed twice or thrice respectively. Don't skip test steps and execute the same step twice or thrice respectively.
            5. If test steps contains "wait for 5 or 10 seconds" then it should be implemented as wait action.
            6. The "value" field must exactly match one of the values from the element list (text, resource-id, or content-desc) - do not make up values.

        Here's the UI XML hierarchy:
        {xml_source}
"""
        response = self.client.chat.completions.create(
            model=os.getenv("AZURE_OPENAI_DEPLOYMENT"),
            messages=[{"role": "user", "content": prompt}],
            temperature=0.5,
        )
        response_text = response.choices[0].message.content
        json_str = self.extract_json_from_response(response_text)
        parsed = try_parse_json_with_fallback(json_str)
        return parsed

    def remap_action_with_alternatives(self, step: str, clickable_elements: list, previous_action: dict) -> dict:
        """
        Ask the LLM for an alternative locator/value for the same step, avoiding the previous locator/value.
        
        Returns a validated action dict with required fields: action, locator, value
        Raises ValueError if remapped action is invalid or missing required fields.
        """
        prev_loc = previous_action.get("locator", "")
        prev_val = previous_action.get("value", "")
        guidance = {
            "step": step,
            "avoid": {"locator": prev_loc, "value": prev_val},
            "ui": clickable_elements,
            "rules": [
                "Prefer content-desc if present.",
                "If step implies Next/Continue/Submit, first try id 'com.microsoft.skype.teams.dev:id/action_next' or content-desc 'next'/'vnext'.",
                "Return only one best alternative mapping.",
                "Response MUST include: action (click/type/tap), locator (id/xpath/content-desc/etc), value (the locator value)",
            ]
        }
        prompt = f"Provide an alternative JSON action mapping for this step, avoiding the previous locator/value.\n{json.dumps(guidance, indent=2)}\nRespond ONLY with a JSON object in the same schema as before. MUST include 'action', 'locator', and 'value' fields."
        response = self.client.chat.completions.create(
            model=os.getenv("AZURE_OPENAI_DEPLOYMENT"),
            messages=[{"role": "user", "content": prompt}],
            temperature=0.2,
        )
        response_text = response.choices[0].message.content
        json_str = self.extract_json_from_response(response_text)
        parsed = try_parse_json_with_fallback(json_str)
        
        # Validate the remapped action
        remapped_action = parsed
        if isinstance(parsed, dict) and "action" in parsed and isinstance(parsed["action"], dict):
            remapped_action = parsed["action"]
        
        # Ensure remapped action has all required fields
        required_fields = ["action", "locator", "value"]
        missing_fields = [field for field in required_fields if field not in remapped_action or remapped_action[field] is None]
        
        if missing_fields:
            raise ValueError(
                f"[Validation] Remapped action is incomplete. Missing fields: {missing_fields}. "
                f"Got: {remapped_action}"
            )
        
        print(f"[Validation] ✓ Remapped action is valid: action={remapped_action.get('action')}, "
              f"locator={remapped_action.get('locator')}, value={remapped_action.get('value')}")
        
        return remapped_action

    def ensure_teams_app_open(self, driver, app_id="com.microsoft.skype.teams.dev", wait_time=2):
        """
        Ensure Teams app is open and in foreground.
        On BrowserStack, the app is already launched via capabilities, so we try launch_app first,
        then fall back to activate_app if needed.
        """
        try:
            print("[AI] Bringing Teams to foreground...")
            # Try launch_app first (works better on BrowserStack for already-installed apps)
            if hasattr(driver, "launch_app"):
                try:
                    driver.launch_app()
                    print("[AI] App launched successfully using launch_app()")
                except Exception as launch_error:
                    print(f"[AI] launch_app() failed: {launch_error}, trying activate_app()...")
                    driver.activate_app(app_id)
                    print("[AI] App activated successfully using activate_app()")
            else:
                # Fallback to activate_app if launch_app is not available
                driver.activate_app(app_id)
                print("[AI] App activated successfully using activate_app()")
            
            print(f"[AI] Waiting {wait_time} seconds for app to load...")
            time.sleep(wait_time)
        except Exception as e:
            # On BrowserStack, if activation fails, the app might already be running
            # Log the error but don't fail - proceed with test execution
            print(f"[AI] Warning: Could not activate app ({e})")
            print("[AI] App may already be running. Proceeding with test steps...")
            time.sleep(1)  # Reduced from 2s - brief wait in case app is loading

    def recover_uiautomator2(self, driver) -> bool:
        """Attempt to recover when UiAutomator2 instrumentation process dies."""
        try:
            print("[Recovery] Restarting UiAutomator2 instrumentation...")
            # Try a benign call to detect session; if it fails, restart session by activating app
            try:
                _ = driver.current_package
            except Exception:
                pass

            # Attempt to re-activate the app which restarts the instrumentation session
            app_id = os.getenv("APP_PACKAGE", "com.microsoft.skype.teams.dev")
            try:
                driver.activate_app(app_id)
                time.sleep(2)
            except Exception as e1:
                print(f"[Recovery] activate_app failed: {e1}")

            # Small wait and a second benign command
            try:
                _ = driver.page_source
            except Exception as e2:
                print(f"[Recovery] page_source after activate failed: {e2}")

            print("[Recovery] UiAutomator2 recovery attempt completed.")
            return True
        except Exception as e:
            print(f"[Recovery] UiAutomator2 recovery failed: {e}")
            return False

    def force_portrait_mode(self, driver):
        try:
            print("[AI] Forcing device orientation to PORTRAIT...")
            driver.orientation = "PORTRAIT"
            time.sleep(1)
        except Exception as e:
            print(f"[AI] Warning: Failed to set portrait orientation: {e}")


    def restart_teams_app_and_navigate_to_chat(self, driver):
        """
        Restart the Teams app and navigate to the chat tab.
        This is used when 'Open the Microsoft Teams App' step comes from ADO.
        """
        try:
            print("[AI] Restarting Teams app and navigating to chat tab...")
            
            # Always target Microsoft Teams (ignore APP_PACKAGE overrides for this step)
            app_package = "com.microsoft.skype.teams.dev"
            app_activity = "com.microsoft.skype.teams.Launcher"

            # Gracefully terminate the app without clearing data
            try:
                if hasattr(driver, "terminate_app"):
                    print(f"[AI] Terminating {app_package} (no data clear)...")
                    driver.terminate_app(app_package)
                    time.sleep(2)
            except Exception as term_err:
                print(f"[AI] Warning: terminate_app failed: {term_err}")

            # Try to bring the app back using start_activity first, then fallback to activate_app
            launched = False
            try:
                if hasattr(driver, "start_activity"):
                    print(f"[AI] Starting activity {app_package}/{app_activity}...")
                    driver.start_activity(app_package, app_activity)
                    launched = True
            except Exception as sa_err:
                print(f"[AI] Warning: start_activity failed: {sa_err}")
            
            if not launched:
                try:
                    print("[AI] Launching Teams app via activate_app...")
                    driver.activate_app(app_package)
                    launched = True
                except Exception as act_err:
                    print(f"[AI] Error: activate_app failed: {act_err}")
                    # Final fallback using ADB am start (no data clear)
                    try:
                        adb = "adb"
                        device_name = os.getenv("ANDROID_DEVICE_NAME", "")
                        target = ["-s", device_name] if device_name else []
                        print("[AI] Fallback: adb am start ...")
                        subprocess.run([adb] + target + [
                            "shell", "am", "start", "-n", f"{app_package}/{app_activity}"
                        ], stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
                        launched = True
                    except Exception as adb_err:
                        print(f"[AI] Fallback adb start failed: {adb_err}")

            # Give the app a moment to render
            time.sleep(6)
            
            # 🔒 FORCE PORTRAIT MODE HERE
            self.force_portrait_mode(driver)

            time.sleep(6)

            # Dismiss notification popup if present (Got it)
            self.dismiss_got_it_popup(driver)

            # Navigate to chat tab
            print("[AI] Navigating to chat tab...")
            self.navigate_to_chat_tab(driver)
            
            return True
            
        except Exception as e:
            print(f"[AI] Error restarting app and navigating to chat: {e}")
            return False

    def reset_between_testcases(self, driver):
        """
        For a fresh start between test cases:
        - Remove Teams from Recents
        - Force stop the app
        - Clear app data/cache (no uninstall)
        - Start launcher activity
        - Navigate to Chat tab
        """
        try:
            app_package = "com.microsoft.skype.teams.dev"
            app_activity = "com.microsoft.skype.teams.Launcher"

            # Dismiss recents and remove the app if present
            try:
                adb = "adb"
                device_name = os.getenv("ANDROID_DEVICE_NAME", "")
                target = ["-s", device_name] if device_name else []
                # Open recents
                subprocess.run([adb] + target + ["shell", "input", "keyevent", "KEYCODE_APP_SWITCH"], stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
                time.sleep(1)
                # Try to swipe away the topmost recent app
                # Generic swipe gesture (x1,y1) -> (x2,y2)
                subprocess.run([adb] + target + ["shell", "input", "swipe", "500", "1200", "500", "200"], stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
                time.sleep(1)
            except Exception as e:
                print(f"[AI] Warning: could not manipulate Recents: {e}")

            # Force stop
            try:
                if hasattr(driver, "terminate_app"):
                    driver.terminate_app(app_package)
                else:
                    adb = "adb"
                    device_name = os.getenv("ANDROID_DEVICE_NAME", "")
                    target = ["-s", device_name] if device_name else []
                    subprocess.run([adb] + target + ["shell", "am", "force-stop", app_package], stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
            except Exception as e:
                print(f"[AI] Warning: terminate_app failed: {e}")

            # Clear data/cache
            try:
                adb = "adb"
                device_name = os.getenv("ANDROID_DEVICE_NAME", "")
                target = ["-s", device_name] if device_name else []
                subprocess.run([adb] + target + ["shell", "pm", "clear", app_package], stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
            except Exception as e:
                print(f"[AI] Warning: pm clear failed: {e}")

            # Start activity
            launched = False
            try:
                if hasattr(driver, "start_activity"):
                    driver.start_activity(app_package, app_activity)
                    launched = True
            except Exception as e:
                print(f"[AI] Warning: start_activity failed: {e}")
            if not launched:
                try:
                    driver.activate_app(app_package)
                    launched = True
                except Exception as e:
                    print(f"[AI] Warning: activate_app failed: {e}")
            time.sleep(6)

            # Dismiss notification popup if present (Got it)
            self.dismiss_got_it_popup(driver)

            # Go to chat
            self.navigate_to_chat_tab(driver)
            return True
        except Exception as e:
            print(f"[AI] Error in reset_between_testcases: {e}")
            return False

    def dismiss_got_it_popup(self, driver, timeout_sec: int = 6) -> bool:
        """
        Dismisses a notification/coachmark popup by tapping the 'Got it' button if visible.
        Returns True if dismissed, False if not found.
        """
        try:
            end_time = time.time() + timeout_sec
            while time.time() < end_time:
                try:
                    # Try common selectors for 'Got it'
                    for locator in [
                        ("xpath", "//*[contains(@text, 'Got it') or contains(@text, 'GOT IT') or contains(@text, 'Got It')]") ,
                        ("xpath", "//*[contains(@content-desc, 'Got it') or contains(@content-desc, 'GOT IT')]") ,
                        ("xpath", "//android.widget.Button[contains(@text, 'Got') or contains(@text, 'GOT')]"),
                        ("accessibility id", "Got it"),
                    ]:
                        try:
                            el = driver.find_element(*locator)
                            if el and el.is_enabled():
                                print("[AI] Dismissing popup: tapping 'Got it'")
                                el.click()
                                time.sleep(1)
                                return True
                        except Exception:
                            continue
                except Exception:
                    pass
                time.sleep(0.5)
        except Exception as e:
            print(f"[AI] Warning: dismiss_got_it_popup encountered an error: {e}")
        return False

    def force_restart_teams_app(self, device_name, app_package):
        """
        Deprecated: We no longer clear app data for the 'Open the Microsoft Teams App' step.
        Kept for backward compatibility but now only force-stops the app.
        """
        try:
            adb = "adb"
            target = ["-s", device_name] if device_name else []

            # Force-stop app if running (do not clear data)
            print(f"[AI] Force stopping {app_package} (no data clear)...")
            subprocess.run([adb] + target + ["shell", "am", "force-stop", app_package],
                           stdout=subprocess.PIPE, stderr=subprocess.STDOUT)

            print("[AI] App force-stop completed")
        except Exception as e:
            print(f"[AI] Error during app force-stop: {e}")
            raise

    def navigate_to_chat_tab(self, driver):
        """
        Navigate to the chat tab in the Teams app.
        """
        try:
            # Try to find and click the Chat tab
            chat_tab_selectors = [
                "//*[contains(@content-desc, 'Chat tab')]",
                "//*[contains(@text, 'Chat')]",
                "//android.widget.Button[contains(@text, 'Chat')]",
                "//*[@content-desc='Chat']"
            ]
            
            for selector in chat_tab_selectors:
                try:
                    chat_tab = driver.find_element(By.XPATH, selector)
                    if chat_tab.is_enabled():
                        print("[AI] Clicking Chat tab...")
                        chat_tab.click()
                        time.sleep(3)
                        
                        # Verify we're on chat screen
                        if self.agent.is_on_chat_screen(driver):
                            print("[AI] Successfully navigated to chat tab")
                            return True
                        else:
                            print("[AI] Chat tab clicked but not on chat screen yet")
                            time.sleep(2)
                            continue
                except NoSuchElementException:
                    continue
            
            # If direct chat tab click didn't work, try alternative navigation
            print("[AI] Trying alternative navigation to chat...")
            return self.try_alternative_chat_navigation(driver)
            
        except Exception as e:
            print(f"[AI] Error navigating to chat tab: {e}")
            return False

    def try_alternative_chat_navigation(self, driver):
        """
        Try alternative methods to navigate to chat tab.
        """
        try:
            # Try clicking on bottom navigation elements
            bottom_nav_selectors = [
                "//*[contains(@content-desc, 'Chat')]",
                "//*[contains(@text, 'Chat')]",
                "//android.widget.Button[contains(@text, 'Chat')]"
            ]
            
            for selector in bottom_nav_selectors:
                try:
                    element = driver.find_element(By.XPATH, selector)
                    if element.is_enabled():
                        print(f"[AI] Trying alternative chat navigation with selector: {selector}")
                        element.click()
                        time.sleep(3)
                        
                        if self.agent.is_on_chat_screen(driver):
                            print("[AI] Alternative navigation to chat successful")
                            return True
                except NoSuchElementException:
                    continue
            
            print("[AI] Could not navigate to chat tab using alternative methods")
            return False
            
        except Exception as e:
            print(f"[AI] Error in alternative chat navigation: {e}")
            return False

    def run(self):
        # Ensure app is foregrounded and user is logged in before executing ADO steps
        try:
            self.ensure_teams_app_open(self.driver)
            self.agent.ensure_logged_in(self.driver)
            # Wait until chat is visible or timeout
            start_wait = time.time()
            while not self.agent.is_on_chat_screen(self.driver) and time.time() - start_wait < 10:
                time.sleep(1)
            if not self.agent.is_on_chat_screen(self.driver):
                print("[Warning] Chat screen not detected after waiting. Proceeding anyway.")
        except Exception as e:
            print(f"[Info] Skipping pre-check login due to error: {e}")

        steps = self.test_case.get("steps", [])
        if not isinstance(steps, list):
            steps = []
        expected_result = self.test_case.get("expected_result", "")
        
        # Debug: Print steps information
        print(f"\n[DEBUG] Test Case: {self.test_case.get('title', 'Unknown')}")
        print(f"[DEBUG] Number of steps found: {len(steps)}")
        if steps:
            print(f"[DEBUG] Steps: {steps}")
        else:
            print("[DEBUG] WARNING: No steps found in test case!")
            print("[DEBUG] Test case data:", self.test_case)
            return

        all_passed = True
        print(f"\n[INFO] Starting execution of {len(steps)} step(s)...")
        for idx, step in enumerate(steps, 1):
            try:
                # Special handling for 'Open the Microsoft Teams App' step
                if step.strip().lower() == "open the microsoft teams app":
                    print(f"[Step {idx}] Restarting Teams app and navigating to chat tab...")
                    restart_success = self.restart_teams_app_and_navigate_to_chat(self.driver)
                    
                    if restart_success:
                        result = "passed"
                        print(f"[Step {idx}] ✅ Teams app restarted and navigated to chat tab successfully")
                    else:
                        result = "failed"
                        print(f"[Step {idx}] ❌ Failed to restart Teams app or navigate to chat tab")
                    
                    screenshot_path = self.reporter.capture_screenshot(self.driver, step=idx)
                    # Log with EXACT ADO step description - no modifications
                    self.reporter.log_step(idx, {"description": step}, result, screenshot_path)
                    time.sleep(1)
                    continue

                # AI-based verification for 'verify' steps
                if "verify" in step.strip().lower():
                    xml_source = self.driver.page_source or ""
                    summary = self.summarize_ui_xml(xml_source)
                    ui_text = json.dumps(summary, indent=2)

                    def _extract_expected_texts(s: str):
                        texts = []
                        try:
                            quotes = re.findall(r'"([^"]+)"', s)
                            for q in quotes:
                                qt = q.strip()
                                if qt:
                                    texts.append(qt)
                            if texts:
                                return texts
                            m = re.search(r'verify\s+(.*?)\s+text\s+on\s+screen', s, flags=re.IGNORECASE)
                            if m:
                                cand = m.group(1).strip()
                                if cand:
                                    texts.append(cand)
                        except Exception:
                            pass
                        return texts

                    no_expected = (not expected_result) or expected_result.strip() == "" or "No expected result" in expected_result
                    if no_expected:
                        expected_texts = _extract_expected_texts(step)
                        haystack = (ui_text + "\n" + xml_source).lower()
                        if expected_texts:
                            missing = [t for t in expected_texts if t.lower() not in haystack]
                            if not missing:
                                ai_result = f"PASS - Found expected text(s): {', '.join(expected_texts)}"
                                print(f"[AI Verification] {ai_result}")
                                screenshot_path = self.reporter.capture_screenshot(self.driver, step=idx)
                                self.reporter.log_step(idx, {"description": step, "ai_verification": ai_result}, "passed", screenshot_path)
                                time.sleep(1)
                                continue
                            else:
                                ai_result = f"FAIL - Missing text: {', '.join(missing)}"
                                print(f"[AI Verification] {ai_result}")
                                screenshot_path = self.reporter.capture_screenshot(self.driver, step=idx)
                                self.reporter.log_step(idx, {"description": step, "ai_verification": ai_result}, "failed", screenshot_path)
                                all_passed = False
                                time.sleep(1)
                                continue

                    prompt = f"""
Here is the extracted UI content:
{ui_text}

Here is the expected result:
{expected_result}

Does the UI content match the expected result? Reply with PASS or FAIL and a brief explanation.
"""
                    response = self.client.chat.completions.create(
                        model=os.getenv("AZURE_OPENAI_DEPLOYMENT"),
                        messages=[{"role": "user", "content": prompt}],
                        temperature=0.3,
                    )
                    ai_result = response.choices[0].message.content.strip()
                    print(f"[AI Verification] {ai_result}")
                    screenshot_path = self.reporter.capture_screenshot(self.driver, step=idx)

                    result = "failed"
                    if ai_result.strip().upper().startswith("PASS"):
                        result = "passed"
                        screenshot_path = self.reporter.capture_screenshot(self.driver, step=idx)
                    self.reporter.log_step(idx, {"description": step, "ai_verification": ai_result}, result, screenshot_path)
                    time.sleep(1)
                    if result == "failed":
                        all_passed = False
                    continue

                # print(f"\n[Step {idx}] {step}")
                
                # Get clickable elements using fast XML parsing (much faster than scanning all elements)
                # This is optimized for BrowserStack where individual element queries are slow
                xml_source = self.driver.page_source or ""
                try:
                    clickable_elements = self.get_clickable_elements(self.driver)
                    if not clickable_elements:
                        print("[Info] No clickable elements found via XML parsing, using page_source summary instead")
                        summary = self.summarize_ui_xml(xml_source, max_elements=50)
                        clickable_elements = summary if isinstance(summary, list) else []
                except Exception as e:
                    print(f"[Warning] Error getting clickable elements: {e}, proceeding with empty list")
                    clickable_elements = []
                
                try:
                    action_dict = self.elaborate_and_map_step(step, clickable_elements, xml_source)
                except Exception as e:
                    error_msg = f"Failed to elaborate step: {str(e)}"
                    print(f"[!] {error_msg}")
                    # Log with EXACT ADO step description, pass reason separately
                    self.reporter.log_step(idx, {"description": step}, "failed", screenshot_path, reason=error_msg)
                    all_passed = False
                    continue

                if not action_dict:
                    print(f"[!] No action returned for step {idx}")
                    # Log with EXACT ADO step description, pass reason separately
                    self.reporter.log_step(idx, {"description": step}, "failed", screenshot_path, reason="No action generated for step")
                    all_passed = False
                    continue

                print(f"\n[Step {idx}] {step}")

                # Deterministic mapping for closing the Android share panel via coordinate tap
                normalized_step = (step or "").strip().lower()
                if normalized_step in [
                    "close the sharing panel",
                    "close sharing panel",
                    "close share panel",
                    "dismiss share sheet",
                    "close share sheet",
                ]:
                    action_dict = {
                        "elaboration": "Dismiss the Android share sheet by tapping outside the panel.",
                        "action": {
                            "action": "tap",
                            "x": 613,
                            "y": 955,
                            "description": "Close the sharing panel",
                            "expected_behaviour": "The sharing panel should close and return to the previous screen."
                        }
                    }
                else:
                    action_dict = self.elaborate_and_map_step(step, clickable_elements, xml_source)

                if "action" in action_dict:
                    action = action_dict["action"]
                    if "elaboration" in action_dict:
                        action["elaboration"] = action_dict["elaboration"]
                else:
                    action = action_dict
                print(f"[Action] {action}")

                try:
                    self.agent.perform_action(self.driver, action)
                    result = "passed"
                    screenshot_path = self.reporter.capture_screenshot(self.driver, step=idx)
                except Exception as e:
                    error_msg = str(e)
                    print(f"[!] Failed to perform action at step {idx}: {error_msg}")
                    
                    # One-time remap and retry with alternative locator/value
                    try:
                        print(f"[Retry] Attempting to remap action with alternative locator for step {idx}...")
                        clickable_elements = self.get_clickable_elements(self.driver)
                        alt_action = self.remap_action_with_alternatives(step, clickable_elements, previous_action=action)
                        
                        # Validate remapped action has all required fields BEFORE executing
                        if not alt_action or not isinstance(alt_action, dict):
                            raise ValueError(f"[Validation] Remapped action is not a dictionary: {alt_action}")
                        
                        required_fields = ["action", "locator", "value"]
                        missing = [f for f in required_fields if f not in alt_action or alt_action[f] is None]
                        if missing:
                            raise ValueError(
                                f"[Validation] Remapped action missing required fields: {missing}. "
                                f"Got: {alt_action}. Will NOT execute incomplete action."
                            )
                        
                        print(f"[Retry] Alternative action: {alt_action}")
                        self.agent.perform_action(self.driver, alt_action)
                        result = "passed"
                        action = alt_action
                        screenshot_path = self.reporter.capture_screenshot(self.driver, step=idx)
                        print(f"[Retry] ✅ Step {idx} succeeded with alternative locator")
                    except Exception as e2:
                        error_msg2 = str(e2)
                        print(f"[Retry] ❌ Alternative attempt failed at step {idx}: {error_msg2}")
                        result = "failed"
                        all_passed = False
                        screenshot_path = self.reporter.capture_screenshot(self.driver, step=idx)

                # Log with EXACT ADO step text, not the elaborated action
                self.reporter.log_step(idx, {"description": step}, result, screenshot_path)
                time.sleep(1)
            except Exception as e:
                error_msg = str(e)
                print(f"[!] Error in test step {idx}: {error_msg}")
                # Log with EXACT ADO step description, pass reason separately
                self.reporter.log_step(idx, {"description": step}, "failed", None, reason=error_msg)
                all_passed = False

        final_result = "passed" if all_passed else "failed"
        # DO NOT log expected result as a step - only ADO steps should appear in report
        print("\n[INFO] Test case execution complete. Report generated.")
