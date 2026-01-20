# ☁️  Monitor and Log with Google Cloud Observability: Challenge Lab

**Rute:** Cloud Operations / Observability
**Topik:** Cloud Monitoring, Custom Metrics, Logging, Alerting, Compute Engine
**Tanggal Pengerjaan:** 20 Januari 2026

---

## 🎯 1. Overview & Tujuan

Lab ini bertujuan untuk menguji kemampuan dalam memonitor infrastruktur aplikasi pemrosesan video. Kita diminta untuk memperbaiki *monitoring agent* yang rusak pada VM, membuat *custom metric* dari log aplikasi, serta memvisualisasikan data tersebut ke dalam Dashboard dan membuat Alerting.

**Misi Utama:**

1. Mengaktifkan Cloud Monitoring.
2. Memperbaiki aplikasi Go pada VM `video-queue-monitor` agar bisa mengirim custom metric (`input_queue_size`).
3. Membuat metric berbasis log (`huge_video_upload_rate`) untuk mendeteksi upload video 4K/8K.
4. Menampilkan kedua metric tersebut di Media Dashboard.
5. Membuat Alert jika rate upload tinggi melebihi batas wajar.

---

## 🛠️ 2. Langkah-Langkah & Solusi

### 🔹 Task 1: Initialize Cloud Monitoring

* **Deskripsi:** Mengaktifkan layanan monitoring pada project.
* **Konfigurasi GUI:**
* Menu: `Navigation Menu > Monitoring`
* Tunggu hingga dashboard monitoring dimuat sepenuhnya.



### 🔹 Task 2: Configure Compute Instance (Custom Metric)

* **Masalah:** Startup script bawaan lab rusak (link file Go 404/Corrupt).
* **Solusi:** Melakukan SSH ke VM dan membuat file aplikasi Go secara manual.
* **Perintah CLI / Script:**
*(Jalankan perintah berikut di SSH Terminal VM `video-queue-monitor`)*

```bash
# 1. Bersihkan file lama & Setup Environment Variable (WAJIB)
rm -f video_queue_tool.go
export GOOGLE_CLOUD_PROJECT=$(curl -s http://metadata.google.internal/computeMetadata/v1/project/project-id -H "Metadata-Flavor: Google")
export MY_GCE_INSTANCE_ID=$(curl -s http://metadata.google.internal/computeMetadata/v1/instance/id -H "Metadata-Flavor: Google")
export MY_GCE_INSTANCE_ZONE=$(curl -s http://metadata.google.internal/computeMetadata/v1/instance/zone -H "Metadata-Flavor: Google" | awk -F/ '{print $NF}')

# 2. Re-create File Go & Dependency (Manual karena link lab mati)
/usr/local/go/bin/go mod init video_queue

# (Disini paste kode Go manual untuk simulasi queue - lihat catatan troubleshooting)

# 3. Download dependency & Jalankan App
/usr/local/go/bin/go get contrib.go.opencensus.io/exporter/stackdriver
/usr/local/go/bin/go get go.opencensus.io/stats
/usr/local/go/bin/go mod tidy
/usr/local/go/bin/go run video_queue_tool.go

```

### 🔹 Task 3: Create Custom Metric (Log-based)

* **Deskripsi:** Membuat metric untuk menghitung jumlah upload video resolusi tinggi.
* **Konfigurasi GUI:**
* Menu: `Logging > Logs Explorer`
* Query: `textPayload=~"file_format\: ([4,8]K).*"`
* Action: Klik **Create Metric**
* Metric Type: `Counter`
* Log metric name: `huge_video_upload_rate`



### 🔹 Task 4: Add Custom Metrics to Dashboard

* **Deskripsi:** Menambahkan chart ke "Media_Dashboard".
* **Langkah Konfigurasi:**
1. Buka `Monitoring > Dashboards > Media_Dashboard`.
2. **Chart 1 (Queue):**
* Metric: `input_queue_size`
* Resource: `VM Instance > Custom metrics`
* **Note:** Matikan tombol "Active" agar metric muncul.


3. **Chart 2 (Upload Rate):**
* Metric: `huge_video_upload_rate`
* Resource: `Logs-based Metric`





### 🔹 Task 5: Create Alert

* **Deskripsi:** Membuat alert jika upload video besar terlalu banyak.
* **Konfigurasi GUI:**
* Menu: `Monitoring > Alerting > Create Policy`
* Select Metric: `huge_video_upload_rate`
* Rolling window: `1 min`
* Threshold type: `Above threshold`
* Threshold value: `5`



---

## 🐛 3. Troubleshooting / Masalah yang Dihadapi

**Error 1:** `video_queue_tool.go:1:1: expected 'package', found '<'`

* **Penyebab:** File yang didownload via `curl` di startup script ternyata berisi pesan error HTML (Broken Link/404), bukan source code Go.
* **Solusi:** Menghapus file corrupt, lalu membuat file `video_queue_tool.go` baru secara manual menggunakan `cat > ... <<EOF` dengan kode Go yang valid, lalu install dependency manual.

**Error 2:** Metric `input_queue_size` tidak muncul di Dashboard ("No results").

* **Penyebab:**
1. VM belum mengirim data karena script error.
2. Filter **"Active"** di menu *Select a metric* menyala, menyembunyikan metric yang baru dibuat.


* **Solusi:**
1. Jalankan aplikasi Go secara manual via SSH.
2. **Matikan (Uncheck)** tombol "Active" saat mencari nama metric.



---

### 📝 4. Catatan Penting (Key Takeaways)

Hal-hal baru yang saya pelajari dari lab ini:

1. **Broken Lab Files:** Jangan selalu percaya 100% pada resource lab (link bucket). Jika file corrupt, kita harus paham coding (Go/Python) dasar untuk memperbaikinya manual.
2. **Environment Variables:** Aplikasi custom metric butuh Env Var (`PROJECT_ID`, `INSTANCE_ID`) yang valid. Startup script sering gagal set ini.
3. **Cloud Monitoring GUI:** Tombol **"Active"** pada menu *Metrics Explorer* seringkali menjebak karena menyembunyikan metric custom yang datanya baru masuk. Selalu matikan filter ini jika metric tidak ketemu.

---

[⬅️ Kembali ke Menu Utama](../README.md)