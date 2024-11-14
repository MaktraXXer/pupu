ХУЙХУХЙХУЙУХЙУХЙУХУХ

from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.chrome.options import Options
from webdriver_manager.chrome import ChromeDriverManager

def setup_selenium_proxy():
    chrome_options = Options()
    chrome_options.add_argument("--headless")  # Запускаем в фоновом режиме
    chrome_options.add_argument("--disable-gpu")
    chrome_options.add_argument("--no-sandbox")
    chrome_options.add_argument("--disable-dev-shm-usage")
    chrome_options.add_argument("--window-size=1920x1080")

    # Задаем параметры корпоративного прокси
    proxy = "http://mi.makhmudov:mR3%5Bypulczaf@proxy-perovo.roscap.com:3128"
    chrome_options.add_argument(f'--proxy-server={proxy}')

    service = Service(ChromeDriverManager().install())
    driver = webdriver.Chrome(service=service, options=chrome_options)
    return driver


def is_product_active(url):
    """
    Checks if a product URL is active on the website by detecting redirection patterns.
    """
    driver = setup_selenium_proxy()  # Используем прокси для Selenium

    is_active = True
    try:
        print(f"Checking product URL: {url}")
        driver.get(url)
        current_url = driver.current_url
        print(f"Initial URL loaded: {current_url}")

        # Ожидание и обработка редиректов
        max_wait_time = 10
        WebDriverWait(driver, max_wait_time).until(EC.url_changes(current_url))

        max_redirects = 10
        redirect_count = 0
        while redirect_count < max_redirects:
            time.sleep(2)
            new_url = driver.current_url

            if "ProductIsNotActive" in new_url or "ProductsNotActive" in driver.page_source:
                print("Detected inactive product message.")
                is_active = False
                break

            if new_url == current_url:
                break

            current_url = new_url
            redirect_count += 1
            print(f"Redirect {redirect_count}: {current_url}")

        print(f"Final URL after redirects: {current_url}")

    except Exception as e:
        print(f"An error occurred during checking: {e}")
        is_active = False
    finally:
        driver.quit()

    return is_active
