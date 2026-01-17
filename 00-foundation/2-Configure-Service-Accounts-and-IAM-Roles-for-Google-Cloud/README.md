# ☁️ Configure Service Accounts and IAM for Google Cloud: Challenge Lab

**Rute:** Rute 0 — Foundation
**Topik:** IAM, Service Accounts, Custom Roles, BigQuery Client Library
**Tanggal Pengerjaan:** 18 Januari 2026

---

## 🎯 1. Overview & Tujuan

Lab ini bertujuan untuk memahami cara membuat identitas non-manusia ("Robot" atau Service Account), memberikan izin spesifik (IAM Roles), dan menempelkan identitas tersebut ke Virtual Machine agar bisa mengakses layanan Google Cloud lain (seperti BigQuery) secara aman tanpa menyimpan password.

**Misi Utama:**

1. Membuat Service Account & Memberikan Izin (IAM).
2. Membuat VM yang ditempeli Service Account tersebut.
3. Membuat Custom Role (Izin kustom) via file YAML.
4. Mengakses dataset BigQuery menggunakan script Python dari dalam VM.

---

## 🛠️ 2. Langkah-Langkah & Solusi

### 🔹 Persiapan Awal (Variable Setup)

*(Jalankan ini di Cloud Shell atau SSH `lab-vm` di awal agar command selanjutnya berjalan lancar)*

```bash
# Sesuaikan Zone dengan lab kamu
export ZONE=us-east4-a
export PROJECT_ID=$(gcloud config get-value project)

```

### 🔹 Task 1: Enable Gemini (Optional)

* **Deskripsi:** Bagian ini opsional dan hanya untuk eksplorasi AI. Kita bisa skip langsung ke teknis.

### 🔹 Task 2: Create a service account using the gcloud CLI

* **Perintah CLI:**
*(Masuk ke SSH `lab-vm` terlebih dahulu, login, lalu buat Service Account `devops`)*

```bash
# 1. Masuk ke VM Lab
gcloud compute ssh lab-vm --zone=$ZONE

# 2. Login Authenticate (Ikuti instruksi link di layar)
gcloud auth login --no-launch-browser

# 3. Buat Service Account
gcloud iam service-accounts create devops --display-name "devops"

```

### 🔹 Task 3: Grant IAM permissions to a service account

* **Perintah CLI:**
*(Memberikan izin `Instance Admin` dan `Service Account User` ke akun `devops`)*

```bash
# 1. Simpan variabel otomatis
export PROJECT_ID=$(gcloud config get-value project)
export SA=devops@$PROJECT_ID.iam.gserviceaccount.com

# 2. Beri izin: Service Account User
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:$SA" \
    --role="roles/iam.serviceAccountUser"

# 3. Beri izin: Compute Instance Admin
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:$SA" \
    --role="roles/compute.instanceAdmin.v1"

```

### 🔹 Task 4: Create a compute instance with a service account attached

* **Perintah CLI:**
*(Membuat VM baru bernama `vm-2` yang menggunakan identitas `devops`)*

```bash
gcloud compute instances create vm-2 \
    --zone=$ZONE \
    --service-account=$SA \
    --scopes=https://www.googleapis.com/auth/cloud-platform

```

### 🔹 Task 5: Create a custom role using a YAML file

* **Perintah CLI:**
*(Membuat role khusus yang hanya bisa connect & get CloudSQL)*

```bash
# 1. Buat file konfigurasi YAML
cat > role-definition.yaml <<EOF
title: "My Custom Role"
description: "Custom role for Cloud SQL"
stage: "GA"
includedPermissions:
- cloudsql.instances.connect
- cloudsql.instances.get
EOF

# 2. Upload role ke Google Cloud
gcloud iam roles create myCustomRole \
    --project=$PROJECT_ID \
    --file=role-definition.yaml

```

### 🔹 Task 6: Use the client libraries to access BigQuery

