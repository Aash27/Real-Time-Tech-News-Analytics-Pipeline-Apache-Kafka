# Real-Time Tech News Analytics & Sentiment Pipeline

An end-to-end real-time streaming system that ingests technology news, processes it through Apache Kafka, enriches it using NLP models and Groq LLM summaries, and outputs structured analytics-ready data for Power BI visualization.

---

## 📌 Project Overview

This project builds a scalable streaming pipeline to monitor technology news in real time.

The system:

- Collects live RSS news feeds
- Streams data using Apache Kafka
- Cleans and validates incoming articles
- Categorizes articles into tech domains
- Performs sentiment analysis using FinBERT
- Extracts named entities (ORG, PERSON, GPE)
- Generates concise summaries using Groq API
- Filters articles based on keyword watchlists
- Detects spikes in trending topics
- Outputs structured data to CSV
- Feeds data into Power BI dashboards

This project demonstrates real-time data engineering, NLP enrichment, and streaming analytics in a modular architecture.

---

# 🏗 System Architecture

## High-Level Flow

```
RSS Sources
   ↓
Kafka: news.raw
   ↓
Cleaner
   ↓
Kafka: news.cleaned
   ↓
Enricher (Category + Sentiment + NER + Groq)
   ↓
Kafka: news.enriched
   ↓
Watchlist Filter
   ↓
Kafka: watchlist.events
   ↓
CSV Sink
   ↓
Power BI Dashboard
```

---

# 🔄 Pipeline Components

---

## 1️⃣ RSS Fetcher (Producer Layer)

**File:** `rss_fetcher.py`

- Fetches live technology news from RSS feeds (BBC Tech, Google News, etc.)
- Normalizes fields:
  - title
  - summary
  - URL
  - source
  - published_at
- Publishes structured JSON messages to:

```
news.raw
```

This is the ingestion entry point of the pipeline.

---

## 2️⃣ Cleaner (Preprocessing Layer)

Consumes from:
```
news.raw
```

Performs:
- Text normalization
- Field validation
- Schema enforcement
- Removal of malformed records

Publishes cleaned data to:

```
news.cleaned
```

This ensures downstream components receive standardized data.

---

## 3️⃣ Enricher Consumer (Core Intelligence Layer)

**File:** `enricher_consumer.py`

Consumes from:
```
news.cleaned
```

This is the most critical stage of the system.

It performs four major enrichment tasks:

---

### 🧩 A) Category Classification

Each article is categorized into predefined tech domains:

- AI
- Blockchain
- Programming
- Gadgets

This allows structured grouping in dashboards.

---

### 😊 B) Sentiment Analysis (FinBERT)

Model Used:
- FinBERT (Transformer-based model)

Output Classes:
- Positive
- Negative
- Neutral

Why FinBERT?
- Handles contextual language well
- Effective for technical and financial news
- Performs better than majority-class baseline

Sentiment helps:
- Detect innovation waves
- Identify crisis events (breaches, layoffs)
- Track market optimism vs negativity

---

### 🏷 C) Named Entity Recognition (NER)

Extracted Entities:
- ORG (Organizations)
- PERSON
- GPE (Geopolitical locations)

This enables:
- Company tracking
- Executive mentions
- Geographic trend analysis

---

### 🧠 D) Summary Generation using Groq API

Groq LLM generates concise one-line summaries.

Process:

1. Title + article body sent to Groq API
2. Prompt requests a short, single-sentence summary
3. Groq returns condensed article representation
4. Summary appended to metadata

Benefits:
- Quick readability
- Dashboard-friendly
- Reduced storage footprint
- Better Power BI presentation

After enrichment, articles are published to:

```
news.enriched
```

---

## 4️⃣ Watchlist Filter (Trend Detection Layer)

**File:** `watchlist_filter.py`

Consumes from:
```
news.enriched
```

Features:

- Predefined tech keyword watchlist
- Tracks first mentions
- Stores mention metadata in SQLite database
- Filters relevant articles

Publishes events to:

```
watchlist.events
```

This enables:
- Real-time keyword monitoring
- Emerging trend detection
- First-occurrence tracking

---

## 5️⃣ Spike Detection

**File:** `spike_detector.py`

Monitors article frequency over time.

Detects:
- Sudden increases in keyword frequency
- Trending topics
- News surges

This supports near real-time event awareness.

---

## 6️⃣ CSV Sink (Output Layer)

**File:** `sink_watch_list.py`

Consumes from:
```
watchlist.events
```

Outputs structured dataset:

```
watchlist_events_all.csv
```

Columns include:

- watch_key
- first_mention
- id
- title
- summary (Groq-generated)
- URL
- source
- published_at
- category
- sentiment

This CSV acts as the data source for Power BI.

---

# 📊 Power BI Integration

The final dataset is connected directly to Power BI for:

- Sentiment distribution
- Article counts by category
- Source-wise distribution
- Daily article trends
- Watchlist event tracking
- Filterable dashboards

---

# 🛠 Tech Stack

### Streaming
- Apache Kafka
- Kafka Producers & Consumers

### Processing
- Python
- Pandas

### NLP
- FinBERT (Transformer-based sentiment model)
- Named Entity Recognition
- Groq LLM API (Summarization)

### Storage
- SQLite (watchlist tracking)
- CSV sink for analytics

### Visualization
- Power BI

---

# ⚙️ How to Run the Project

## 1️⃣ Clone Repository

```bash
git clone https://github.com/apoorva183/Real-Time-Tech-News-Analytics-Pipeline.git
cd Real-Time-Tech-News-Analytics-Pipeline
```

---

## 2️⃣ Create Virtual Environment

```bash
python -m venv venv
venv\Scripts\activate  # Windows
```

---

## 3️⃣ Install Dependencies

```bash
pip install -r requirements.txt
```

---

## 4️⃣ Start Kafka & Zookeeper

Make sure Kafka is running before starting producers/consumers.

---

## 5️⃣ Add API Keys

Create `.env` file:

```
GROQ_API_KEY=your_key_here
NEWS_API_KEY=your_key_here
```

---

## 6️⃣ Run Pipeline (In Order)

Terminal 1:
```bash
python rss_fetcher.py
```

Terminal 2:
```bash
python cleaner.py
```

Terminal 3:
```bash
python enricher_consumer.py
```

Terminal 4:
```bash
python watchlist_filter.py
```

Terminal 5:
```bash
python sink_watch_list.py
```

---

# 🎯 Key Capabilities Demonstrated

- Real-time streaming architecture
- Event-driven data processing
- Transformer-based sentiment classification
- LLM-based summarization
- Named entity extraction
- Keyword spike detection
- Analytics-ready structured output
- BI integration

---

# 🧱 Design Principles

- Producer–Consumer architecture
- Topic-based decoupling
- Stateless micro-components
- Enrichment before storage
- Analytics-first schema design
- Modular extensibility

---

# 🚀 Future Improvements

- Deploy Kafka cluster in cloud
- Dockerize services
- Add model retraining pipeline
- Introduce SHAP/attention visualization
- Real-time streaming Power BI dataset
- Add alerting system for negative sentiment spikes

---

# 📄 License

Open-source project for academic and research purposes.
