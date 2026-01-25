# ☁️ Vector Search and Embeddings: Challenge Lab

**Rute:** Rute 03 - Vector Search RAG
**Topik:** Vector Search and Embeddings
**Tanggal Pengerjaan:** 25 Januari 2026

---

# Lab 1: Getting Started with Vector Search and Embeddings (GSP1202)

**Rute:** Vector Search & Embeddings
**Topik:** Vertex AI, Text Embeddings, Vector Search Index, Python SDK
**Tanggal Pengerjaan:** 25 Januari 2026

---

## 🎯 1. Overview & Tujuan

Lab ini bertujuan untuk membangun sistem pencarian semantik (*semantic search*) menggunakan **Vertex AI Embeddings** dan **Vector Search**. Kita mengubah data teks menjadi vektor (angka) agar mesin bisa memahami konteks makna, bukan sekadar kata kunci.

**Misi Utama:**

1. **Generate Embeddings:** Mengubah dataset pertanyaan Stack Overflow menjadi vektor menggunakan model `gemini-embedding-001`.
2. **Create Index:** Membuat Index Vector Search untuk menyimpan jutaan vektor.
3. **Deploy & Query:** Men-deploy index ke endpoint dan melakukan query untuk mencari pertanyaan yang mirip (*nearest neighbors*).

---

## 🛠️ 2. Langkah-Langkah & Full Code

### 🔹 Step 1: Setup Environment & Install SDK

Persiapan environment di Vertex AI Workbench. Menginstall library yang diperlukan dan mengatur project ID.

**1. Install Python SDK**

```python
%pip install --upgrade --user \
    google-genai \
    google-cloud-storage \
    google-cloud-logging \
    'google-cloud-bigquery[pandas]' \
    google-cloud-aiplatform

```

**2. Restart Kernel** (Wajib agar library terbaca)

```python
# Restart kernel after installs so that your environment can access the new packages
import IPython

app = IPython.Application.instance()
app.kernel.do_shutdown(True)

```

**3. Set Environment Variables**

```python
# get project ID and region
PROJECT_ID = "qwiklabs-gcp-03-bce37a1a4cb6"
LOCATION = "us-east4"

```

**4. Generate UID** (Untuk penamaan unik resource)

```python
# generate an unique id for this session
from datetime import datetime

UID = datetime.now().strftime("%m%d%H%M")

```

**5. Setup Cloud Logging** (Penting untuk Checker Qwiklabs)

```python
# Instantiate the Google Cloud Logging client
import logging
from google.cloud import logging as gcp_logging
from google.cloud.logging.handlers import CloudLoggingHandler

# Instantiate the Google Cloud Logging client
client = gcp_logging.Client()

# Create a specific logger for cloud logging
logger = logging.getLogger('my_cloud_logger')

# Prevent this logger from sending messages to its parent (the root logger),
# which has the default console handler
logger.propagate = False

# Create and add the Cloud Logging handler
handler = CloudLoggingHandler(client)
logger.addHandler(handler)

# Set the logging level
logger.setLevel(logging.INFO)

```

**6. Import Libraries & Enable APIs**

```python
import random
import time
import numpy as np
import tqdm

! gcloud services enable compute.googleapis.com aiplatform.googleapis.com storage.googleapis.com bigquery.googleapis.com --project {PROJECT_ID}

```

---

### 🔹 Step 2: Data Preparation & Embedding Generation

Mengambil data dari BigQuery dan mengubahnya menjadi vektor.

**1. Load Data dari BigQuery** (Mengambil 1000 pertanyaan Stack Overflow)

```python
# load the BQ Table into a Pandas DataFrame
from google.cloud import bigquery

QUESTIONS_SIZE = 1000

bq_client = bigquery.Client(project=PROJECT_ID)
QUERY_TEMPLATE = """
        SELECT distinct q.id, q.title
        FROM (SELECT * FROM `bigquery-public-data.stackoverflow.posts_questions`
        where Score > 0 ORDER BY View_Count desc) AS q
        LIMIT {limit} ;
        """
query = QUERY_TEMPLATE.format(limit=QUESTIONS_SIZE)
query_job = bq_client.query(query)
rows = query_job.result()
df = rows.to_dataframe()

# examine the data
df.head()

```

**2. Init Model Embedding (Gemini)**

```python
# init the vertexai package
from google import genai
from google.genai import types

EMBEDDING_MODEL = "gemini-embedding-001"
client = genai.Client(vertexai=True, project=PROJECT_ID, location=LOCATION)

```

