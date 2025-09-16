
***

### **Project Scope: YouTube Data Scraper & Processor**

#### **Project Summary**
A Python-based tool to download all video transcripts and comments from specified YouTube channels. It will use proxy services to avoid IP bans, process the collected data with the Gemini API, and store the refined information in an SQLite database for easy querying.

#### **Core Features / User Stories**
*   The system downloads all video transcripts for a given YouTube channel.
*   The system downloads all video comments for a given YouTube channel.
*   The system periodically checks all previously processed channels for new videos and downloads their associated data.
*   The system uses the Gemini API to select and refine the raw transcript and comment data.
*   The system provides a powerful, flexible suite of querying options for manually digesting/analyzing the final data in the database.

#### **Technical Stack**
*   **Programming Language:** Python
*   **Key Libraries:** `youtube_transcript_api`, Google API Client (for YouTube Data API v3), `asyncio`, `python-dotenv`, Google Generative AI SDK (for Gemini), `sqlite3`.
*   **Services:** Webshare (for proxies), YouTube Data API v3, Gemini API.
*   **Database:** SQLite.

#### **Implementation Details / Logic**
*   **Credentials:** API keys and Webshare credentials will be loaded from a `.env` file.
*   **Networking:** For downloading transcripts via youtube_transcript_api, asynchronous programming will be used, and all requests will be routed through Webshare proxies to prevent IP bans.
*   **Channel Updates:** A mechanism is needed to efficiently find new videos from previously processed channels.
    *   ---------#1.)----[For each channel, store the `published_date` of the last video fetched. When checking for updates, only request videos published after that date to minimize API quota usage.]------------
*   **Data Integrity:** The system must be resilient to crashes and prevent data corruption during execution.
    *   ---------#2.)----[Implement a state-tracking mechanism. For example, log the download/processing status of each video (e.g., 'pending', 'completed', 'failed') in the database. On startup, the script can automatically resume work on any videos not marked 'completed'.]------------

#### **Data Model / Schema**
*   **Initial Data Storage:** The method for storing raw, pre-processed data is TBD.
    *   ---------#3.)----[Store raw transcripts and comments as structured JSON or YAML files first. This separates the download and processing stages, allowing you to re-run the Gemini processing step on existing data without needing to re-download it.]------------
*   **Database:** Processed data will be stored in an SQLite database.
    *   ---------#4.)----[Establish a clear schema with related tables, such as `Channels`, `Videos`, `ProcessedTranscripts`, and `ProcessedComments`.]------------

#### **Testing**
*   ---------#5.)----[Instead of formal unit testing, begin by creating a few simple test scripts for core workflows (e.g., 'download and process a single video'). This will help verify functionality without the complexity of a full testing framework.]------------

***

Do any of these proposals fit the scope?