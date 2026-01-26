Tentu, ini adalah materi modul lengkap berdasarkan transkrip video dan kuis yang kamu berikan. Modul ini disusun agar mudah dipahami, lengkap dengan visualisasi konsep (melalui image tags) dan navigasi yang rapi.

Silakan salin konten di bawah ini ke dalam file `README.md` atau dokumen belajarmu.

---

# 📚 Modul Lengkap: RAG, Embeddings, & Vector Search dengan BigQuery

Selamat datang di kursus mendalam tentang **RAG (Retrieval Augmented Generation)**. Modul ini akan membahas mengapa AI sering "berhalusinasi", bagaimana cara memperbaikinya menggunakan data perusahaan sendiri, dan langkah-langkah teknis implementasinya di BigQuery.

---

## 📑 Daftar Isi Materi

1. [Mengapa Kita Butuh RAG? (Masalah Halusinasi AI)](#1-mengapa-kita-butuh-rag-masalah-halusinasi-ai)
2. [Embeddings: Mengubah Data Menjadi Angka Bermakna](#2-embeddings-mengubah-data-menjadi-angka-bermakna)
3. [Vector Search: Mencari Jarum di Tumpukan Jerami](#3-vector-search-mencari-jarum-di-tumpukan-jerami)
4. [RAG in Action: Menggabungkan Semuanya](#4-rag-in-action-menggabungkan-semuanya)
5. [Ringkasan Teknis (BigQuery Functions)](#5-ringkasan-teknis-bigquery-functions)
6. [Kuis Evaluasi](#6-kuis-evaluasi)

---

## 1. Mengapa Kita Butuh RAG? (Masalah Halusinasi AI)

Kita sering menganggap AI sebagai ahli terpercaya, tetapi mereka memiliki kelemahan fatal: mereka bisa memberikan jawaban yang tidak akurat, kadaluarsa, atau terlalu umum. Fenomena ini sering disebut sebagai **AI Hallucination**.

### Keterbatasan Model Gen AI (LLM):

1. **Tidak Tahu Data Privat:** Model dilatih menggunakan data publik online, bukan data spesifik bisnis Anda (seperti stok gudang atau feedback pelanggan terkini).
2. **Kadaluarsa (Knowledge Cut-off):** Pengetahuan mereka terbatas pada tanggal terakhir mereka dilatih.
3. **Tidak Transparan:** Sulit untuk melacak dari mana sumber jawaban mereka berasal.

### Solusi: Retraining vs RAG

Ada dua cara untuk membuat AI "pintar" kembali:

* **Retraining/Fine-tuning:** Melatih ulang model dengan data baru. Ini **tidak praktis** karena memakan waktu berminggu-minggu dan biaya sangat mahal.
* **RAG (Retrieval Augmented Generation):** Memberikan konteks dan informasi relevan kepada model *pada saat* query dilakukan, tanpa mengubah otak model itu sendiri.

**Analogi:** Daripada menyuruh siswa menghafal seluruh isi perpustakaan (Retraining), lebih baik izinkan siswa membuka buku referensi saat ujian (RAG).

---

## 2. Embeddings: Mengubah Data Menjadi Angka Bermakna

Langkah pertama dalam RAG adalah **Embeddings**. Komputer tidak memahami teks atau gambar, mereka hanya mengerti angka.

### Tantangan Representasi Teks

* **Cara Lama (One-Hot Encoding):** Mengubah kata menjadi angka 0 dan 1. Masalahnya, matriksnya menjadi sangat besar (sparse) dan **tidak menangkap makna**. Kalimat "Anjing mengejar orang" dan "Orang mengejar anjing" akan terlihat sama secara matematis meskipun maknanya berlawanan.
* **Cara Baru (Word Embeddings):** Mengubah data menjadi vektor padat (dense vectors) berdimensi rendah yang menyimpan makna semantik.

### Konsep Semantic Meaning

Bayangkan mendeskripsikan seekor anjing. Anda tidak hanya bilang "itu anjing", tapi menjelaskan dimensinya: ras, ukuran, warna bulu, sifat, dll.

Embeddings melakukan hal yang sama pada kata. Kata ditempatkan dalam ruang vektor multidimensi.

* Kata yang mirip (misal: "Paris" dan "Tokyo") akan berada **berdekatan**.
* Kata yang beda (misal: "Paris" dan "Apel") akan berjauhan.
* Kita bahkan bisa bermain matematika dengan makna: **Paris - Perancis + Jepang = Tokyo**.

Fungsi BigQuery untuk ini: `ML.GENERATE_EMBEDDING`.

---

## 3. Vector Search: Mencari Jarum di Tumpukan Jerami

Setelah data diubah menjadi vektor (langkah Encode), kita perlu menyimpannya dan mencarinya. Ini disebut **Vector Search**.

### Proses Vector Search:

1. **Encode:** Ubah data multimodal (teks/gambar) menjadi vektor.
2. **Index:** Bangun ruang pencarian (index) agar pencarian bisa cepat dan skalabel.
3. **Search:** Cari vektor yang paling mirip dengan input query.

### Bagaimana Mengukur Kemiripan?

Kita mengukur jarak antar vektor di dalam ruang vektor. Ada beberapa metrik:

* **Cosine Distance:** Mengukur sudut antar vektor (fokus pada arah/kemiripan konteks).
* **Dot Product:** Mengukur arah dan besaran (magnitude).
* **Euclidean Distance:** Mengukur jarak garis lurus.

### Algoritma Pencarian

Untuk mencari di antara miliaran data dengan cepat, kita tidak bisa mengecek satu per satu (Brute Force). Google menggunakan algoritma seperti **ScaNN (Scalable Approximate Nearest Neighbor)** atau TreeAh yang membagi ruang pencarian menjadi pohon-pohon kecil untuk mempercepat proses.

Fungsi BigQuery untuk ini: `VECTOR_SEARCH`.

---

## 4. RAG in Action: Menggabungkan Semuanya

Bagaimana menggabungkan Embeddings, Vector Search, dan LLM untuk menjawab pertanyaan bisnis (misal: analisis feedback pelanggan Coffee on Wheels)?.

### Alur Kerja Pipeline RAG:

1. **User Bertanya:** "Bagaimana pendapat pelanggan tentang layanan kami akhir-akhir ini?".
2. **Encode Query:** Pertanyaan user diubah menjadi vektor.
3. **Search & Retrieve:** Cari di database vektor untuk menemukan 5-10 review pelanggan yang paling mirip/relevan dengan pertanyaan.
4. **Augment & Generate:** Kirim pertanyaan user **DITAMBAH** hasil review yang ditemukan tadi ke LLM (Gemini). Prompt-nya menjadi: *"Jawab pertanyaan ini berdasarkan data review berikut..."*.
5. **Hasil:** Gemini memberikan jawaban yang akurat, terkini, dan berlandaskan fakta.

Fungsi BigQuery untuk ini: `ML.GENERATE_TEXT`.

---

## 5. Ringkasan Teknis (BigQuery Functions)

Berikut adalah pemetaan fungsi SQL di BigQuery untuk setiap tahapan RAG:

| Tahapan | Deskripsi | Fungsi BigQuery SQL |
| --- | --- | --- |
| **1. Embeddings** | Mengubah teks/data menjadi vektor angka. | `ML.GENERATE_EMBEDDING` |
| **2. Indexing** | Membuat struktur data agar pencarian cepat. | `CREATE VECTOR INDEX` |
| **3. Vector Search** | Mencari kemiripan semantik. | `VECTOR_SEARCH` |
| **4. Generation** | Membuat jawaban AI (Gemini) dengan konteks. | `ML.GENERATE_TEXT` |

---

## 6. Kuis Evaluasi

Uji pemahaman Anda tentang materi RAG dan BigQuery.

**1. Apa urutan proses umum untuk melakukan Vector Search?**

* [ ] Connect, upload, and embed.
* [ ] Search, retrieve, and generate.
* [x] **Encode, index, and search.**
* [ ] Retrieve, evaluate, and generate.

> *Penjelasan: Data harus di-encode jadi vektor dulu, lalu di-index biar rapi, baru bisa di-search.*

**2. Anda sedang membangun pipeline RAG. Apa urutan langkah yang benar?**

* [ ] 1 → 3 → 5
* [ ] 1 → 2 → 3
* [x] **2 → 4 → 5**
* [ ] 2 → 3 → 4

> *Penjelasan: (2) Generate Embeddings -> (4) Search Vector Space -> (5) Augment & Generate Answer. Langkah training/fine-tuning tidak diperlukan di RAG standar.*

**3. Fungsi BigQuery apa yang digunakan untuk melakukan pencarian vektor?**

* [ ] HYBRID_SEARCH
* [ ] GENAI_SEARCH
* [ ] MULTIMODAL_SEARCH
* [x] **VECTOR_SEARCH**

**4. Fungsi BigQuery apa yang digunakan untuk membuat embeddings?**

* [ ] GENERATE_EMBEDDING
* [x] **ML.GENERATE_EMBEDDING**
* [ ] ML.GENERATE_TEXT
* [ ] GENERATE_MULTIMODAL

**5. Apa tujuan dari multimodal embedding?**

* [x] **Mengubah data multimodal menjadi representasi numerik sambil mempertahankan maknanya.**
* [ ] Mengidentifikasi sentimen dalam gambar.
* [ ] Membuat ruang vektor.
* [ ] Menggunakan one-hot encoding untuk mengubah teks menjadi angka.

> *Penjelasan: Tujuan utamanya adalah representasi numerik yang bermakna (semantic meaning) untuk berbagai tipe data.*

---

[⬅️ Kembali ke Menu Utama](../README.md)