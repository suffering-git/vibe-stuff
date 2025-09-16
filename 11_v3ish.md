### **Project Scope: YouTube Data Scraper & Processor v2.2 (Technical Specification)**

#### **Project Summary**
A Python-based tool to download video transcripts and comments from YouTube channels for market research. The goal is to inform a market garden and orchard business plan focused on regenerative agriculture. The system performs a multi-stage AI processing pipeline on the raw data, structuring it into a normalized database of topic-based summaries and atomic insights. A hybrid search system combines full-text search with AI-powered semantic search via vector embeddings for comprehensive data discovery.

#### **Core Features / User Stories**
*   **Downloads Raw Data:** Downloads all video transcripts and comments for specified channels, saving each directly in the database.
*   **Automated Translation:** Automatically translates all non-English transcripts to English during the initial download phase.
*   **Resumable Processing:** A robust status tracking system ensures that if the script is stopped or crashes at any point, it can be resumed without re-doing completed work.
*   **Structured AI Analysis:** A logical 3-stage AI pipeline extracts and summarizes raw text into topic-based summaries, then refines and atomizes this content into two distinct data types: `quantitative` (numerical insights) and `qualitative` (descriptive insights), then generates embeddings for semantic search. All stages are fully decoupled for maximum flexibility.
*   **Vector Embedding Generation:** Automatically generates AI embeddings for all atomic insights to enable semantic search capabilities.
*   **Hybrid Search System:** Combines traditional full-text search (FTS5) for exact word matching on atomic insights with vector similarity search for semantic/conceptual queries across atomic insights.
*   **Database Reset Utility:** A safe reset option to clear all AI processing results while preserving raw data and video metadata, enabling complete re-processing with different prompts or models.
*   **Consistent Testing Environment:** Allows a user to define and manage a persistent control sample of videos for consistent, comparable prompt/model testing.
*   **Simplified Query Interface:** Provides a suite of user-friendly Python functions (in `query_utils.py`) for retrieving data from the database without needing to write raw SQL, including both text and semantic search capabilities.
*   **Comprehensive Logging:** Industry-standard logging with emoji indicators for clear visual feedback on successes, failures, and progress throughout all processing stages.

#### **Technical Stack**
*   **Programming Language:** Python
*   **Dependency Management:** A `requirements.txt` file will list all necessary packages and their versions for reproducible environments.
*   **Key Libraries:**
    *   `youtube_transcript_api`: For fetching video transcripts.
    *   `google-api-python-client`: The official Google client for interacting with the YouTube Data API v3 to fetch comments and video metadata.
    *   `asyncio`: For performing concurrent transcript downloads to improve speed when using proxies.
    *   `python-dotenv`: To load credentials (API keys, proxy info) from a `.env` file into the script's environment.
    *   `google-generativeai`: The SDK for making calls to the Gemini API for all AI processing stages and embedding generation.
    *   `sqlite3`: The built-in Python library for interacting with the SQLite database file.
    *   `sqlite-vec`: SQLite extension for vector similarity search functionality.
    *   `pydantic`: For data validation at critical points where external data could be corrupted or malformed.
    *   `logging`: Python's built-in logging library for structured logging with emoji indicators.
    *   `colorama`: For cross-platform colored terminal output to enhance log readability.
*   **Services:** Webshare (for proxies), YouTube Data API v3, Gemini API (for processing and embeddings).
*   **Database:** SQLite, with the **FTS5 (Full-Text Search 5) extension** enabled for traditional text queries and **sqlite-vec extension** for vector similarity search.

#### **Implementation Details / Logic**
*   **Configuration:** A central `config.py` file will manage all user-configurable settings, including the `list of target YouTube channels`, all `API parameters` (e.g., model names for each AI stage, embedding model), and `update intervals` (the number of days to wait before re-checking a channel for new videos).
*   **Credentials:** All sensitive credentials (YouTube API Key, Gemini API Key, Webshare proxy credentials) will be stored in a `.env` file and loaded into the script's environment using the `python-dotenv` library.
*   **Logging System:** A comprehensive logging framework will provide clear feedback throughout all operations:
    *   **Log Levels:** DEBUG, INFO, WARNING, ERROR with emoji indicators for quick visual identification.
    *   **Success Indicators:** ✅ for successful operations (downloads, processing completions, database writes, embedding generation).
    *   **Failure Indicators:** ❌ for errors and failures, 🟡 for warnings and retry attempts.
    *   **Progress Indicators:** 🔄 for ongoing operations, 📊 for statistics and summaries.
    *   **Information Indicators:** ℹ️ for general information, 🎯 for important milestones.
    *   **Log Output:** Console output with colored formatting, plus optional file logging for debugging.
    *   **Structured Format:** Timestamp, log level, module name, emoji indicator, and detailed message.
