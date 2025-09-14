About Elasticsearch

- Elasticsearch → search & analytics engine (JSON-based, schema flexible)
- Elasticsearch stores documents in JSON format
- JSON doc text fields are indexed for full-text search

---

* Elasticsearch stores documents in **JSON format**.
* Example log/document:

  ```json
  {
    "timestamp": "2025-09-14T12:34:56Z",
    "level": "ERROR",
    "message": "User login failed due to invalid password",
    "userId": 123
  }
  ```
* Each field can be indexed differently:

  * `"timestamp"` → time-series index
  * `"level"` → keyword index (exact match)
  * `"message"` → full-text index (tokenized into words)
  * `"userId"` → numeric index

---

## 🔹 Where Full-Text Search Fits In

* JSON is just the **container** for structured + unstructured data.
* Full-text search kicks in when a field (like `"message"`) is mapped as a `text` type.

  * Elasticsearch analyzes `"User login failed due to invalid password"`.
  * Builds an **inverted index** for the `message` field.

So:

* **JSON = input/output format** (how you send and retrieve data).
* **Full-text search = indexing and querying mechanism** applied to text fields inside that JSON.

---

## 🔹 Why JSON is important here

* Logs, events, metrics → naturally fit JSON (nested fields, key-value pairs).
* Schema flexibility → You don’t need to predefine rigid SQL schemas. If tomorrow you send a new field `"ipAddress"`, Elasticsearch can index it automatically.

---

👉 In short:

* **JSON** makes Elasticsearch easy to integrate with modern apps/logs.
* **Full-text search** is how Elasticsearch processes text fields inside those JSON docs.

# How full-text search works

Great follow-up 👌

Yes — **Elasticsearch is one of the most popular systems for full-text search**, but the *idea* of full-text search is broader and applies to many databases and search engines. Let me break it down:

---

## 🔹 How Full-Text Search Works (step by step)

1. **Indexing (preparation phase)**

   * When you insert documents (logs, articles, etc.), the system doesn’t just store them as raw text.
   * It builds an **inverted index**, like a dictionary.
   * Example:

     ```
     Document 1: "User login failed"
     Document 2: "System error caused failed login"
     ```

     Inverted index becomes:

     ```
     "user"  → [Doc1]
     "login" → [Doc1, Doc2]
     "failed"→ [Doc1, Doc2]
     "system"→ [Doc2]
     "error" → [Doc2]
     ```

     → Now the system can instantly find which docs contain each word.

2. **Text processing**

   * **Tokenization** → break text into words/tokens.
   * **Normalization** → lowercase, remove punctuation, maybe stopwords ("the", "a", "is").
   * **Stemming / Lemmatization** → reduce words to their root (e.g., *running → run, failed → fail*).
   * **Synonyms / analyzers** → treat "error" and "failure" as related.

3. **Querying**

   * When you search for `"failed login"`, the system looks up those terms in the inverted index.
   * It combines results, ranks them, and returns matches.

4. **Scoring & ranking**

   * Uses algorithms like **TF-IDF** or **BM25** to decide which document is *most relevant*.
   * Example: If "failed login" appears multiple times in a log, that log ranks higher.

---

## 🔹 Who does this?

* **Elasticsearch / OpenSearch** → built entirely around full-text search + analytics.
* **Databases with FTS features**: PostgreSQL (`tsvector`), MySQL (`FULLTEXT` index).
* **Other engines**: Apache Solr, MeiliSearch, Typesense, Whoosh, Lucene (Elasticsearch is built on Lucene).

# Building Inverted index takes time in elasticsearch?

**building an inverted index in Elasticsearch takes time and resources**, but the key is:

* It’s done **asynchronously in the background** when documents are ingested.
* Elasticsearch is optimized to do it **very fast** using Lucene under the hood.

---

## 🔹 What happens when you send a log/document to Elasticsearch?

1. **Document arrives** → JSON log or message is sent to an index.
2. **Analysis pipeline**:

   * Text fields go through analyzers (tokenization, lowercasing, stemming, stopword removal, etc.).
   * Numbers, dates, and keywords are handled differently (no need for inverted index).
3. **Inverted index update** → Terms are mapped to document IDs.

   * Example: `"login failed"` → `login:[doc123]`, `failed:[doc123]`.
4. **Stored in segments** → Lucene writes data in immutable segments on disk.

---

## 🔹 Performance considerations

* **Indexing speed** → Can handle **tens of thousands of documents per second per node**.
* **Latency tradeoff** → By default, Elasticsearch makes documents searchable within **\~1 second** (refresh interval).
* **Cost** → Heavy indexing = high CPU/RAM/disk I/O usage. That’s why at very high volumes (like logs at 100k+/sec), it can become expensive.

---

## 🔹 Optimizations Elasticsearch uses

* **Batching** → Documents are buffered and indexed in bulk.
* **Segments** → Immutable, periodically merged to optimize storage and search.
* **Sharding** → Data split across multiple nodes, indexing and searching in parallel.
* **Compression** → Terms and postings are stored efficiently.

---

📌 So yes, building an inverted index **costs time and resources**, but Elasticsearch is designed to make it fast and scalable.
That’s also why people sometimes prefer **ClickHouse** for logs — no heavy text analysis/indexing, just fast writes + analytics.

# MYSQL `%...%` syntax is full-text search?

## 🔹 MySQL and `%...%` syntax

If you write something like:

```sql
SELECT * FROM logs WHERE message LIKE '%login failed%';
```

* This is **not full-text search**.
* It’s a **pattern match** using the `LIKE` operator.
* MySQL has to scan through rows and check each string → **slow for large tables** (no inverted index).

---

## 🔹 MySQL Full-Text Search

MySQL **does** support real full-text search, but with a different syntax:

```sql
SELECT * FROM logs 
WHERE MATCH(message) AGAINST('login failed');
```

* `MATCH(column)` → tells MySQL to use the full-text index on that column.
* `AGAINST('...')` → specifies the search phrase.
* Requires a **FULLTEXT index** on the column:

  ```sql
  ALTER TABLE logs ADD FULLTEXT(message);
  ```

---

## 🔹 Difference

| Syntax                    | What it does                                           | Performance             |
| ------------------------- | ------------------------------------------------------ | ----------------------- |
| `LIKE '%word%'`           | Scans the entire table, checks substrings              | **Slow** on big data    |
| `MATCH(...) AGAINST(...)` | Uses an **inverted index** (like Elasticsearch/Lucene) | **Fast** for large text |

---

✅ So:

* `%...%` = simple substring search (not full-text search).
* `MATCH() AGAINST()` = true **full-text search** in MySQL.