**3. Define Wrapper Function** (Untuk menghindari kena limit quota API)

```python
# get embeddings for a list of texts
BATCH_SIZE = 5

def get_embeddings_wrapper(texts: list[str]) -> list[list[float]]:
    embeddings: list[list[float]] = []
    for i in tqdm.tqdm(range(0, len(texts), BATCH_SIZE)):
        time.sleep(1)  # to avoid the quota error
        response = client.models.embed_content(
            model=EMBEDDING_MODEL, contents=texts[i : i + BATCH_SIZE]
        )
        embeddings = embeddings + [e.values for e in response.embeddings]
    return embeddings

```

**4. Generate Embeddings untuk Dataset**

```python
# get embeddings for the question titles and add them as "embedding" column
df = df.assign(embedding=get_embeddings_wrapper(list(df.title)))
df.head()

```

**5. Cek Similarity (Lokal Test)**

```python
# pick one of them as a key question
key = random.randint(0, len(df))

# calc dot product between the key and other questions
embs = np.array(df.embedding.to_list())
similarities = np.dot(embs[key], embs.T)

# print the question
print(f"Key question: {df.title[key]}\n")

# sort and print the questions by similarities
sorted_questions = sorted(
    zip(df.title, similarities), key=lambda x: x[1], reverse=True
)[:20]
for i, (question, similarity) in enumerate(sorted_questions):
    print(f"{similarity:.4f} {question}")

```

---

### 🔹 Step 3: Setup Vector Search (Index & Endpoint)

Menyiapkan data untuk Vector Search dan membuat Index di Cloud.

**1. Save Embeddings to JSON**

```python
# save id and embedding as a json file
jsonl_string = df[["id", "embedding"]].to_json(orient="records", lines=True)
with open("questions.json", "w") as f:
    f.write(jsonl_string)

# show the first few lines of the json file
! head -n 3 questions.json

```

**2. Upload ke Cloud Storage**

```python
BUCKET_URI = f"gs://{PROJECT_ID}-embvs-tutorial-{UID}"
! gsutil mb -l $LOCATION -p {PROJECT_ID} {BUCKET_URI}
! gsutil cp questions.json {BUCKET_URI}

```

**3. Create Index** (Proses lama ~20 menit)

```python
# create index
from google.cloud import aiplatform

PROJECT_ID = "qwiklabs-gcp-03-bce37a1a4cb6"
LOCATION = "us-east4"
BUCKET_URI = f"gs://{PROJECT_ID}-embvs-tutorial-{UID}"

aiplatform.init(project=PROJECT_ID, location=LOCATION)

my_index = aiplatform.MatchingEngineIndex.create_tree_ah_index(
    display_name="embvs-tutorial-index",
    contents_delta_uri=BUCKET_URI,
    dimensions=3072,
    approximate_neighbors_count=20,
    distance_measure_type="DOT_PRODUCT_DISTANCE",
    leaf_node_embedding_count=100,        # optional
    leaf_nodes_to_search_percent=10       # optional
)

```

**4. Create Index Endpoint**

```python
# create IndexEndpoint
my_index_endpoint = aiplatform.MatchingEngineIndexEndpoint.create(
    display_name=f"embvs-tutorial-index-endpoint-{UID}",
    public_endpoint_enabled=True,
)

```

**5. Deploy Index ke Endpoint**

```python
DEPLOYED_INDEX_ID = f"embvs_tutorial_deployed_{UID}"

# deploy the Index to the Index Endpoint
my_index_endpoint.deploy_index(
    index=my_index,
    deployed_index_id=DEPLOYED_INDEX_ID,
    machine_type="e2-standard-16",
    min_replica_count=1,
    max_replica_count=1  
)

```

---

### 🔹 Step 4: Run Query (Test Search)

Melakukan pencarian "nyata" ke database vektor.

**1. Generate Embedding untuk Kalimat Test**

```python
test_embeddings = get_embeddings_wrapper(["How to read JSON with Python?"])

```

**2. Query & Logging Hasil** (Code ini penting buat Check Progress)

```python
# Test query
response = my_index_endpoint.find_neighbors(
    deployed_index_id=DEPLOYED_INDEX_ID,
    queries=test_embeddings,
    num_neighbors=20,
)

for idx, neighbor in enumerate(response[0]):
    id = np.int64(neighbor.id)
    similar = df.query("id == @id", engine="python")
    print(f"{neighbor.distance:.4f} {similar.title.values[0]}")
    # Do not remove or modify this logging call, it will be used for tracking purposes
    logger.info(f'Task 4. Similar question with the vector search is: {similar.title.values[0]}')

```

