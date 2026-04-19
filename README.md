# Real-Time Tech News Analytics Pipeline

A production-grade streaming pipeline that ingests tech news from 26 RSS sources, processes articles through Apache Kafka, enriches them with 3 ML models and Groq LLM summaries, and serves structured analytics via a REST API — all with full observability through Prometheus and Grafana.

**Key numbers:** 26 RSS sources | 5 Kafka topics | 3 HuggingFace models | 101 watchlist keywords | Prometheus + Grafana dashboards | FastAPI REST endpoint

---

## Architecture

```
RSS Sources (26 feeds)
        |
        v
  [rss_fetcher.py] ──> Kafka: news.raw
        |
        v
  [cleaner.py] ──> Kafka: news.cleaned
        |                        |
        v                        v (on failure)
  [enricher_consumer.py]    Kafka: news.failed (DLQ)
        |
        v
  Kafka: news.enriched
        |
        v
  [watchlist_filter.py] ──> Kafka: news.filtered
        |
        v
  [csv_writer.py] ──> data/tech_news.csv
        |
        v
  [api.py] ──> FastAPI REST API (localhost:8080)
        
  All components ──> Prometheus (localhost:9090) ──> Grafana (localhost:3000)
```

---

## What Each Component Does

| Component | File | Purpose |
|-----------|------|---------|
| **RSS Fetcher** | `src/rss_fetcher.py` | Continuously streams articles from 26 tech RSS feeds with deduplication and 24-hour recency filtering |
| **Cleaner** | `src/cleaner.py` | Deduplicates via SQLite, filters unsafe content (blacklist), normalizes HTML/text fields |
| **Enricher** | `src/enricher_consumer.py` | Runs 3 ML models: zero-shot categorization (BART-MNLI), sentiment analysis (DistilBERT-SST2), NER (BERT-NER) + Groq LLM summaries |
| **Watchlist Filter** | `src/watchlist_filter.py` | Regex-matches articles against 101 tech keywords, routes matches to `news.filtered` |
| **CSV Writer** | `src/csv_writer.py` | Persists filtered articles to CSV with deduplication for Power BI/analytics |
| **REST API** | `src/api.py` | FastAPI server with `/articles`, `/stats` endpoints — filterable by category, sentiment, source |
| **Dead Letter Queue** | `src/dlq.py` | Routes failed messages to `news.failed` with error metadata; retry with exponential backoff |
| **Metrics** | `src/metrics.py` | Prometheus counters, histograms, gauges for every pipeline stage |

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Streaming** | Apache Kafka (Redpanda) |
| **Processing** | Python 3.11+ |
| **ML / NLP** | HuggingFace Transformers (BART-MNLI, DistilBERT-SST2, BERT-NER) |
| **LLM** | Groq API (Llama 3.1 8B) |
| **API** | FastAPI + Uvicorn |
| **Observability** | Prometheus + Grafana (pre-configured dashboards) |
| **Orchestration** | Docker Compose |
| **Storage** | SQLite (dedup state) + CSV (analytics output) |

---

## Quick Start

### 1. Clone and install

```bash
git clone https://github.com/apoorva183/Real-Time-Tech-News-Analytics-Pipeline.git
cd Real-Time-Tech-News-Analytics-Pipeline
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Start infrastructure

```bash
docker compose up -d
bash scripts/create-topics.sh
```

This starts:
- **Redpanda** (Kafka) on `localhost:9092`
- **Prometheus** on `localhost:9090`
- **Grafana** on `localhost:3000` (login: admin / pipeline)

### 3. Configure API keys

Create a `.env` file:

```
GROQ_API_KEY=your_groq_key_here
```

### 4. Run the pipeline

Each component runs in its own terminal:

```bash
# Terminal 1: Start fetching news
cd src && python rss_fetcher.py

# Terminal 2: Clean incoming articles
cd src && python cleaner.py

# Terminal 3: Enrich with ML models (first run downloads ~1.6GB of models)
cd src && python enricher_consumer.py

# Terminal 4: Filter against watchlist
cd src && python watchlist_filter.py

# Terminal 5: Write to CSV
cd src && python csv_writer.py

