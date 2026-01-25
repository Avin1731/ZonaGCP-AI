# 📚 Modul Lengkap: Vector Search & Embeddings

Selamat datang di kursus komprehensif **Vector Search dan Embeddings**. Modul ini merangkum seluruh materi teknis mendalam tentang bagaimana membangun mesin pencari cerdas (AI-powered search) menggunakan teknologi Google Cloud Vertex AI.

---

## 📑 Daftar Isi Materi

## 📑 Daftar Isi

1. [Pendahuluan: Era Baru Pencarian](#pendahuluan-era-baru-pencarian)
2. [Deep Dive: Vector Search Basics](#deep-dive-vector-search-basics)
3. [Teknik Encoding: Dari One-Hot ke Embeddings](#teknik-encoding-dari-one-hot-ke-embeddings)
4. [Mekanisme Indexing & Algoritma Pencarian](#mekanisme-indexing--algoritma-pencarian)
5. [Implementasi dengan Vertex AI Vector Search](#implementasi-dengan-vertex-ai-vector-search)
6. [Solusi Halusinasi AI: RAG (Retrieval Augmented Generation)](#solusi-halusinasi-ai-rag-retrieval-augmented-generation)
7. [Hybrid Search: Menggabungkan Terbaik dari Dua Dunia](#hybrid-search-menggabungkan-terbaik-dari-dua-dunia)
8. [Kuis Evaluasi](#kuis-evaluasi)

---

## 1. Pendahuluan: Era Baru Pencarian

Pencarian tradisional hanya mencocokkan kata. Pencarian masa kini (AI-powered) harus memahami **konteks**, **multimodal**, dan bisa bertindak sebagai **agen**.

### Contoh Kasus Penggunaan Canggih:

1. **Multimodal Search (Gambar ke Teks):** Kamu melihat foto tempat liburan, upload fotonya, lalu bertanya: "Di mana ini? Buatkan rencana liburan 10 hari di area ini." Mesin harus paham gambar dan teks sekaligus.
2. **Factual Knowledge (Pencarian Fakta):** "Carikan video 10 menit tentang sejarah Italia." Ini butuh pencarian keyword yang spesifik namun kontekstual.
3. **AI Agents (Aksi Nyata):** "Cek jatah cuti saya, jadwal kerja tim, dan peringatan keamanan perjalanan, lalu booking tiketnya." Di sini search engine harus memverifikasi informasi internal perusahaan (RAG) dan melakukan aksi nyata.

---

## 2. Deep Dive: Vector Search Basics

### Apa itu Vector Search?

Vector Search adalah teknologi pencarian yang berfokus pada **kemiripan makna (semantic similarity)**. Ia memungkinkan kita mencari item (teks, gambar, audio) yang memiliki "makna" serupa, bukan hanya kata yang sama.

### Mengapa Keyword Search Saja Tidak Cukup?

Keyword search tradisional (seperti SQL `LIKE`) tidak paham konteks.

* *Kasus:* User mencari "baju pesta pantai".
* *Keyword Search:* Mungkin gagal menemukan "bikini" atau "celana renang" karena katanya beda.
* *Vector Search:* Paham bahwa "bikini" berhubungan erat dengan "pantai" dan "pesta".

### 3 Tahapan Utama (The Vector Search Pipeline)

Untuk membuat sistem ini, ada 3 langkah wajib yang harus dilakukan berurutan:

1. **Encode:** Mengubah data (teks/gambar) menjadi angka (vektor) menggunakan model AI (Embeddings).
2. **Index:** Mengorganisir miliaran vektor tersebut agar bisa dicari dengan cepat (seperti membuat daftar isi buku).
3. **Search:** Melakukan pencarian di dalam ruang vektor (Vector Space) untuk menemukan tetangga terdekat dari query user.

---

## 3. Teknik Encoding: Dari One-Hot ke Embeddings

Bagaimana cara komputer memahami kata "Ratu" atau "Anjing"? Kita harus mengubahnya menjadi angka.

### Cara Lama: One-Hot Encoding (Basic Vectorization)

Teknik ini mengubah kata menjadi angka 1 dan 0.

* Misal kosakata: [Anjing, Kucing, Kejar, Orang].
* Kalimat "Anjing kejar orang" -> [1, 0, 1, 1].
* **Kelemahan Fatal:**
1. **Sparse (Jarang):** Jika kosakata ada 10.000, maka vektornya punya 10.000 dimensi dengan 99% isinya angka nol. Boros memori.
2. **Tidak Ada Makna:** Tidak ada hubungan matematis antara "Anjing" dan "Kucing" dalam model ini.



### Cara Baru: Text Embeddings (Dense Vectors)

Teknik ini menggunakan Neural Network (seperti Word2Vec, GloVe) untuk membuat vektor yang **Padat (Dense)** dan **Berdimensi Rendah** (misal 768 dimensi, bukan 10.000).

* **Keunggulan Utama:** Menangkap hubungan semantik. Kata yang mirip akan memiliki angka vektor yang mirip.
* **Analogi Matematika Kata:**
Sistem embedding bisa melakukan operasi matematika pada makna:
> **King (Raja) - Man (Pria) + Woman (Wanita) ≈ Queen (Ratu)**
> Ini membuktikan bahwa model memahami konsep gender dan status kerajaan dalam bentuk angka.



---

## 4. Mekanisme Indexing & Algoritma Pencarian

Setelah data jadi vektor, kita butuh cara mengukur "kemiripan" dan cara mencarinya dengan cepat.

### 4 Metode Mengukur Jarak (Distance Metrics)

Bagaimana kita tahu dua vektor itu mirip? Kita ukur jaraknya.

1. **Manhattan Distance (L1):** Mengukur jarak tegak lurus (seperti blok jalanan kota).
2. **Euclidean Distance (L2):** Mengukur garis lurus terpendek antar titik.
3. **Cosine Distance:** Mengukur **sudut** antar vektor. 0 berarti sejajar (mirip), 1 berarti tegak lurus (beda). Fokus pada arah, bukan panjang.
4. **Dot Product Distance:** Mengukur kesamaan berdasarkan arah **DAN** besaran (magnitude). Ini adalah default yang disarankan di Vertex AI.

### Algoritma Pencarian: Brute Force vs ANN

* **Brute Force:** Mengecek satu per satu data. Akurat 100%, tapi lambat banget kalau datanya miliaran.
* **TreeAh (Shallow Tree + Asymmetric Hashing):** Algoritma pendekatan (approximate) yang digunakan di production. Sangat cepat.

### Teknologi ScaNN (Scalable Approximate Nearest Neighbor)

Vertex AI menggunakan algoritma ScaNN dari Google Research. ScaNN menggunakan 3 trik agar pencarian jadi super cepat:

1. **Space Pruning (Pemangkasan Ruang):** Membagi data ke dalam struktur pohon (tree). Kalau query ada di cabang A, sistem langsung membuang (prune) cabang B, C, D agar tidak perlu dicek.
2. **Data Quantization (Kompresi):** Mengecilkan ukuran vektor (misal dari float besar ke integer kecil) agar hemat memori dan cepat dihitung.
3. **Business Logic Filtering:** Menambahkan filter aturan (misal: "Hanya cari baju warna merah") di awal proses.

---

## 5. Implementasi dengan Vertex AI Vector Search

Google Cloud menyediakan layanan **Vertex AI Vector Search** (dulunya Matching Engine) yang *fully managed*.

### Fitur Utama:

* Skala miliaran vektor.
* Latensi rendah (milidetik).
* Biaya 30-50% lebih murah dari layanan sejenis.

### Workflow Pembuatan (UI & Code):

1. **Persiapan Data:** Simpan embeddings (dari BigQuery/API) ke format JSON di Cloud Storage.
2. **Create Index:**
* Tentukan algoritma (TreeAh/BruteForce).
* Tentukan dimensi (sesuai model, misal 768).
* Tentukan *Shard Size* (pecahan data).


3. **Deploy Endpoint:** Pasang index ke endpoint agar bisa diakses aplikasi.
4. **Query:** Kirim request pencarian dan dapatkan hasil similarity.

---

## 6. Solusi Halusinasi AI: RAG (Retrieval Augmented Generation)

### Masalah: AI Hallucination

LLM (seperti Gemini/GPT) itu pintar tapi "sok tahu".

* Mereka tidak tahu data internal perusahaanmu.
* Mereka tidak tahu berita hari ini (terbatas pada data training lama).
* Akibatnya, mereka sering **berhalusinasi** (mengarang jawaban).

### Solusi: RAG (Ujian Buka Buku)

RAG menggabungkan **Generative AI** dengan **Retrieval (Pencarian)**.
Mekanismenya:

1. **User Bertanya:** "Berapa sisa cuti saya?"
2. **Retrieval:** Sistem mencari dokumen cuti terbaru di database HR menggunakan **Vector Search**.
3. **Augmentation:** Hasil pencarian disisipkan ke dalam prompt sebagai konteks.
4. **Generation:** LLM menjawab berdasarkan data yang ditemukan: "Berdasarkan dokumen HR, sisa cuti Anda 5 hari.".

Ini menciptakan **Grounded Agent**, agen AI yang berlandaskan fakta.

---

## 7. Hybrid Search: Menggabungkan Terbaik dari Dua Dunia

### Kelemahan Semantic Search

Semantic Search (Vector) terkadang gagal mengenali:

* **Out-of-domain data:** Kata-kata yang tidak ada di data training model (misal: kode SKU produk baru, nama project rahasia, merk obat baru).

### Solusi: Hybrid Search

Menggabungkan dua metode pencarian sekaligus:

1. **Dense Embedding (Semantic):** Menggunakan model AI untuk menangkap makna konteks (misal: "baju anak" cocok dengan "kaos youth").
2. **Sparse Embedding (Keyword):** Menggunakan teknik statistik seperti **TF-IDF**.
* **TF-IDF (Term Frequency - Inverse Document Frequency):** Menghitung seberapa penting sebuah kata dalam dokumen. Kata unik/jarang (seperti nama merk spesifik) diberi bobot tinggi.
* Hasilnya adalah vektor yang "bolong-bolong" (Sparse) tapi sangat presisi untuk kata kunci spesifik.



### Cara Menggabungkan (Reranking dengan RRF)

Hasil dari pencarian Dense dan Sparse digabung menggunakan algoritma **Reciprocal Rank Fusion (RRF)**.

* **Logika RRF:** Jika sebuah item muncul di ranking atas pada kedua metode (Dense & Sparse), item tersebut akan didorong ke posisi paling atas di hasil akhir.
* Ini memastikan kita mendapatkan item yang secara makna relevan DAN secara kata kunci tepat.

---

## 8. Kuis Evaluasi

Jawablah pertanyaan berikut untuk menguji pemahamanmu (Kunci jawaban ada di materi).

**1. Apa keunggulan utama Semantic Search dibandingkan Keyword Search tradisional?**

* [ ] Lebih cepat.
* [x] **Paham makna dan konteks kata, bukan cuma mencocokkan huruf.**
* [ ] Bisa mencari data gambar saja.

**2. Apa yang dimaksud dengan "Dense Vectors" dalam Text Embeddings?**

* [ ] Vektor dengan banyak angka nol.
* [x] **Vektor berdimensi rendah yang padat informasi makna (semantic meaning).**
* [ ] Vektor yang dibuat manual oleh manusia.

**3. Manakah metrik jarak yang mengukur SUDUT antar dua vektor?**

* [ ] Euclidean Distance.
* [ ] Manhattan Distance.
* [x] **Cosine Distance.**

**4. Apa fungsi utama teknik "Quantization" pada algoritma ScaNN?**

* [ ] Menambah dimensi data.
* [x] **Mengkompres ukuran data vektor agar hemat memori dan cepat dicari.**
* [ ] Menghapus data yang duplikat.

**5. Bagaimana RAG mencegah halusinasi pada AI?**

* [ ] Dengan melatih ulang model setiap hari.
* [x] **Dengan mengambil data fakta (Retrieval) lewat Vector Search dan memberikannya ke LLM sebagai referensi.**
* [ ] Dengan membatasi panjang jawaban AI.

**6. Apa peran Sparse Embedding (TF-IDF) dalam Hybrid Search?**

* [ ] Memahami emosi user.
* [ ] Mengubah gambar jadi teks.
* [x] **Menangkap kata kunci spesifik (keyword matching) yang mungkin tidak dikenali oleh model semantik.**

**7. Algoritma apa yang digunakan untuk menggabungkan hasil ranking dari Dense dan Sparse search?**

* [ ] Brute Force.
* [ ] TreeAh.
* [x] **Reciprocal Rank Fusion (RRF).**

---

*Materi disusun berdasarkan kurikulum Google Cloud Skills Boost: Vector Search & Embeddings.*

[⬅️ Kembali ke Menu Utama](../README.md)