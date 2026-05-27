# puthon-web-scrapers
Production-ready Python scripts for data extraction and automation.
import logging
import random
import time
from concurrent.futures import ThreadPoolExecutor
from bs4 import BeautifulSoup
import pandas as pd
import requests

logging.basicConfig(
    level=logging.INFO, format="%(asctime)s - [%(levelname)s] - %(message)s"
)

BASE_URL = "https://quotes.toscrape.com"
HEADERS = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
}


def get_page_data(page_number):
    """Fetch and parse data from a single page."""
    url = f"{BASE_URL}/page/{page_number}/"
    items = []

    try:
        # Anti-bot detection: random delay between requests
        time.sleep(random.uniform(0.5, 1.5))

        response = requests.get(url, headers=HEADERS, timeout=10)

        if response.status_code == 404:
            logging.warning(f"Page {page_number} not found (404).")
            return items

        response.raise_for_status()
        soup = BeautifulSoup(response.text, "html.parser")
        cards = soup.find_all("div", class_="quote")

        for card in cards:
            text = (
                card.find("span", class_="text").text.strip().replace("“", "").replace("”", "")
            )
            author = card.find("small", class_="author").text.strip()
            tags = [tag.text for tag in card.find_all("a", class_="tag")]

            items.append(
                {
                    "Page": page_number,
                    "Text Data": text,
                    "Author/Brand": author,
                    "Tags/Category": ", ".join(tags),
                }
            )

        logging.info(f"Successfully parsed page {page_number}. Items found: {len(cards)}")

    except requests.exceptions.RequestException as e:
        logging.error(f"Network error on page {page_number}: {e}")
    except Exception as e:
        logging.error(f"Unexpected error on page {page_number}: {e}")

    return items


def main():
    logging.info("Starting target scraping project...")
    pages_to_scrape = range(1, 6)
    all_results = []

    with ThreadPoolExecutor(max_workers=3) as executor:
        futures = executor.map(get_page_data, pages_to_scrape)

        for result in futures:
            if result:
                all_results.extend(result)
    
    if all_results:
        df = pd.DataFrame(all_results)
        output_file = "scraped_data_result.xlsx"

        with pd.ExcelWriter(output_file, engine="openpyxl") as writer:
            df.to_excel(writer, index=False, sheet_name="Data")

        logging.info(f"Job done! Data saved to: {output_file}")
    else:
        logging.error("No data collected.")


if __name__ == "__main__":
    start_time = time.time()
    main()
    logging.info(f"Execution time: {round(time.time() - start_time, 2)} sec.")
