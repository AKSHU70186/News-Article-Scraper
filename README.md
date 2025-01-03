import requests  # For sending HTTP requests to the website
from bs4 import BeautifulSoup  # For parsing HTML content
from selenium import webdriver  # For controlling the web browser
from selenium.webdriver.chrome.service import Service  # To configure the chromedriver service
from selenium.webdriver.chrome.options import Options  # To set Chrome options
import pandas as pd  # For data manipulation and analysis
import gspread  # For interacting with Google Sheets
from oauth2client.service_account import ServiceAccountCredentials  # For Google Sheets API authentication
import time  # For adding delays

# Configure Selenium with headless Chrome and user agent
chrome_options = Options()
chrome_options.add_argument("--headless")  # Run Chrome in headless mode (without a GUI)
chrome_options.add_argument("--no-sandbox")  # Bypass OS security model
chrome_options.add_argument("--disable-dev-shm-usage")  # Overcome limited resource problems
user_agent = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/58.0.3029.110 Safari/537.3"
chrome_options.add_argument(f"user-agent={user_agent}")  # Set user agent string

# Update the path to chromedriver as needed
service = Service('/path/to/chromedriver')  # Path to your chromedriver
driver = webdriver.Chrome(service=service, options=chrome_options)  # Initialize Chrome webdriver with the specified options

def scrape_news_articles(url):
    """
    Scrape news articles from the given URL.
    
    Args:
        url (str): The URL of the website to scrape.
    
    Returns:
        list: A list of dictionaries containing news articles' titles, dates, and content.
    """
    try:
        headers = {"User-Agent": user_agent}  # Set headers to mimic a real browser
        response = requests.get(url, headers=headers)  # Send a GET request to the URL
        response.raise_for_status()  # Check if the request was successful

        soup = BeautifulSoup(response.text, 'html.parser')  # Parse the HTML content

        articles = soup.find_all('article')  # Find all article elements
        news_data = []

        for article in articles:  # Loop through each article
            title = article.find('h1') or article.find('h2') or article.find('h3')  # Find the title of the article
            title = title.text.strip() if title else 'N/A'  # Get the text content of the title

            date = article.find('time')  # Find the publication date of the article
            date = date.text.strip() if date else 'N/A'  # Get the text content of the date

            content = article.find('p')  # Find the main content of the article
            content = content.text.strip() if content else 'N/A'  # Get the text content of the article

            news_data.append({'Title': title, 'Date': date, 'Content': content})  # Append the article data to the list

        return news_data

    except requests.RequestException as e:  # Handle any request exceptions
        print(f"Error fetching the website: {e}")
        return None

def store_in_google_sheets(data, spreadsheet_id, json_cred_path):
    """
    Store the scraped data in Google Sheets.
    
    Args:
        data (list): The data to store.
        spreadsheet_id (str): The ID of the Google Sheet.
        json_cred_path (str): The path to the JSON credentials file.
    """
    try:
        scope = ["https://spreadsheets.google.com/feeds", 'https://www.googleapis.com/auth/spreadsheets',
                 "https://www.googleapis.com/auth/drive.file", "https://www.googleapis.com/auth/drive"]
        creds = ServiceAccountCredentials.from_json_keyfile_name(json_cred_path, scope)  # Authenticate using the JSON credentials file
        client = gspread.authorize(creds)  # Authorize the client
        sheet = client.open_by_key(spreadsheet_id).sheet1  # Open the specified Google Sheet

        # Insert data into Google Sheets
        sheet.clear()  # Clear the existing content
        sheet.insert_row(["Title", "Date", "Content"], 1)  # Insert headers
        for article in data:  # Loop through each article
            sheet.append_row([article['Title'], article['Date'], article['Content']])  # Append the article data to the sheet

    except Exception as e:  # Handle any exceptions
        print(f"Error storing data in Google Sheets: {e}")

# Example usage
url = 'https://news.ycombinator.com/'  # The URL to scrape
spreadsheet_id = 'your_google_sheet_id'  # The ID of your Google Sheet
json_cred_path = 'path/to/credentials.json'  # The path to your JSON credentials file
data = scrape_news_articles(url)  # Scrape news articles
if data:
    store_in_google_sheets(data, spreadsheet_id, json_cred_path)  # Store the data in Google Sheets
    print("Data successfully stored in Google Sheets")
else:
    print("Failed to scrape the website")