---

### 🔹 (Opsional) Utilities: Reconnect to Existing Index

Gunakan code ini jika notebook terputus dan ingin connect ulang ke Index yang sudah jadi.

```python
# Connect ke Index
my_index_id = "[your-index-id]"  # @param {type:"string"}
my_index = aiplatform.MatchingEngineIndex(my_index_id)

# Connect ke Endpoint
my_index_endpoint_id = "[your-index-endpoint-id]"  # @param {type:"string"}
my_index_endpoint = aiplatform.MatchingEngineIndexEndpoint(my_index_endpoint_id)

```

---

### 🐛 3. Troubleshooting & Notes

* **Error Logging (Permission Denied):** Terjadi jika akun Service Account VM tidak punya izin tulis log. Solusi: Login manual di terminal dengan `gcloud auth application-default login`.
* **Dimensionality Mismatch:** Pastikan parameter `dimensions` saat create index sesuai dengan model. Model `gemini-embedding-001` menggunakan dimensi **3072**. Jika pakai model lama (Gecko), gunakan **768**.

---

# ☁️ Lab 2: Create Hybrid Search With Vertex AI Vector Search (GSP1297)

**Rute:** Hybrid Search & RAG Architecture
**Topik:** Vertex AI Vector Search, Sparse Embeddings (TF-IDF), Dense Embeddings, Hybrid Search
**Tanggal Pengerjaan:** 25 Januari 2026

---

## 🎯 1. Overview & Tujuan

Lab ini bertujuan untuk membangun sistem **Hybrid Search** yang menggabungkan dua metode pencarian:

1. **Semantic Search (Dense):** Mencari berdasarkan makna/konteks (menggunakan model embedding AI).
2. **Keyword Search (Sparse):** Mencari berdasarkan kecocokan kata kunci spesifik (menggunakan algoritma TF-IDF).

**Misi Utama:**

1. **Sparse Embedding:** Membuat representasi vektor berbasis kata kunci (TF-IDF) dari dataset Google Merch Shop.
2. **Dense Embedding:** Membuat representasi vektor berbasis makna menggunakan model `text-embedding-005`.
3. **Hybrid Index:** Menggabungkan kedua embedding tersebut ke dalam satu Index Vector Search.
4. **Deploy & Query:** Melakukan query kombinasi untuk mendapatkan hasil pencarian yang lebih akurat dan relevan.

---

## 🛠️ 2. Langkah-Langkah & Full Code

### 🔹 Step 1: Install Packages & Configure Notebook

Persiapan library dan environment variable.

**1. Install Library**

```python
%pip install --upgrade --quiet --user google-cloud-aiplatform google-cloud-storage

```

**2. Restart Kernel** (Wajib)

```python
# Restart kernel after installs of that your environment can access the new packages
import IPython

app = IPython.Application.instance()
app.kernel.do_shutdown(True)

```

**3. Set Project ID & Region**
*(Note: Project ID disesuaikan dengan lab instruction: `qwiklabs-gcp-02-69bfa4ad4857`)*

```python
# get project ID and set your region.
PROJECT_ID = ! gcloud config get project
PROJECT_ID = PROJECT_ID[0]
LOCATION = "us-central1" # Sesuai instruksi lab
print(f"Project ID: {PROJECT_ID}")
print(f"Region: {LOCATION}")

if PROJECT_ID == "(unset)":
    print(f"Please set the project ID manually below")

```

**4. Generate UID**

```python
# generate an unique id for this session
from datetime import datetime

UID = datetime.now().strftime("%m%d%H%M")

```

---

### 🔹 Step 2: Prepare Dataset

Mengambil dataset produk Google Merch Shop.

```python
import pandas as pd

CSV_URL = "https://storage.googleapis.com/spls/gsp1297/google_merch_shop_items.csv"

# Load the CSV file into a DataFrame
df = pd.read_csv(CSV_URL)
df["title"]

```

---

### 🔹 Step 3: Create Sparse Embeddings (Keyword Search)

Bagian ini membuat "peta" kata kunci menggunakan TF-IDF.

**1. Train TF-IDF Vectorizer**

```python
from sklearn.feature_extraction.text import TfidfVectorizer

# Sample Text Data
corpus = df.title.tolist()

# Initialize TfidfVectorizer
vectorizer = TfidfVectorizer()

# Fit and Transform
vectorizer.fit_transform(corpus)

```