*   **Data Validation:** Pydantic models will be used at key points where data integrity is critical:
    *   **YouTube API Responses:** Models for video metadata, comments, and channel data to catch unexpected API response structures.
    *   **AI Model Outputs:** Models for Stage 1 topic summaries (including source attribution) and Stage 2 atomic insights to ensure required fields are present and properly formatted JSON is returned.
    *   **Configuration Parsing:** A model for validating the main config file to catch typos in channel IDs, API keys, and other critical settings.
*   **Idempotency & State Management:** The entire pipeline will be resumable and will not repeat completed work. Each stage is fully decoupled, allowing for independent execution and inspection of intermediate results.
    *   **Primary Mechanism:** A `VideoProcessingStatus` table in the database will have one row for every video and will track its state through all stages: `transcript_status`, `comments_status`, `stage_1_status`, `stage_2_status`, `embedding_status`. Before any action is taken on a video, the script will first query this table to check its status.
    *   **Stage Dependencies:** The system enforces clear dependencies between stages:
        *   Stage 1 requires completed transcript/comment downloads
        *   Stage 2 requires completed Stage 1 (topic summaries must exist)
        *   Stage 3 (embeddings) requires completed Stage 2 (atomic insights must exist)
    *   **Single Source of Truth:** The database status table is the authoritative source for all processing state.
*   **Channel & Video Setup Logic:** The system handles both initial setup and reset scenarios:
    *   **Channel Discovery:** For each configured channel ID, the system first checks if a record exists in the `Channels` table. If not, it fetches the channel name from the YouTube API and creates the record with ✅ logging.
    *   **Video Discovery:** For each channel, the system fetches the latest video list from the YouTube API. For each video, it checks if a record exists in the `Videos` table. If not, it creates the video record and initializes a corresponding `VideoProcessingStatus` record with all statuses set to 'pending'.
    *   **Reset Compatibility:** After a database reset, existing `Channels` and `Videos` records are preserved, so the video discovery logic will find existing records and skip re-creating them, only creating `VideoProcessingStatus` records for truly new videos discovered since the last run.
*   **Data Fetching & Translation:**
    *   **Transcripts:** Fetched via asynchronous requests routed through Webshare proxies to mitigate IP bans and improve speed. The logic will first list available transcript languages, preferably fetch English, but otherwise fetch an available language and immediately translate it to English using the `youtube_transcript_api`'s built-in `.translate('en')` method. Raw transcript data is stored directly in the database.
    *   **Comments:** Fetched via sequential, synchronous requests to the YouTube Data API. A `time.sleep()` call of a configurable duration will be placed between each request to ensure rate limits are respected. Comment data is stored directly in the database.
*   **Data Integrity (Database Transactions):** To prevent database corruption from a crash during a write operation, the system will employ database transactions with rollback capability on failure.
*   **Error Handling:** The script will not crash on non-critical errors, with comprehensive logging at each step:
    *   **Permanent Errors:** If the YouTube API reports that transcripts are disabled or comments are disabled for a video, the `VideoProcessingStatus` table will be updated with a final state (e.g., 'unavailable', 'disabled'), and the specific error message will be logged with ❌ indicator. The video will be skipped in all future runs.
    *   **Transient Errors:** For network timeouts, proxy connection errors, or temporary API rate limits, the error will be logged with 🟡 indicator and full traceback, but the video's status in the database will remain unchanged, making it eligible to be retried automatically on the next script run.
    *   **Success Logging:** Each successful operation will be logged with ✅ indicator, including download completions, processing milestones, database updates, and embedding generation.
*   **AI Processing Pipeline (3-Stage, Fully Decoupled):**
    *   **Stage 1 (Extract & Summarize):** **Input:** Raw text from both transcript and all comments for a video retrieved from the database, processed together in a single request. **Process:** A cost-effective model (e.g., Gemini Flash) is prompted to extract all valuable data and organize it into paragraph summaries of each major topic discussed, clearly identifying whether each topic originated from the transcript or a comment. **Output:** Multiple topic-based paragraph blurbs stored in the `TopicSummaries` table with source attribution. **Status Update:** `stage_1_status` set to 'complete' with ✅ log entry.
    *   **Stage 2 (Refine & Atomize):** **Input:** All topic blurbs for a video (both transcript and comment-derived) from the `TopicSummaries` table. **Process:** A powerful model (e.g., Gemini Pro) processes the entire video's blurbs at once to: 1) filter out vague or low-value content, and 2) break down the remaining valuable content into atomic records classified as either `quantitative` or `qualitative`. Source attribution is preserved through foreign key relationships. **Output:** Atomic insights stored in the `AtomicInsights` table with foreign key references to their source topic summaries. **Status Update:** `stage_2_status` set to 'complete' with ✅ log entry.
    *   **Stage 3 (Generate Embeddings):** **Input:** All atomic insights for a video from the `AtomicInsights` table. **Process:** Each atomic insight text is sent to Gemini's embedding API to generate a vector representation. **Output:** Embedding vectors stored in the `embedding_vector` column of the `AtomicInsights` table and indexed via sqlite-vec. **Status Update:** `embedding_status` set to 'complete' with ✅ log entry.
