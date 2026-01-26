# ☁️ Create Embeddings, Vector Search, and RAG with BigQuery: Challenge Lab

**Rute:** BigQuery & Generative AI
**Topik:** RAG (Retrieval Augmented Generation), Vector Search, BigQuery ML, Gemini
**Tanggal Pengerjaan:** 26 Januari 2026

* [📂 Materi](./Materi/README.md)

---

## 🎯 1. Overview & Tujuan

Lab ini bertujuan untuk memahami cara membangun aplikasi **RAG (Retrieval Augmented Generation)** tanpa coding Python yang rumit, melainkan sepenuhnya menggunakan **SQL di BigQuery**. Kita akan menghubungkan data review pelanggan dengan model AI (Gemini) untuk menghasilkan ringkasan yang akurat dan berbasis fakta (grounding).

**Misi Utama:**

1. **Connect & IAM:** Membuat koneksi eksternal antara BigQuery dan Vertex AI.
2. **Generate Embeddings:** Mengubah teks review pelanggan menjadi vektor angka.
3. **Vector Search:** Mencari review yang relevan secara semantik dengan kata kunci tertentu.
4. **Generate Answer (RAG):** Menggunakan Gemini untuk merangkum hasil pencarian review tersebut.

---

## 🛠️ 2. Langkah-Langkah & Solusi

### 🔹 Task 1: Create a Source Connection and Grant IAM Permissions

* **Deskripsi:** Menyiapkan "jembatan" agar BigQuery bisa mengakses model AI di Vertex AI dan memberikan izin akses.
* **Konfigurasi GUI (BigQuery):**
* Menu: `BigQuery > + ADD DATA > External data source`
* Connection type: `Vertex AI remote models, remote functions, BigLake and Spanner (Cloud Resource)`
* Connection ID: `embedding_conn`
* Location: `us` (atau `multi-region us`)
* **Action:** Klik **Create connection**.


* **Konfigurasi GUI (IAM & Admin):**
* *Step Penting:* Buka detail connection `embedding_conn` di BigQuery Explorer, lalu **Copy Service Account ID**.
* Menu: `IAM & Admin > IAM > Grant Access`
* New Principals: `[Paste Service Account ID tadi]`
* Role 1: `BigQuery Data Owner`
* Role 2: `Vertex AI User`
* **Action:** Klik **Save**.



### 🔹 Task 2: Generate Embeddings

* **Perintah SQL:**
*(Jalankan query berikut satu per satu di BigQuery Editor)*

**1. Membuat Dataset & Koneksi Model Embedding**

```sql
-- Membuat Dataset
CREATE SCHEMA IF NOT EXISTS CustomerReview;

-- Membuat Model Koneksi ke Gemini Embedding
CREATE OR REPLACE MODEL `CustomerReview.Embeddings`
REMOTE WITH CONNECTION `us.embedding_conn`
OPTIONS (ENDPOINT = 'gemini-embedding-001');

```

**2. Load Data Review Pelanggan**

```sql
LOAD DATA OVERWRITE CustomerReview.customer_reviews
(
    customer_review_id INT64,
    customer_id INT64,
    location_id INT64,
    review_datetime DATETIME,
    review_text STRING,
    social_media_source STRING,
    social_media_handle STRING
)
FROM FILES (
    format = 'CSV',
    uris = ['gs://spls/gsp1249/customer_reviews.csv']
);

```

**3. Generate Embeddings (Teks ke Vektor)**

```sql
CREATE OR REPLACE TABLE `CustomerReview.customer_reviews_embedded` AS
SELECT *
FROM ML.GENERATE_EMBEDDING(
    MODEL `CustomerReview.Embeddings`,
    (SELECT review_text AS content FROM `CustomerReview.customer_reviews`)
);

```

### 🔹 Task 3: Search the Vector Space

* **Perintah SQL:**

**1. Create Vector Index (Opsional/Sering Error)**
*(Note: Langkah ini sering gagal jika data < 5.000 baris, bisa di-skip jika error)*

```sql
CREATE OR REPLACE VECTOR INDEX `CustomerReview.reviews_index`
ON `CustomerReview.customer_reviews_embedded`(ml_generate_embedding_result)
OPTIONS (distance_type = 'COSINE', index_type = 'IVF');

```

**2. Vector Search (Mencari Review tentang "Service")**

```sql
CREATE OR REPLACE TABLE `CustomerReview.vector_search_result` AS
SELECT
    query.query,
    base.content
FROM
    VECTOR_SEARCH(
        TABLE `CustomerReview.customer_reviews_embedded`,
        'ml_generate_embedding_result',
        (
            SELECT
                ml_generate_embedding_result,
                content AS query
            FROM
                ML.GENERATE_EMBEDDING(
                    MODEL `CustomerReview.Embeddings`,
                    (SELECT 'service' AS content)
                )
        ),
        top_k => 5,
        options => '{"fraction_lists_to_search": 0.01}'
    );

```

### 🔹 Task 4: Generate an Improved Answer

* **Perintah SQL:**

**1. Koneksi ke Model Gemini Flash**

```sql
CREATE OR REPLACE MODEL `CustomerReview.Gemini`
REMOTE WITH CONNECTION `us.embedding_conn`
OPTIONS (ENDPOINT = 'gemini-2.5-flash');

```

**2. Eksekusi RAG (Summarization)**

```sql
SELECT
    ml_generate_text_llm_result AS generated
FROM
    ML.GENERATE_TEXT(
        MODEL `CustomerReview.Gemini`,
        (
            SELECT
                CONCAT(
                    'Summarize what customers think about our services: ',
                    STRING_AGG(FORMAT('review text: %s', base.content), ',\n')
                ) AS prompt
            FROM
                `CustomerReview.vector_search_result` AS base
        ),
        STRUCT(
            0.4 AS temperature,
            300 AS max_output_tokens,
            0.5 AS top_p,
            5 AS top_k,
            TRUE AS flatten_json_output
        )
    );

```

---

### 🐛 3. Troubleshooting / Masalah yang Dihadapi

**Error:** `Total rows 50 is smaller than min allowed 5000 for CREATE VECTOR INDEX`

* **Penyebab:** Data di lab ini hanya sampel kecil (50 baris), sedangkan pembuatan Index IVF membutuhkan minimal 5.000 baris data.
* **Solusi:** Abaikan error pada query `CREATE VECTOR INDEX`. Langsung jalankan query `VECTOR_SEARCH` selanjutnya. Fungsi pencarian tetap berjalan menggunakan metode *Brute Force* yang otomatis aktif untuk dataset kecil.

**Error:** `Access Denied: BigQuery: Permission denied while getting the connection`

* **Penyebab:** Service Account milik Connection belum diberikan role IAM yang benar.
* **Solusi:** Kembali ke menu IAM, pastikan Service Account ID dari `embedding_conn` sudah memiliki role **Vertex AI User** dan **BigQuery Data Owner**. Tunggu 1-2 menit agar permission update.

---

### 📝 4. Catatan Penting (Key Takeaways)

Hal-hal baru yang saya pelajari dari lab ini:

1. **RAG Tanpa Python:** Ternyata kita bisa membangun pipeline AI canggih (Embedding & Generation) hanya menggunakan SQL standard di BigQuery.
2. **External Connection:** Fitur krusial untuk menghubungkan BigQuery (Data Warehouse) dengan Vertex AI (Model Garden).
3. **Vector Search Native:** BigQuery memiliki fungsi bawaan `VECTOR_SEARCH` yang memudahkan pencarian semantik tanpa perlu memindahkan data ke Vector Database terpisah.

---

[⬅️ Kembali ke Menu Utama](../README.md)