**2. Define Wrapper Function (Sparse)**

```python
def get_sparse_embedding(text):
    # Transform Text into TF-IDF Sparse Vector
    tfidf_vector = vectorizer.transform([text])

    # Create Sparse Embedding for the New Text
    values = []
    dims = []
    for i, tfidf_value in enumerate(tfidf_vector.data):
        values.append(float(tfidf_value))
        dims.append(int(tfidf_vector.indices[i]))
    return {"values": values, "dimensions": dims}

```

**3. Test Sparse Embedding**

```python
text_text = "Chrome Dino Pin"
get_sparse_embedding(text_text)

```

**4. Generate Sparse Data for All Items**

```python
# set BUCKET_URI
BUCKET_URI = f"gs://{PROJECT_ID}"

items = []
for i in range(len(df)):
    id = i
    title = df.title[i]
    sparse_embedding = get_sparse_embedding(title)
    items.append({"id": id, "title": title, "sparse_embedding": sparse_embedding})
items[:5]

```

**5. Save JSON & Upload to GCS**

```python
# output as a JSONL file and save to the GCS bucket
with open("items.json", "w") as f:
    for item in items:
        f.write(f"{item}\n")
! gsutil cp items.json $BUCKET_URI

```

---

### 🔹 Step 4: Setup Index Endpoint

Membuat "rumah" (Endpoint) untuk index kita nanti.

```python
# init the aiplatform package
from google.cloud import aiplatform

aiplatform.init(project=PROJECT_ID, location=LOCATION)

# create `IndexEndpoint`
my_index_endpoint = aiplatform.MatchingEngineIndexEndpoint.create(
    display_name=f"vs-hybridsearch-index-endpoint-{UID}", public_endpoint_enabled=True
)

```

---

### 🔹 Step 5: Create Hybrid Index (Dense + Sparse) & Deploy

Ini langkah paling penting: menggabungkan kecerdasan AI (Dense) dengan TF-IDF (Sparse).

**1. Load Text Embedding Model**

```python
# get text embedding model
from vertexai.preview.language_models import TextEmbeddingModel

model = TextEmbeddingModel.from_pretrained("text-embedding-005")

# wrapper
def get_dense_embedding(text):
    return model.get_embeddings([text])[0].values

# test it
get_dense_embedding("Chrome Dino Pin")

```

**2. Combine Dense & Sparse Embeddings**

```python
items = []
for i in range(len(df)):
    id = i
    title = df.title[i]
    dense_embedding = get_dense_embedding(title)
    sparse_embedding = get_sparse_embedding(title)
    items.append(
        {
            "id": id,
            "title": title,
            "embedding": dense_embedding,
            "sparse_embedding": sparse_embedding,
        }
    )
items[0]

```

**3. Save Hybrid Data & Upload**

```python
# output as a JSONL file and save to the GCS bucket
with open("items.json", "w") as f:
    for item in items:
        f.write(f"{item}\n")
! gsutil cp items.json $BUCKET_URI

```

**4. Create Hybrid Index (Using GAPIC Client)**
*Note: Lab ini menggunakan client level rendah (GAPIC) untuk konfigurasi advanced hybrid index.*

```python
from google.cloud import aiplatform_v1

# 1. Initialize the low-level GAPIC client
api_endpoint = f"{LOCATION}-aiplatform.googleapis.com"
client = aiplatform_v1.IndexServiceClient(client_options={"api_endpoint": api_endpoint})

# 2. Manually define the Index configuration (fixing the missing algorithmConfig)
index_request = {
    "display_name": f"vs-hybridsearch-index-{UID}",
    "metadata": {
        "contentsDeltaUri": BUCKET_URI,
        "config": {
            "dimensions": 768,
            "approximateNeighborsCount": 10,
            "distanceMeasureType": "DOT_PRODUCT_DISTANCE",
            "algorithmConfig": {
                "treeAhConfig": {
                    "leafNodeEmbeddingCount": 1000,
                    "leafNodesToSearchPercent": 10
                }
            }
        }
    }
}

# 3. Send the creation request
parent = f"projects/{PROJECT_ID}/locations/{LOCATION}"
print("Submitting Index creation request via GAPIC...")
operation = client.create_index(parent=parent, index=index_request)

# 4. Wait for the operation to complete (this can take 5-10 minutes)
print("Waiting for Index creation...")
response = operation.result(timeout=1800) 

# 5. Re-wrap the result in the high-level SDK object for the rest of the notebook
my_hybrid_index = aiplatform.MatchingEngineIndex(index_name=response.name)

print(f"Successfully created Index: {my_hybrid_index.resource_name}")

```

