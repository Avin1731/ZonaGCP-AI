# ☁️ Cloud Run Functions: 3 Ways: Challenge Lab

**Rute:** Skill Badge - Cloud Run Functions
**Topik:** Serverless, Cloud Functions Gen 2, Eventarc, Cloud Storage
**Tanggal Pengerjaan:** 22 Januari 2026

---

## 🎯 1. Overview & Tujuan

Lab ini bertujuan untuk menguji kemampuan dalam membuat dan men-deploy Google Cloud Functions (Generasi ke-2) yang berjalan di atas Cloud Run. Kita diminta membuat fungsi yang berjalan otomatis saat ada file di-upload ke Storage, dan fungsi yang merespon request HTTP.

**Misi Utama:**

1. Membuat Cloud Storage Bucket untuk menampung file.
2. Membuat fungsi `cs-logger` (Triggered by Storage) untuk mencatat log aktivitas bucket.
3. Membuat fungsi `http-responder` (Triggered by HTTP) dengan konfigurasi scaling instance khusus.

---

## 🛠️ 2. Langkah-Langkah & Solusi

### 🔹 Persiapan Awal (Wajib)

Sebelum masuk ke Task 1, setup environment variable dan **enable API** agar tidak error di tengah jalan.

* **Perintah CLI:**

```bash
# Setup Variabel
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-east1
export BUCKET_NAME=$PROJECT_ID  # Nama bucket disamakan dengan Project ID sesuai soal

# Enable API (Penting! Sering bikin gagal kalau lupa)
gcloud services enable \
  cloudfunctions.googleapis.com \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  eventarc.googleapis.com

```

### 🔹 Task 1: Create a Cloud Storage bucket

* **Deskripsi:** Membuat bucket penyimpanan di region `us-east1`.
* **Perintah CLI:**

```bash
gcloud storage buckets create gs://$BUCKET_NAME --location=$REGION

```

### 🔹 Task 2: Create, deploy, and test a Cloud Storage function

* **Deskripsi:** Deploy fungsi Gen 2 yang akan jalan kalau ada file baru di bucket.
* **Langkah Penting (Anti Error Permission):**
Sebelum deploy, kita harus memberi izin ke Service Account Storage agar bisa kirim event ke Eventarc.
```bash
# Ambil Project Number & Service Account Storage
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
STORAGE_SERVICE_ACCOUNT=$(gsutil kms serviceaccount -p $PROJECT_NUMBER)

# Tambah Role Pub/Sub Publisher
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member serviceAccount:$STORAGE_SERVICE_ACCOUNT \
  --role roles/pubsub.publisher

```


* **Script Deploy:**
```bash
# 1. Buat folder & file
mkdir ~/cs-logger && cd ~/cs-logger

cat > index.js <<EOF
const functions = require('@google-cloud/functions-framework');
functions.cloudEvent('cs-logger', (cloudevent) => {
  console.log('A new event in your Cloud Storage bucket has been logged!');
  console.log(cloudevent);
});
EOF

cat > package.json <<EOF
{
  "name": "nodejs-functions-gen2-codelab",
  "version": "0.0.1",
  "main": "index.js",
  "dependencies": {
    "@google-cloud/functions-framework": "^2.0.0"
  }
}
EOF

# 2. Deploy Function
gcloud functions deploy cs-logger \
  --gen2 \
  --runtime nodejs20 \
  --entry-point cs-logger \
  --source . \
  --region $REGION \
  --trigger-bucket $BUCKET_NAME \
  --trigger-location $REGION \
  --max-instances 2 \
  --quiet

```


> **⚠️ NOTED:** Di soal diminta `max-instances 2`, tapi kadang saat dicek di Console hasilnya malah `max-instances 5` atau default lainnya. **Abaikan saja**, sistem Qwiklabs tetap mendeteksi ini sebagai **Complete/Checklist**. Jangan pusing di sini.



### 🔹 Task 3: Create and deploy a HTTP function

* **Deskripsi:** Deploy fungsi HTTP dengan minimal 1 instance (biar tidak *cold start*).
* **Perintah CLI:**

```bash
# 1. Buat folder baru
cd ~
mkdir ~/http-responder && cd ~/http-responder

# 2. Buat index.js
cat > index.js <<EOF
const functions = require('@google-cloud/functions-framework');
functions.http('http-responder', (req, res) => {
  res.status(200).send('HTTP function (2nd gen) has been called!');
});
EOF

# 3. Buat package.json
cat > package.json <<EOF
{
  "name": "nodejs-functions-gen2-codelab",
  "version": "0.0.1",
  "main": "index.js",
  "dependencies": {
    "@google-cloud/functions-framework": "^2.0.0"
  }
}
EOF

# 4. Deploy Function
gcloud functions deploy http-responder \
  --gen2 \
  --runtime nodejs20 \
  --entry-point http-responder \
  --source . \
  --region $REGION \
  --trigger-http \
  --allow-unauthenticated \
  --min-instances 1 \
  --max-instances 2 \
  --quiet

```

---

## 🐛 3. Troubleshooting / Masalah yang Dihadapi

**Error:**
`Permission denied while using the Eventarc Service Agent.`

**Penyebab:**
Cloud Functions Gen 2 menggunakan Eventarc di belakang layar. Secara default di lab ini, Service Account Cloud Storage tidak punya izin untuk mem-publish event ke Eventarc (via Pub/Sub).

**Solusi:**
Menambahkan IAM binding `roles/pubsub.publisher` secara manual ke Service Account Storage (lihat langkah di Task 2 bagian "Langkah Penting"). Setelah command IAM dijalankan, **tunggu 1-2 menit** sebelum deploy ulang agar permission terpropagasi.

---

## 📝 4. Catatan Penting (Key Takeaways)

Hal-hal baru yang saya pelajari dari lab ini:

1. **Eventarc & Permissions:** Gen 2 Functions jauh lebih ketat soal permission dibanding Gen 1. Service account trigger (seperti Storage) harus eksplisit diberi izin Pub/Sub.
2. **Delay Propagasi IAM:** Setelah mengubah IAM/Permission, command deploy tidak bisa langsung sukses. Harus sabar menunggu 1-2 menit.
3. **Anomaly Max Instances:** Pada Task 2, meskipun kita set flag `--max-instances 2`, kadang GUI menampilkan angka berbeda (misal 5 atau 10). Selama command tidak error, progress check tetap valid (hijau).

---

[⬅️ Kembali ke Menu Utama](../README.md)