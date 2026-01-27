# ☁️ Inspect Rich Documents with Gemini Multimodality and Multimodal RAG: Challenge Lab 

**Rute:** Generative AI on Vertex AI
**Topik:** Gemini Multimodality, Video Analysis, RAG (Retrieval Augmented Generation), Python SDK
**Tanggal Pengerjaan:** 28 Januari 2026

* [📂 Notebook/inspect_rich_documents_w_gemini_multimodality_and_multimodal_rag.ipynb](./Notebook/inspect_rich_documents_w_gemini_multimodality_and_multimodal_rag.ipynb)

---

## 🎯 1. Overview & Tujuan

Lab ini bertujuan untuk menguji kemampuan dalam menggunakan model **Gemini 1.5/2.5 Flash** untuk menganalisis data multimodal (Gambar & Video) dan membangun sistem **RAG (Retrieval Augmented Generation)** sederhana untuk mengekstrak informasi dari dokumen PDF (Google 10K Report).

**Misi Utama:**

1. **Multimodal Insights:** Menggunakan Gemini untuk menganalisis gambar branding dan video iklan Google Pixel 8 Pro.
2. **Video Understanding:** Mengekstrak tags, sentiment, dan deskripsi dari video.
3. **Multimodal RAG:** Membangun pipeline untuk mencari jawaban spesifik dari dokumen PDF menggunakan teknik *chunking* dan *similarity search*.

---

## 🛠️ 2. Langkah-Langkah & Solusi

Lab ini dikerjakan menggunakan **Vertex AI Workbench (JupyterLab)**. Berikut adalah potongan kode Python untuk mengisi bagian `"ENTER YOUR CODE HERE"` atau `"COMPLETE THE MISSING PART"`.

### 🔹 Task 1: Generate Multimodal Insights

**1.1 Image Understanding (Prompt 1)**

* **Tujuan:** Mengidentifikasi judul gambar branding.
* **Kode:**

```python
# Pastikan urutan contents sesuai instruksi: Instruksi -> Gambar -> Gambar -> Prompt1
contents = [
    instructions,
    image_ask_first_1,
    image_dont_do_this_1,
    prompt1
]

# Generate
responses = multimodal_model.generate_content(contents)

# Print
print("Prompts:")
print_multimodal_prompt(contents)
print("\nModel Responses:")
print(responses.text)

```

**1.2 Image Comparison (Prompt 2 & 3)**

* **Tujuan:** Membandingkan dua gambar branding (Do's and Don'ts).
* **Kode:**

```python
contents = [
    prompt1,
    image_ask_first_3,
    prompt2,
    image_dont_do_this_3,
    prompt3
]

generation_config = {
    "temperature": 0.1,
    "max_output_tokens": 2048
}

responses = multimodal_model.generate_content(contents, generation_config=generation_config)

```

**1.3 - 1.6 Video Analysis (Pixel 8 Pro)**

* **Tujuan:** Menganalisis video untuk deskripsi, tags, dan extra info.
* **Kode Umum untuk Video:**

```python
# Definisi Video (Pastikan MIME Type benar)
video = Part.from_uri(
    uri="gs://spls/gsp520/google-pixel-8-pro.mp4",
    mime_type="video/mp4",
)

# Gabung dengan Prompt
contents = [video, prompt]

# Generate
responses = multimodal_model.generate_content(contents)
print(responses.text)

```

*(Ulangi pola ini untuk sub-task 1.3, 1.4, 1.5, dan 1.6 dengan prompt yang sudah disediakan di notebook).*

### 🔹 Task 2: Multimodal RAG

**2.1 Build Metadata (Extract PDF)**

* **Penting:** Import fungsi helper terlebih dahulu.
* **Kode:**

```python
# Import (Wajib dijalankan jika belum ada)
from utils.intro_multimodal_rag_utils import get_document_metadata

# Ekstrak Metadata
text_metadata_df, image_metadata_df = get_document_metadata(
    multimodal_model,
    pdf_folder_path, # Gunakan variabel path folder yang benar
    image_save_dir="images",
    image_description_prompt=image_description_prompt,
    embedding_size=1408,
)

```

**2.3 Get Relevant Text Chunks**

* **Tujuan:** Mencari potongan teks yang relevan dengan query user.
* **Kode:**

```python
matching_results_chunks_data = get_similar_text_from_query(
    query,
    text_metadata_df,
    column_name="text_embedding_chunk",
    top_n=5,
    chunk_text=True,
)

```

**2.4 Create Context**

* **Tujuan:** Menggabungkan potongan teks menjadi satu string konteks.
* **Kode:**

```python
context_text = []
for key, value in matching_results_chunks_data.items():
    context_text.append(value["chunk_text"])

final_context_text = "\n".join(context_text)

```

**2.5 Generate Final RAG Response**

* **Tujuan:** Mengirim konteks + query ke Gemini untuk dijawab.
* **Kode:**

```python
generation_config = GenerationConfig(temperature=0.1, max_output_tokens=2048)

get_gemini_response(
    multimodal_model,
    model_input=[prompt],
    stream=True,
    generation_config=generation_config,
)

```

---

### 🐛 3. Troubleshooting / Masalah yang Dihadapi

**Error 1:** `NameError: name 'model' is not defined`

* **Penyebab:** Kode copy-paste menggunakan variabel `model`, sedangkan notebook bawaan lab mendefinisikan model sebagai `multimodal_model`.
* **Solusi:** Ubah semua referensi `model` menjadi `multimodal_model`.

**Error 2:** `Check my progress` gagal di Task 1.1 ("Please run recommended cells...").

* **Penyebab:**
1. Salah menggunakan variabel prompt (menggunakan `prompt2` padahal task meminta `prompt1`).
2. Notebook belum disimpan (**Save**) sebelum tombol check ditekan.


* **Solusi:** Gunakan `prompt1` untuk task pertama, run cell, lalu tekan `Ctrl+S` untuk menyimpan notebook sebelum verifikasi.

**Error 3:** `NameError: name 'get_document_metadata' is not defined` (Task 2).

* **Penyebab:** Cell import fungsi dari `utils` terlewat atau belum dijalankan.
* **Solusi:** Jalankan cell import: `from utils.intro_multimodal_rag_utils import ...`.

---

### 📝 4. Catatan Penting (Key Takeaways)

Hal-hal baru yang saya pelajari dari lab ini:

1. **Multimodal Inputs:** Gemini sangat fleksibel menerima input campuran dalam satu list `contents` (misal: `[Teks Instruksi, Gambar A, Teks Prompt, Gambar B]`).
2. **MIME Types:** Saat menggunakan `Part.from_uri` untuk video, sangat penting mendefinisikan `mime_type="video/mp4"` agar model bisa memprosesnya.
3. **RAG Workflow:** Inti dari RAG adalah: *Query -> Embed Query -> Search Similarity (Vector Database) -> Retrieve Chunks -> Create Context -> Generate Answer*.

---

[⬅️ Kembali ke Menu Utama](../README.md)