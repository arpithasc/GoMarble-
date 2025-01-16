# GoMarble-

from flask import Flask, request, jsonify
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.service import Service
import openai
import time

# Initialize Flask app
app = Flask(__name__)

# OpenAI API key
openai.api_key = "YOUR_OPENAI_API_KEY"  # Replace with your OpenAI API key

# Setup Selenium WebDriver
def get_driver():
    service = Service("/path/to/chromedriver")  # Replace with your ChromeDriver path
    options = webdriver.ChromeOptions()
    options.add_argument("--headless")  # Headless mode for no UI
    return webdriver.Chrome(service=service, options=options)

# Get dynamic CSS selectors using OpenAI API
def get_css_selector(page_content):
    prompt = f"Identify the CSS selectors for comment text, user name, and post date from the following HTML content:\n\n{page_content}"
    response = openai.Completion.create(
        engine="text-davinci-003",
        prompt=prompt,
        max_tokens=100
    )
    return response.choices[0].text.strip()

# Extract comments from the page
def extract_comments(url):
    driver = get_driver()
    driver.get(url)
    time.sleep(3)  # Wait for page to load

    page_content = driver.page_source
    css_selectors = get_css_selector(page_content)

    selectors = css_selectors.splitlines()
    comment_selector, author_selector, date_selector = [s.split(":")[1].strip() for s in selectors]

    comments = []
    while True:
        try:
            comments_elements = driver.find_elements(By.CSS_SELECTOR, comment_selector)
            authors_elements = driver.find_elements(By.CSS_SELECTOR, author_selector)
            dates_elements = driver.find_elements(By.CSS_SELECTOR, date_selector)

            for i in range(len(comments_elements)):
                comments.append({
                    "author": authors_elements[i].text,
                    "date": dates_elements[i].text,
                    "comment": comments_elements[i].text
                })

            # Pagination: check for next page button
            next_button = driver.find_element(By.CSS_SELECTOR, "button.next-page")
            if next_button.is_enabled():
                next_button.click()
                time.sleep(3)
            else:
                break
        except Exception as e:
            break

    driver.quit()
    return {"comments_count": len(comments), "comments": comments}

# API endpoint to fetch comments
@app.route("/api/comments", methods=["GET"])
def comments_api():
    page_url = request.args.get("page")
    if not page_url:
        return jsonify({"error": "Missing 'page' parameter"}), 400

    try:
        data = extract_comments(page_url)
        return jsonify(data)
    except Exception as e:
        return jsonify({"error": str(e)}), 500

if __name__ == "__main__":
    app.run(debug=True)