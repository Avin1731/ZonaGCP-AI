# ☁️ Implement Multimodal Vector Search with BigQuery: Challenge Lab

**Rute:** BigQuery & Generative AI
**Topik:** Multimodal Vector Search, BigQuery ML, Object Tables, Embeddings
**Tanggal Pengerjaan:** 27 Januari 2026

---

## 🎯 1. Overview & Tujuan

Challenge Lab ini menguji kemampuan dalam membangun pipeline pencarian produk yang mirip menggunakan data multimodal (gambar & teks) sepenuhnya di BigQuery. Kita menggunakan dataset gambar produk dari Cloud Storage dan melakukan pencarian semantik berdasarkan deskripsi teks.

**Misi Utama:**

1. **Connection & IAM:** Membuat koneksi eksternal yang aman ke Vertex AI di region `europe-west4`.
2. **Object Table:** Membuat tabel referensi di BigQuery untuk mengakses file gambar di Cloud Storage.
3. **Multimodal Embeddings:** Mengubah gambar produk menjadi vektor menggunakan model `multimodalembedding@001`.
4. **Vector Search:** Melakukan pencarian produk yang mirip dengan kata kunci "Men Sweaters".

---

## 🛠️ 2. Langkah-Langkah & Solusi

### 🔹 Task 1: Create a Source Connection and Grant IAM Permissions

* **Deskripsi:** Membuat jembatan koneksi agar BigQuery bisa mengakses model Vertex AI di region Eropa.
* **Konfigurasi GUI (BigQuery):**
* Menu: `BigQuery > + ADD DATA > External data source`
* Connection type: `Vertex AI remote models, remote functions, BigLake and Spanner`
* Connection ID: `vector_conn`
* Location: `europe-west4` (⚠️ **Wajib region ini**)
* **Action:** Klik **Create connection**.


* **Konfigurasi GUI (IAM & Admin):**
* *Step Penting:* Copy Service Account ID dari detail connection `vector_conn`.
* Menu: `IAM & Admin > IAM > Grant Access`
* New Principals: `[Paste Service Account ID]`
* **Roles:**
1. `BigQuery Data Owner`
2. `Storage Object Viewer`
3. `Vertex AI User`


* **Action:** Klik **Save**.



### 🔹 Task 2: Create an Object Table

* **Deskripsi:** Membuat tabel eksternal yang membaca metadata file gambar dari bucket Cloud Storage.
* **Perintah SQL:**

```sql
CREATE OR REPLACE EXTERNAL TABLE `qwiklabs-gcp-01-9bd3d23f752f.gcc_bqml_dataset.gcc_image_object_table`
WITH CONNECTION `qwiklabs-gcp-01-9bd3d23f752f.europe-west4.vector_conn`
OPTIONS (
  object_metadata = 'SIMPLE',
  uris = ['gs://qwiklabs-gcp-01-9bd3d23f752f/*']
);

```

*(Ganti Project ID sesuai dengan lab environment)*

### 🔹 Task 3: Generate Embeddings

* **Deskripsi:** Membuat model remote dan menghasilkan embedding (vektor) dari gambar-gambar di Object Table.
* **Perintah SQL (Create Model):**

```sql
CREATE OR REPLACE MODEL `qwiklabs-gcp-01-9bd3d23f752f.gcc_bqml_dataset.gcc_embedding`
REMOTE WITH CONNECTION `qwiklabs-gcp-01-9bd3d23f752f.europe-west4.vector_conn`
OPTIONS (
  ENDPOINT = 'multimodalembedding@001'
);

```

* **Perintah SQL (Generate Image Embeddings):**

```sql
CREATE OR REPLACE TABLE `qwiklabs-gcp-01-9bd3d23f752f.gcc_bqml_dataset.gcc_retail_store_embeddings` AS
SELECT *, REGEXP_EXTRACT(uri, r'[^/]+$') as product_name
FROM ML.GENERATE_EMBEDDING(
  MODEL `qwiklabs-gcp-01-9bd3d23f752f.gcc_bqml_dataset.gcc_embedding`,
  TABLE `qwiklabs-gcp-01-9bd3d23f752f.gcc_bqml_dataset.gcc_image_object_table`
);

```

### 🔹 Task 4: Run a Vector Search

* **Deskripsi:** Mencari gambar produk yang paling relevan dengan query teks "Men Sweaters".
* **Perintah SQL:**

```sql
CREATE OR REPLACE TABLE `qwiklabs-gcp-01-9bd3d23f752f.gcc_bqml_dataset.gcc_vector_search_table` AS
SELECT
  base.uri,
  base.product_name,
  base.content_type,
  distance
FROM
  VECTOR_SEARCH(
    TABLE `qwiklabs-gcp-01-9bd3d23f752f.gcc_bqml_dataset.gcc_retail_store_embeddings`,
    'ml_generate_embedding_result',
    (
      SELECT ml_generate_embedding_result as embedding_col
      FROM ML.GENERATE_EMBEDDING(
        MODEL `qwiklabs-gcp-01-9bd3d23f752f.gcc_bqml_dataset.gcc_embedding`,
        (SELECT 'Men Sweaters' AS content),
        STRUCT(TRUE AS flatten_json_output)
      )
    ),
    top_k => 3,
    distance_type => 'COSINE'
  );

```

---

### 🐛 3. Troubleshooting / Masalah yang Dihadapi

**Error:** `Invalid region for connection` atau Model tidak ditemukan.

* **Penyebab:** Salah memilih lokasi saat membuat Connection. Lab ini mewajibkan region `europe-west4`, bukan `US`.
* **Solusi:** Hapus connection yang salah, buat ulang dengan memilih region `europe-west4`.

**Error:** `Access Denied` saat menjalankan query.

* **Penyebab:** Service Account belum diberikan izin IAM yang lengkap atau API belum di-enable.
* **Solusi:** Pastikan Service Account connection memiliki 3 role wajib (`BigQuery Data Owner`, `Storage Object Viewer`, `Vertex AI User`).

---

### 📝 4. Catatan Penting (Key Takeaways)

Hal-hal baru yang saya pelajari dari lab ini:

1. **Multimodal Embedding:** Kita bisa mencari gambar menggunakan teks ("Men Sweaters") karena keduanya berada dalam ruang vektor yang sama (multimodal embedding space).
2. **Object Table:** Fitur BigQuery yang sangat berguna untuk mengolah *unstructured data* (gambar, audio, PDF) yang tersimpan di GCS seolah-olah data tabel biasa.
3. **Region Specific:** Penting untuk selalu memperhatikan lokasi region resource (Connection, Dataset, Model) agar bisa saling berkomunikasi.

---

[⬅️ Kembali ke Menu Utama](../README.md)