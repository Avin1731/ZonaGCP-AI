# ☁️ Use APIs to Work with Cloud Storage: Challenge Lab

**Rute:** Cloud Engineering / API Fundamentals
**Topik:** Cloud Storage, JSON API, REST API, curl
**Tanggal Pengerjaan:** 19 Januari 2026

---

## 🎯 1. Overview & Tujuan

Lab ini bertujuan untuk memahami cara berinteraksi dengan Google Cloud Storage (GCS) secara manual menggunakan **JSON/REST API** dan perintah `curl`, tanpa menggunakan `gcloud storage` atau Console GUI. Ini melatih pemahaman tentang struktur request HTTP, Header Authorization, dan payload JSON.

**Misi Utama:**

1. Membuat dua buah Bucket menggunakan API.
2. Mengupload file image ke Bucket 1 menggunakan API.
3. Menyalin (Copy/Rewrite) file dari Bucket 1 ke Bucket 2 menggunakan API.
4. Mengubah akses file menjadi Public (ACL) menggunakan API.
5. Menghapus file dan Bucket menggunakan API.

---

## 🛠️ 2. Langkah-Langkah & Solusi

### 🔹 Persiapan (Setup Environment)

Sebelum masuk ke Task, simpan variabel agar command lebih rapi dan token otentikasi tidak expired.

* **Perintah CLI / Script:**

```bash
# Set Project ID dan Token OAuth
export PROJECT_ID=$(gcloud config get-value project)
export OAUTH_TOKEN=$(gcloud auth print-access-token)

# Set Nama Bucket (Sesuaikan dengan yang diminta lab)
export BUCKET_1=qwiklabs-gcp-04-7233fc0e946c-bucket-1
export BUCKET_2=qwiklabs-gcp-04-7233fc0e946c-bucket-2
export OBJECT_NAME=demo-image.png

```

### 🔹 Task 1: Create Two Buckets

* **Deskripsi:** Membuat dua bucket baru dengan storage class `multi_regional` di lokasi `us` menggunakan payload JSON.
* **Perintah CLI / Script:**

```bash
# 1. Buat Bucket Pertama
cat > bucket_1.json <<EOF
{
   "name": "$BUCKET_1",
   "location": "us",
   "storageClass": "multi_regional"
}
EOF

curl -X POST -H "Authorization: Bearer $OAUTH_TOKEN" \
     -H "Content-Type: application/json" \
     --data-binary @bucket_1.json \
     "https://storage.googleapis.com/storage/v1/b?project=$PROJECT_ID"

# 2. Buat Bucket Kedua
cat > bucket_2.json <<EOF
{
   "name": "$BUCKET_2",
   "location": "us",
   "storageClass": "multi_regional"
}
EOF

curl -X POST -H "Authorization: Bearer $OAUTH_TOKEN" \
     -H "Content-Type: application/json" \
     --data-binary @bucket_2.json \
     "https://storage.googleapis.com/storage/v1/b?project=$PROJECT_ID"

```

### 🔹 Task 2: Upload Object

* **Deskripsi:** Mengunduh gambar dummy ke Cloud Shell lalu menguploadnya ke Bucket 1 sebagai media.
* **Perintah CLI / Script:**

```bash
# Download gambar contoh
curl -o $OBJECT_NAME https://upload.wikimedia.org/wikipedia/commons/thumb/a/a4/Gnome-face-smile.svg/48px-Gnome-face-smile.svg.png

# Upload ke Bucket 1
curl -X POST -H "Authorization: Bearer $OAUTH_TOKEN" \
     -H "Content-Type: image/png" \
     --data-binary @$OBJECT_NAME \
     "https://storage.googleapis.com/upload/storage/v1/b/$BUCKET_1/o?uploadType=media&name=$OBJECT_NAME"

```

### 🔹 Task 3: Copy Object

* **Deskripsi:** Menyalin file dari Bucket 1 ke Bucket 2 menggunakan endpoint `rewriteTo`.
* **Perintah CLI / Script:**

```bash
curl -X POST -H "Authorization: Bearer $OAUTH_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{}' \
     "https://storage.googleapis.com/storage/v1/b/$BUCKET_1/o/$OBJECT_NAME/rewriteTo/b/$BUCKET_2/o/$OBJECT_NAME"

```

### 🔹 Task 4: Make Object Public (ACL)

* **Deskripsi:** Mengatur Access Control List (ACL) agar file bisa diakses oleh `allUsers` (Public Reader).
* **Perintah CLI / Script:**

```bash
# Buat konfigurasi JSON untuk ACL
cat > acl.json <<EOF
{
  "entity": "allUsers",
  "role": "READER"
}
EOF

# Terapkan ke file di Bucket 1
curl -X POST -H "Authorization: Bearer $OAUTH_TOKEN" \
     -H "Content-Type: application/json" \
     --data-binary @acl.json \
     "https://storage.googleapis.com/storage/v1/b/$BUCKET_1/o/$OBJECT_NAME/acl"

```

### 🔹 Task 5: Cleanup (Delete Object & Bucket)

* **Deskripsi:** Menghapus file di Bucket 1 terlebih dahulu, kemudian menghapus Bucket 1.
* **Perintah CLI / Script:**

```bash
# 1. Hapus File dulu (Wajib, bucket tidak bisa dihapus kalau ada isinya)
curl -X DELETE -H "Authorization: Bearer $OAUTH_TOKEN" \
     "https://storage.googleapis.com/storage/v1/b/$BUCKET_1/o/$OBJECT_NAME"

# 2. Hapus Bucket 1
curl -X DELETE -H "Authorization: Bearer $OAUTH_TOKEN" \
     "https://storage.googleapis.com/storage/v1/b/$BUCKET_1"

```

---

## 🐛 3. Troubleshooting / Masalah yang Dihadapi

**Error:**

```json
{
  "error": {
    "code": 404,
    "message": "The specified bucket does not exist.",
    "reason": "notFound"
  }
}

```

**Penyebab:**
Terjadi saat mencoba menghapus bucket di Task 5. Ternyata bucket tersebut sudah terhapus (mungkin karena perintah sebelumnya dijalankan dua kali atau ada lag di Cloud Shell).

**Solusi:**
Tidak perlu panik. Error `404 Not Found` pada perintah DELETE artinya **objek tujuan sudah tidak ada**. Secara teknis, tugas "Menghapus Bucket" sudah tercapai. Langsung klik *Check My Progress*.

---

## 📝 4. Catatan Penting (Key Takeaways)

Hal-hal baru yang saya pelajari dari lab ini:

1. **Oentikasi Manual:** Setiap request API butuh header `Authorization: Bearer <token>`.
2. **Struktur Endpoint:** URL Endpoint GCS API memiliki pola `storage.googleapis.com/storage/v1/b/[BUCKET]/o/[OBJECT]`.
3. **Proses Delete:** Menghapus bucket yang masih ada isinya akan memunculkan error `409 Conflict`. Harus kosongkan isi dulu baru hapus wadahnya.
4. **Verifikasi Error:** Error code 404 saat menghapus sesuatu itu sebenarnya "Good News" karena artinya barangnya sudah hilang.

---

[⬅️ Kembali ke Menu Utama](../README.md)