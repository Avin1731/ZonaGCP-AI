# ☁️ Automate Data Capture at Scale with Document AI: Challenge Lab

**Rute:** Machine Learning & Data Engineering
**Topik:** Document AI, Cloud Functions Gen 2, Cloud Storage, BigQuery, Python
**Tanggal Pengerjaan:** 29 Januari 2026

---

## 🎯 1. Overview & Tujuan

Lab ini bertujuan untuk membangun pipeline pemrosesan dokumen otomatis (Invoice) menggunakan Google Cloud. Sistem akan membaca file PDF yang diupload ke Cloud Storage, mengekstrak datanya menggunakan Document AI (Form Parser), dan menyimpannya ke BigQuery.

**Misi Utama:**

1. Mengaktifkan API dan menyiapkan source code function.
2. Membuat Document AI Form Processor.
3. Menyiapkan infrastruktur (Bucket Input/Output/Archive & BigQuery).
4. Deploy Cloud Function (Gen 2) dengan konfigurasi Environment Variable dan IAM yang ketat.
5. Validasi pipeline dengan mengupload sampel invoice.

---

## 🛠️ 2. Langkah-Langkah & Solusi

### 🔹 Task 1: Enable API & Copy Resources

* **Perintah CLI:**
Mengaktifkan Document AI API dan mengambil source code starter pack.

```bash
gcloud services enable documentai.googleapis.com

mkdir ./document-ai-challenge
gsutil -m cp -r gs://spls/gsp367/* ~/document-ai-challenge/

```

### 🔹 Task 2: Create Form Processor

* **Konfigurasi GUI:**
* Menu: `Document AI > Processors > Create Processor`
* Type: `Form Parser` (General)
* Name: `form-processor`
* Region: `US` (Penting! Harus sesuai dengan config nanti)
* **Note:** Simpan **Processor ID** untuk Task 4.



### 🔹 Task 3: Create Google Cloud Resources

* **Perintah CLI:**
Membuat 3 Bucket (Input, Output, Archive) dan Dataset BigQuery beserta Tabelnya.

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=europe-west4

# 1. Buat Buckets
gsutil mb -c standard -l $REGION -b on gs://${PROJECT_ID}-input-invoices
gsutil mb -c standard -l $REGION -b on gs://${PROJECT_ID}-output-invoices
gsutil mb -c standard -l $REGION -b on gs://${PROJECT_ID}-archived-invoices

# 2. Buat BigQuery Dataset & Table
bq mk -d --location=US invoice_parser_results
bq mk --table invoice_parser_results.doc_ai_extracted_entities \
~/document-ai-challenge/scripts/table-schema/doc_ai_extracted_entities.json

```

### 🔹 Task 4: Deploy Cloud Function (The Hard Part)

Bagian ini paling krusial. Kita harus memperbaiki file config dan requirements sebelum deploy.

**Langkah 1: Perbaiki `requirements.txt` & `.env.yaml**`

```bash
cd ~/document-ai-challenge/scripts

# Fix Requirements (Pastikan library lengkap)
cat > cloud-functions/process-invoices/requirements.txt <<EOF
google-cloud-documentai==2.16.0
google-cloud-bigquery
google-cloud-storage
functions-framework
EOF

# Fix Environment Variables (PROCESSOR_ID dari Task 2)
# Penting: TIMEOUT harus String ("200") agar tidak error TypeError
export PROCESSOR_ID=8e8ccfdc2055013d  # Ganti dengan ID Processor kamu

cat > cloud-functions/process-invoices/.env.yaml <<EOF
PROCESSOR_ID: $PROCESSOR_ID
PARSER_LOCATION: us
PROJECT_ID: $PROJECT_ID
TIMEOUT: "200"
GCS_OUTPUT_URI_PREFIX: doc_ai_results
EOF

