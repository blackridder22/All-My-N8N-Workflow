# n8n Workflow Explanation: Website Content Scraper

##  workflow Overview

This workflow automates the process of scraping all pages from a website's sitemap, extracting the main content of each page, and saving it as a structured Markdown file on the local disk. It's an end-to-end pipeline for content ingestion.

---

## Step-by-Step Breakdown

### 1. `When clicking ‘Execute workflow’ (Manual Trigger)`

*   **Action:** This is the entry point. The workflow starts when a user manually clicks the "Execute Workflow" button.
*   **Purpose:** To initiate the entire scraping process on demand.

### 2. `HTTP Request1`

*   **Action:** Sends an HTTP `GET` request to the specified URL.
*   **URL:** `https://docmost.com/docs/sitemap.xml`
*   **Purpose:** To fetch the website's sitemap, which contains a list of all discoverable URLs.

### 3. `XML`

*   **Action:** Parses the XML data received from the previous `HTTP Request1` node.
*   **Purpose:** To convert the raw XML sitemap into a structured JSON format, making it easy to access the list of URLs.

### 4. `Split Out`

*   **Action:** Takes the array of URLs from the parsed sitemap and splits it into individual items.
*   **Field to Split:** `urlset.url`
*   **Purpose:** To process each URL one by one in the subsequent steps, rather than handling them all at once.

### 5. `Add Right name` (Set Node)

*   **Action:** Creates two new fields, `Lien` and `Title`, for each item. Both are populated with the URL from the sitemap.
*   **Value:** `{{ $json.loc }}`
*   **Purpose:** To prepare and rename the data for clarity before it enters the main processing loop.

### 6. `Loop Over Items`

*   **Action:** Iterates through each item received from the `Add Right name` node. It is configured to run in batches of 1, effectively processing one URL at a time.
*   **Purpose:** To execute the core scraping and saving logic for every single URL found in the sitemap.

---

### Inside the Loop (Executed for Each URL)

The following nodes run once for every item processed by the `Loop Over Items` node.

#### 7. `HTTP Request`

*   **Action:** Sends an HTTP `POST` request to a local crawling service.
*   **URL:** `http://host.docker.internal:11235/crawl`
*   **Body:** It sends the URL to be scraped (`{{ $json.Lien }}`) along with a configuration for the crawler.
    ```json
    {
      "urls": ["{{ $json.Lien }}"],
      "crawler_config": {
        "type": "CrawlerRunConfig",
        "params": {
          "scraping_strategy": { "type": "WebScrapingStrategy" },
          "stream": true
        }
      }
    }
    ```
*   **Purpose:** To delegate the complex task of web scraping to a specialized local service, which extracts the page's content.

#### 8. `Synthetize ALL` (Set Node)

*   **Action:** Processes the response from the crawler and sets two new fields.
    1.  **`Markdown`**: Extracts the raw markdown content from the crawler's result.
        *   **Value:** `{{ $json.results[0].markdown.raw_markdown }}`
    2.  **`Title`**: Generates a clean, URL-safe filename from the original URL using a JavaScript expression. For example, `https://docmost.com/docs/getting-started` becomes `docmost-getting-started`.
*   **Purpose:** To structure the scraped data and create a standardized filename for the output file.

#### 9. `Convert to File`

*   **Action:** Converts the text content of the `Markdown` field into a binary file format.
*   **Purpose:** A necessary technical step to prepare the data to be written to the filesystem.

#### 10. `Read/Write Files from Disk`

*   **Action:** Writes the binary file data from the previous step to the disk.
*   **File Name:** Uses the `Title` field to construct a dynamic file path.
    *   **Path:** `=/external-files/docmost/{{ $('Synthetize ALL').item.json.Title }}.md`
*   **Purpose:** To save the scraped content as a Markdown file in a designated directory. The loop then continues with the next URL until all are processed.

---

## Use Cases & Requirements

### What is it for?

*   **Knowledge Base Creation:** Ingesting website content into a vector database to power a Retrieval-Augmented Generation (RAG) system or a custom AI assistant.
*   **Content Archiving:** Creating a complete, offline backup of a documentation site or blog.
*   **Data Migration:** Scraping content from one platform to be transformed and imported into another.

### To use this workflow, you need:

1.  A running **n8n instance**.
2.  A **local web crawling service** accessible at `http://host.docker.internal:11235`. This is a custom dependency and not a standard n8n feature.
3.  If using Docker, a **mounted volume** that maps a local directory to `/external-files/` inside the n8n container, with write permissions.