# Terminal 6: Start the API
cd src && uvicorn api:app --host 0.0.0.0 --port 8080 --reload
```

### 5. Explore the data

- **API**: http://localhost:8080/articles?category=AI&limit=10
- **Stats**: http://localhost:8080/stats
- **Grafana**: http://localhost:3000 (pre-loaded dashboard: "News Pipeline Overview")
- **Prometheus**: http://localhost:9090

---

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `GET /` | GET | Health check with article count |
| `GET /articles` | GET | List articles (query params: `category`, `sentiment`, `source`, `watchname`, `search`, `limit`, `offset`) |
| `GET /articles/{id}` | GET | Get single article by ID |
| `GET /stats` | GET | Aggregated stats: counts by category, sentiment, source, top watchlist terms |

**Example:**

```bash
# Get AI articles with positive sentiment
curl "http://localhost:8080/articles?category=AI&sentiment=POSITIVE&limit=5"

# Pipeline statistics
curl "http://localhost:8080/stats"
```

---

## Observability

Every pipeline component exposes Prometheus metrics on its own port:

| Component | Metrics Port | Key Metrics |
|-----------|-------------|-------------|
| RSS Fetcher | `:8000` | `pipeline_articles_fetched_total`, `pipeline_fetch_errors_total`, `pipeline_fetch_duration_seconds` |
| Cleaner | `:8001` | `pipeline_articles_cleaned_total`, `pipeline_articles_deduped_total`, `pipeline_articles_blocked_total` |
| Enricher | `:8002` | `pipeline_articles_enriched_total`, `pipeline_enrichment_duration_seconds`, `pipeline_groq_requests_total` |
| Watchlist Filter | `:8003` | `pipeline_articles_filtered_total`, `pipeline_articles_dropped_total` |
| CSV Writer | `:8004` | `pipeline_csv_rows_written_total`, `pipeline_csv_duplicates_skipped_total` |

Grafana comes pre-configured with a **"News Pipeline Overview"** dashboard showing:
- Articles fetched per source (time series)
- Enrichment latency (p50/p95)
- Pipeline throughput counters
- Sentiment distribution (pie chart)
- Articles by category (bar gauge)
- Groq API request rates
- Dead-letter queue activity
- Fetch errors by feed

---

## Fault Tolerance

### Dead Letter Queue (`news.failed`)

When any component fails to process a message, instead of silently dropping it:

1. The message is sent to the `news.failed` Kafka topic
2. Error metadata is attached (stage, error type, traceback, timestamp)
3. The `pipeline_dlq_messages_total` metric is incremented
4. Processing continues with the next message

### Retry with Exponential Backoff

External API calls (Groq) use automatic retry:

- Up to 3 attempts with exponential backoff (1s, 2s, 4s delays)
- Each retry is tracked via `pipeline_retry_attempts_total`
- After exhausting retries, falls back to truncated text summary

---

## Kafka Topics

| Topic | Purpose |
|-------|---------|
| `news.raw` | Raw articles from RSS fetcher |
| `news.cleaned` | Deduplicated and normalized articles |
| `news.enriched` | Articles with sentiment, category, entities, summaries |
| `news.filtered` | Articles matching watchlist keywords |
| `news.failed` | Dead-letter queue for failed processing |

---

## Project Structure

```
.
├── docker-compose.yml          # Redpanda + Prometheus + Grafana
├── requirements.txt
├── data/
│   └── watchlist.txt           # 101 tech keywords
├── monitoring/
│   ├── prometheus.yml          # Scrape config for all components
│   └── grafana/
│       ├── provisioning/       # Auto-configured datasource + dashboard provider
│       └── dashboards/         # Pre-built pipeline overview dashboard
├── scripts/
│   └── create-topics.sh        # Create all Kafka topics
└── src/
    ├── rss_fetcher.py          # Producer: 26 RSS feeds
    ├── cleaner.py              # Dedup + content filter
    ├── enricher_consumer.py    # 3 ML models + Groq LLM
    ├── watchlist_filter.py     # Keyword matching
    ├── csv_writer.py           # CSV sink
    ├── api.py                  # FastAPI REST API
    ├── metrics.py              # Prometheus metrics definitions
    ├── dlq.py                  # Dead-letter queue + retry logic
    └── common_utils.py         # Shared utilities
```

---

## License

MIT