*   **Database Reset Logic:** The system includes a safe reset mechanism for experimentation:
    *   **What Gets Reset:** All AI processing results are cleared from the database:
        *   All records from `TopicSummaries` table (Stage 1 results)
        *   All records from `AtomicInsights` table (Stage 2 results and embeddings)
        *   Processing status flags reset to 'pending' for `stage_1_status`, `stage_2_status`, and `embedding_status` in `VideoProcessingStatus` table
    *   **What Gets Preserved:** Raw data and metadata remain intact:
        *   All raw transcript and comment data stored in the `RawTranscripts` and `RawComments` tables
        *   All records in `Channels`, `Videos`, and `VideoProcessingStatus` tables
        *   Download status flags (`transcript_status`, `comments_status`) remain unchanged
    *   **Implementation:** A dedicated `reset_processing.py` script provides a safe, logged operation with confirmation prompts and progress indicators using ✅ and 🔄 emoji logging.
*   **Hybrid Search System:**
    *   **Full-Text Search (FTS5):** The system will use SQLite's FTS5 extension for exact keyword-based search on atomic insights only.
        *   A virtual table (`AtomicInsights_fts`) will index the `insight_text` column for searching atomic insights directly.
    *   **Vector Similarity Search:** The system will use sqlite-vec for semantic search capabilities.
        *   Atomic insight embeddings will be stored in the `embedding_vector` column and indexed for similarity search.
        *   Search queries will be converted to embeddings via Gemini API and matched against stored vectors using cosine similarity.
        *   Results can be filtered by similarity threshold and combined with traditional search results.
    *   **Query-Time Embedding Generation:** Each semantic search requires converting the user's search query to an embedding vector via a real-time API call to Gemini's embedding service.
    *   **Search Strategy:** Use FTS5 for exact word searches ("lettuce pricing") and vector search for broader conceptual searches ("sustainable farming practices").

