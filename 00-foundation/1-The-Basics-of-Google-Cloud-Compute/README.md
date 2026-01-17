# ☁️ The Basics of Google Cloud Compute: Challenge Lab

**Rute:** Rute 0 — Foundation
**Topik:** Compute Engine, Cloud Storage, Persistent Disk, Nginx
**Tanggal Pengerjaan:** 17 Januari 2026

---

## 🎯 1. Overview & Tujuan
Lab ini bertujuan untuk menguji pemahaman dasar tentang infrastruktur Google Cloud dengan cara membangun server web sederhana yang memiliki penyimpanan terpisah dan file storage.

**Misi Utama:**
1. Membuat Cloud Storage Bucket (untuk menyimpan file).
2. Membuat VM Instance dengan Persistent Disk tambahan (untuk server & storage).
3. Menginstall Nginx Web Server via SSH.

---

## 🛠️ 2. Langkah-Langkah & Solusi

**Konfigurasi Global:**
* **Region:** `europe-west4`
* **Zone:** `europe-west4-b`

### 🔹 Task 1: Create a Cloud Storage bucket
* **Deskripsi:** Membuat bucket penyimpanan dengan nama unik yang sudah ditentukan.
* **Konfigurasi GUI:**
  * Menu: `Cloud Storage > Buckets > Create`
  * Name: `qwiklabs-gcp-01-63fef0cd2e00-bucket`
  * Location Type: `Multi-region`
  * Location: `US` (Sesuai instruksi khusus task, walaupun default lab di Europe)

### 🔹 Task 2: Create and attach a persistent disk to a Compute Engine instance
**Langkah A: Membuat VM Instance**
* **Konfigurasi GUI:**
  * Menu: `Compute Engine > VM instances > Create Instance`
  * Name: `my-instance`
  * Region: `europe-west4`
  * Zone: `europe-west4-b`
  * Series: `E2`
  * Machine type: `e2-medium`
  * Boot disk: `Debian GNU/Linux 12 (bookworm)`, Type `Balanced persistent disk`, Size `10 GB`
  * Firewall: Centang `Allow HTTP traffic`
  * **Klik Create**

**Langkah B: Membuat Persistent Disk**
* **Konfigurasi GUI:**
  * Menu: `Compute Engine > Disks > Create Disk`
  * Name: `mydisk`
  * Region: `europe-west4`
  * Zone: `europe-west4-b` (**Wajib sama** dengan Zone VM)
  * Size: `200 GB`
  * **Klik Create**

**Langkah C: Attach Disk ke VM**
* **Langkah:**
  1. Kembali ke menu `VM instances`.
  2. Klik nama VM `my-instance`, lalu klik **Edit**.
  3. Scroll ke bagian **Additional disks**.
  4. Klik **+ ATTACH EXISTING DISK**.
  5. Pilih `mydisk`.
  6. Scroll ke bawah dan klik **Save**.

### 🔹 Task 3: Install a NGINX web server
* **Perintah CLI / Script:**
  *(Masuk ke SSH `my-instance` lalu jalankan perintah berikut)*
```bash
  # 1. Update repository OS
  sudo apt update

  # 2. Install Nginx Web Server
  sudo apt install nginx -y
```
## 🐛 3. Troubleshooting / Masalah yang Dihadapi
Error: Disk mydisk tidak muncul saat mau di-attach ke VM.

Penyebab: Zone Disk berbeda dengan Zone VM (Misal VM di europe-west4-b, tapi Disk dibuat di europe-west4-a).

Solusi:

Hapus Disk yang salah, lalu buat ulang Disk baru dengan memastikan Zone sama persis dengan VM (europe-west4-b).

## 📝 4. Catatan Penting (Key Takeaways)
Hal-hal baru yang saya pelajari dari lab ini:

Region vs Zone: Disk fisik harus berada di Zone yang sama dengan Komputernya (VM) agar bisa dicolok (attach).

Exception Rule: Meskipun lab meminta default di europe-west4, instruksi Bucket meminta US multi-region. Instruksi spesifik per-task mengalahkan instruksi general.

Web Server: Menginstall Nginx mengubah VM biasa menjadi Web Server yang bisa diakses via External IP.