**5. Deploy Index** (Proses ini memakan waktu ~30 menit)

```python
# deploy index
DEPLOYED_HYBRID_INDEX_ID = f"vs_hybridsearch_deployed_{UID}"
my_index_endpoint.deploy_index(
    index=my_hybrid_index, 
    deployed_index_id=DEPLOYED_HYBRID_INDEX_ID,
    machine_type="e2-standard-16",
    min_replica_count=1,
    max_replica_count=1
)

```

---

### 🔹 Step 6: Run Hybrid Query

Melakukan pencarian dengan parameter hybrid (RRF Ranking Alpha).

**1. Create HybridQuery Object**

```python
from google.cloud.aiplatform.matching_engine.matching_engine_index_endpoint import (
    HybridQuery,
)

# create HybridQuery
query_text = "Kids"
query_dense_emb = get_dense_embedding(query_text)
query_sparse_emb = get_sparse_embedding(query_text)
query = HybridQuery(
    dense_embedding=query_dense_emb,
    sparse_embedding_dimensions=query_sparse_emb["dimensions"],
    sparse_embedding_values=query_sparse_emb["values"],
    rrf_ranking_alpha=0.5,
)

```

**2. Execute Query & Print Results**

```python
# run a hybrid query
response = my_index_endpoint.find_neighbors(
    deployed_index_id=DEPLOYED_HYBRID_INDEX_ID,
    queries=[query],
    num_neighbors=10,
)

# print results
for idx, neighbor in enumerate(response[0]):
    title = df.title[int(neighbor.id)]
    dense_dist = neighbor.distance if neighbor.distance else 0.0
    sparse_dist = neighbor.sparse_distance if neighbor.sparse_distance else 0.0
    print(f"{title:<40}: dense_dist: {dense_dist:.3f}, sparse_dist: {sparse_dist:.3f}")

```

---

### 🔹 Step 7: Clean Up (Optional)

Menghapus resource agar tidak memakan biaya (hanya jika menggunakan akun pribadi).

```python
# wait for a confirmation
input("Press Enter to delete Index Endpoint, Index and Cloud Storage bucket:")

# delete Index Endpoint
my_index_endpoint.undeploy_all()
my_index_endpoint.delete(force=True)

# delete Indexes
my_hybrid_index.delete()

# delete Cloud Storage bucket
! gsutil rm -r "{BUCKET_URI}"

```

---

### 🐛 3. Troubleshooting / Masalah yang Dihadapi

* **Deployment Lama:** Proses `my_index_endpoint.deploy_index` membutuhkan waktu sekitar 30-40 menit. Jangan close tab JupyterLab agar koneksi tidak putus.
* **Permissions:** Jika menggunakan project sendiri (bukan Qwiklabs), pastikan API `aiplatform` dan `storage` sudah di-enable dan Service Account memiliki role `Vertex AI User` dan `Storage Object Admin`.
* **Kernel Crash:** Jika kernel restart tiba-tiba, jalankan ulang cell import library dan konfigurasi variable (`PROJECT_ID`, `UID`) sebelum melanjutkan ke cell deployment/query.

### 📝 4. Catatan Penting (Key Takeaways)

1. **Hybrid Search Power:** Menggabungkan kekuatan pencarian kata kunci (presisi pada istilah spesifik) dan pencarian semantik (pemahaman konteks).
2. **RRF (Reciprocal Rank Fusion):** Algoritma yang digunakan Google untuk menggabungkan ranking hasil dari Dense Search dan Sparse Search. Parameter `rrf_ranking_alpha` (0.5) menentukan bobot keseimbangan kedua metode tersebut.
3. **Struktur Data:** Hybrid Index membutuhkan file JSONL dimana setiap item memiliki field `embedding` (array float untuk dense) dan `sparse_embedding` (object berisi values & dimensions untuk sparse).

---

### 📂 Daftar Lampiran

* [Materi](./Materi/README.md)
* [Notebook/intro-textemb-vectorsearch.ipynb](./Notebook/intro-textemb-vectorsearch.ipynb)
* [Notebook/hybrid-search-v1.0.0.ipynb](./Notebook/hybrid-search-v1.0.0.ipynb)

---

[⬅️ Kembali ke Menu Utama](../README.md)