#### **Data Model / Schema**
*   **Database Indexing:** To ensure fast queries, the foreign key columns (`video_id` in `TopicSummaries`, and `summary_id` in `AtomicInsights`) will be explicitly indexed. Vector similarity searches will be optimized via sqlite-vec indexing on the `embedding_vector` column in the `AtomicInsights` table.
*   **Database Schema:**

    *   **`Channels` Table**
        *   `channel_id` (TEXT, Primary Key) --- *The unique YouTube channel ID (e.g., UC295-Dw_tDNtZXFeAPAW6Aw).*
        *   `channel_name` (TEXT) --- *The human-readable name of the channel.*

    *   **`Videos` Table**
        *   `video_id` (TEXT, Primary Key) --- *The unique YouTube video ID (e.g., dQw4w9WgXcQ).*
        *   `channel_id` (TEXT, Foreign Key) --- *Links to the `Channels` table.*
        *   `video_title` (TEXT) --- *The title of the video.*
        *   `published_date` (DATETIME) --- *The original upload date of the video.*
        *   `duration_seconds` (INTEGER) --- *The length of the video in seconds.*
        *   `view_count` (INTEGER) --- *The view count at the time of metadata fetching.*
        *   `like_count` (INTEGER) --- *The like count at the time of metadata fetching.*

    *   **`VideoProcessingStatus` Table**
        *   `video_id` (TEXT, Primary Key, Foreign Key referencing `Videos`) --- *Links to the `Videos` table.*
        *   `transcript_status` (TEXT) --- *e.g., 'pending', 'downloaded', 'unavailable'.*
        *   `comments_status` (TEXT) --- *e.g., 'pending', 'downloaded', 'disabled'.*
        *   `stage_1_status` (TEXT) --- *Tracks Stage 1 processing (e.g., 'pending', 'complete', 'failed').*
        *   `stage_2_status` (TEXT) --- *Tracks Stage 2 processing (e.g., 'pending', 'complete', 'failed').*
        *   `embedding_status` (TEXT) --- *Tracks embedding generation (e.g., 'pending', 'complete', 'failed').*
        *   `last_updated` (DATETIME) --- *Timestamp of the last status change for this video.*

    *   **`RawTranscripts` Table**
        *   `transcript_id` (INTEGER, Primary Key) --- *Unique identifier for each transcript record.*
        *   `video_id` (TEXT, Foreign Key) --- *Links to the `Videos` table.*
        *   `original_language` (TEXT) --- *The original language of the transcript (e.g., 'en', 'es').*
        *   `transcript_text` (TEXT) --- *The raw transcript text in English.*
        *   `is_translated` (BOOLEAN) --- *Whether this transcript was auto-translated to English.*
        *   `downloaded_at` (DATETIME) --- *Timestamp when the transcript was fetched.*

    *   **`RawComments` Table**
        *   `comment_id` (TEXT, Primary Key) --- *The unique YouTube comment ID.*
        *   `video_id` (TEXT, Foreign Key) --- *Links to the `Videos` table.*
        *   `author_name` (TEXT) --- *The display name of the comment author.*
        *   `comment_text` (TEXT) --- *The raw comment text.*
        *   `like_count` (INTEGER) --- *The like count for this specific comment.*
        *   `published_at` (DATETIME) --- *When the comment was originally posted.*
        *   `downloaded_at` (DATETIME) --- *Timestamp when the comment was fetched.*

    *   **`TopicSummaries` Table**
        *   `summary_id` (INTEGER, Primary Key) --- *Unique identifier for each topic-based summary blurb.*
        *   `video_id` (TEXT, Foreign Key) --- *Links to the `Videos` table.*
        *   `summary_text` (TEXT) --- *The AI-generated paragraph summary of a specific topic from the video.*
        *   `source_type` (TEXT) --- *The origin of this topic: 'transcript' or 'comment'.*
        *   `like_count` (INTEGER) --- *NULL for transcript-derived topics, like count for comment-derived topics.*

    *   **`AtomicInsights` Table**
        *   `insight_id` (INTEGER, Primary Key) --- *Unique identifier for each atomic insight.*
        *   `summary_id` (INTEGER, Foreign Key referencing `TopicSummaries`) --- *Links each insight back to its source topic summary.*
        *   `insight_type` (TEXT) --- *The data classification: `"quantitative"` or `"qualitative"`.*
        *   `confidence_score` (INTEGER) --- *The AI's score (1-100) of the insight's relevance and value.*
        *   `insight_text` (TEXT) --- *A single, concise sentence that describes this specific atomic insight.*
        *   `embedding_vector` (BLOB) --- *The vector embedding representation of the insight_text for semantic search.*
        *   `source_type` (TEXT) --- *Inherited from parent TopicSummary: 'transcript' or 'comment'.*
        *   `like_count` (INTEGER) --- *Inherited from parent TopicSummary: NULL for transcript, like count for comments.*

#### **Testing Strategy**
For a project of this scope, testing should focus on the core workflows and data integrity rather than exhaustive unit testing:

*   **Integration Tests (Priority 1):** Test the complete workflow with a small sample dataset:
    *   Create a test database with 2-3 videos from a single channel
    *   Run the entire pipeline from data fetching through embedding generation  
    *   Verify database state matches expectations at each stage
    *   Test the database reset functionality preserves the right data
    *   Validate that search functionality (both FTS5 and vector) returns reasonable results

*   **API Integration Tests (Priority 2):** Since this project heavily depends on external APIs:
    *   Test YouTube API integration with mock responses for edge cases (disabled comments, no transcripts)
    *   Test Gemini API integration with known inputs to ensure consistent structured outputs
    *   Test error handling when APIs return unexpected responses or rate limits

*   **Data Validation Tests (Priority 3):** Leverage your existing Pydantic models:
    *   Test each Pydantic model with both valid and invalid data
    *   Verify that malformed AI responses are caught and handled gracefully
    *   Test configuration validation catches common setup errors

*   **Status Management Tests (Priority 4):** Verify the state tracking system works correctly:
    *   Test that stage dependencies are enforced (can't run Stage 2 without Stage 1 completion)
    *   Test resume functionality after simulated crashes at different stages
    *   Verify idempotency - running the same operation twice doesn't create duplicate data

The testing approach prioritizes catching the most likely failure modes (external API issues, data corruption, state management bugs) while keeping the test suite manageable for a single-developer project. The comprehensive logging system serves as a runtime debugging tool that complements the formal tests.