Tugas ini dibagi menjadi 2 Fase: Membuat Infrastruktur & Menjalankan Script.

**Fase 1: Setup Infrastructure (Di terminal `lab-vm` yang sekarang)**

```bash
# 1. Simpan variabel Service Account baru
export BQ_SA_EMAIL=bigquery-qwiklab@$PROJECT_ID.iam.gserviceaccount.com

# 2. Buat Service Account "bigquery-qwiklab"
gcloud iam service-accounts create bigquery-qwiklab \
    --display-name "bigquery-qwiklab"

# 3. Beri Izin BigQuery (Viewer & User)
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:$BQ_SA_EMAIL" \
    --role="roles/bigquery.dataViewer"

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:$BQ_SA_EMAIL" \
    --role="roles/bigquery.user"

# 4. Buat VM "bigquery-instance" dengan SA tersebut
gcloud compute instances create bigquery-instance \
    --zone=$ZONE \
    --service-account=$BQ_SA_EMAIL \
    --scopes=https://www.googleapis.com/auth/cloud-platform

```

**Fase 2: Script Execution (Pindah ke VM Baru)**
*(Jalankan perintah ini satu per satu)*

```bash
# 1. Keluar dari lab-vm
exit

# 2. Masuk ke bigquery-instance
export ZONE=us-east4-a
gcloud compute ssh bigquery-instance --zone=$ZONE

# 3. Install Dependencies Python (Di dalam bigquery-instance)
sudo apt update
sudo apt install python3-pip -y
pip3 install google-cloud-bigquery pandas db-dtypes --break-system-packages

# 4. Buat Script Python (Otomatis isi credential)
export PROJECT_ID=$(gcloud config get-value project)
export BQ_SA_EMAIL=bigquery-qwiklab@$PROJECT_ID.iam.gserviceaccount.com

cat > query.py <<EOF
from google.auth import compute_engine
from google.cloud import bigquery

credentials = compute_engine.Credentials(
    service_account_email='$BQ_SA_EMAIL')

query = '''
SELECT name, SUM(number) as total_people
FROM \`bigquery-public-data.usa_names.usa_1910_2013\`
WHERE state = 'TX'
GROUP BY name, state
ORDER BY total_people DESC
LIMIT 20
'''

client = bigquery.Client(
    project='$PROJECT_ID',
    credentials=credentials)

print(client.query(query).to_dataframe())
EOF

# 5. Jalankan Script
python3 query.py

```

---

## 🐛 3. Troubleshooting / Masalah yang Dihadapi

* **Error:** `ERROR: (gcloud.compute.instances.create) could not parse resource []`
* **Penyebab:** Variabel `$ZONE` atau `$SA` kosong karena sesi terminal terputus atau lupa di-export ulang.
* **Solusi:**
> Jalankan ulang perintah `export` variable sebelum menjalankan perintah pembuatan VM.


* **Error:** `Permission denied (publickey)` saat SSH.
* **Penyebab:** Mencoba SSH ke diri sendiri atau kunci SSH belum ter-generate dengan benar.
* **Solusi:**
> Pastikan keluar (`exit`) dulu ke Cloud Shell utama sebelum pindah dari `lab-vm` ke `bigquery-instance`.



---

## 📝 4. Catatan Penting (Key Takeaways)

Hal-hal baru yang saya pelajari dari lab ini:

* **Service Account (SA):** Adalah identitas untuk aplikasi/VM, bukan untuk manusia. Sangat penting untuk keamanan automation.
* **Least Privilege:** Dengan Custom Role, kita bisa membatasi izin robot hanya untuk hal spesifik (misal: cuma bisa connect CloudSQL, gak bisa hapus database).
* **Client Library:** Saat menggunakan Python di dalam VM yang sudah ditempel SA, kita tidak perlu login manual/simpan password. Library `google.auth` otomatis mendeteksi identitas VM tersebut.

---

[⬅️ Kembali ke Menu Utama](https://www.google.com/search?q=../../README.md)