```

**Langkah 2: Deploy Function (Gen 2)**
Menggunakan Service Account **App Engine** (`@appspot`) agar checker valid, dan membuka akses Ingress.

```bash
gcloud functions deploy process-invoices \
  --gen2 \
  --region=${REGION} \
  --entry-point=process_invoice \
  --runtime=python313 \
  --service-account=${PROJECT_ID}@appspot.gserviceaccount.com \
  --source=cloud-functions/process-invoices \
  --timeout=400 \
  --env-vars-file=cloud-functions/process-invoices/.env.yaml \
  --trigger-resource=gs://${PROJECT_ID}-input-invoices \
  --trigger-event=google.storage.object.finalize \
  --allow-unauthenticated \
  --ingress-settings=all

```

**Langkah 3: Pastikan Public Access (Permission IAM)**
Karena Gen 2 berbasis Cloud Run, permission `invoker` harus ditembak ke Cloud Run service.

```bash
gcloud run services add-iam-policy-binding process-invoices \
  --region=${REGION} \
  --member=allUsers \
  --role=roles/run.invoker

```

### 🔹 Task 5: Test Pipeline

Mengupload invoice untuk memicu trigger.

```bash
gsutil cp ~/document-ai-challenge/invoices/* gs://${PROJECT_ID}-input-invoices/

```

*Cek hasil di BigQuery -> Preview Table.*

---

### 🐛 3. Troubleshooting / Masalah yang Dihadapi

**Error 1: `TypeError: int() argument must be a string**`

* **Penyebab:** Di file `.env.yaml`, variabel `TIMEOUT: 200` ditulis sebagai integer (angka). Cloud Function Python mengharapkan semua environment variable berupa string.
* **Solusi:** Ubah menjadi string dengan tanda kutip: `TIMEOUT: "200"`.

**Error 2: `Container Healthcheck failed` / Crash saat Startup**

* **Penyebab:** Library di `requirements.txt` bawaan lab kurang lengkap atau versinya bentrok.
* **Solusi:** Re-create `requirements.txt` dan definisikan versi library secara eksplisit (seperti `google-cloud-documentai==2.16.0`).

**Error 3: "Please deploy a Gopher HTTP..." (Task 4 Merah Terus)**

* **Penyebab:** Bot Checker (Gopher) tidak bisa mengakses URL Cloud Function karena terblokir firewall atau autentikasi.
1. **Authentication:** Masih `Require authentication`.
2. **Ingress:** Masih `Internal traffic only`.


* **Solusi:**
* Set Authentication ke **Allow unauthenticated invocations**.
* Set Ingress Control ke **Allow all traffic**.
* Jalankan command `gcloud run services add-iam-policy-binding ... --role=roles/run.invoker`.



**Error 4: `500 Internal Server Error` saat test `curl**`

* **Penyebab:** Payload JSON saat testing `curl` kurang lengkap (tidak ada `contentType`), atau konfigurasi region Processor (`PARSER_LOCATION`) tidak sesuai dengan lokasi asli Processor di Document AI console.
* **Solusi:** Pastikan `PARSER_LOCATION: us` (jika processor dibuat di US) dan gunakan payload curl yang lengkap saat testing manual.

---

### 📝 4. Catatan Penting (Key Takeaways)

1. **Cloud Functions Gen 2 = Cloud Run:** Saat berurusan dengan permission (IAM) atau networking (Ingress) di Gen 2, seringkali kita harus mengubah settingannya di level **Cloud Run Service**, bukan hanya di Cloud Functions command.
2. **Service Account Matters:** Instruksi lab kadang membingungkan antara Compute Engine SA dan App Engine SA. Jika checker gagal mendeteksi, cobalah ganti Service Account yang digunakan (dalam kasus ini, `@appspot` yang berhasil).
3. **Strict YAML Formatting:** Python environment variables via YAML sangat sensitif terhadap tipe data. Selalu gunakan tanda kutip untuk angka.
4. **Ingress Settings:** Secara default, deploy function mungkin menutup akses public ("Internal Only"). Flag `--ingress-settings=all` sangat penting untuk lab yang checkernya berasal dari luar project.

---

[⬅️ Kembali ke Menu Utama](../README.md)