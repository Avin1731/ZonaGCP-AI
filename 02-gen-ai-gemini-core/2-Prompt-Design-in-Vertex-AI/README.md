# ☁️ Prompt Design in Vertex AI: Challenge Lab

**Rute:** Generative AI on Vertex AI
**Topik:** Prompt Engineering, Vertex AI Studio, Python SDK, Gemini 2.5 Flash
**Tanggal Pengerjaan:** 21 Januari 2026

---

## 🎯 1. Overview & Tujuan

Lab ini mensimulasikan skenario di mana kita bekerja untuk startup konten edukasi yang bermitra dengan "Cymbal Direct". Tujuannya adalah membuat aset pemasaran (deskripsi produk dan tagline) menggunakan kemampuan Generative AI di Google Cloud Vertex AI.

**Misi Utama:**

1. **Image Analysis Tool:** Membuat prompt di Vertex AI Studio untuk menganalisis gambar produk dan menghasilkan deskripsi puitis/iklan.
2. **Tagline Generator:** Membuat prompt dengan teknik *few-shot prompting* untuk menghasilkan tagline produk yang menarik.
3. **Modifikasi Kode Python (Image Analysis):** Mengedit kode Python di Jupyter Notebook untuk mengubah panjang output (< 10 kata) dan meningkatkan kreativitas (Temperature).
4. **Modifikasi Kode Python (Tagline Generator):** Mengedit kode Python untuk menyertakan keyword spesifik ("nature") dalam output.

---

## 🛠️ 2. Langkah-Langkah & Solusi

### 🔹 Task 1: Build a Gemini image analysis tool

* **Deskripsi:** Membuat prompt analisis gambar menggunakan model Gemini 2.5 Flash di Vertex AI Studio.
* **Konfigurasi GUI:**
* **Menu:** `Vertex AI > Vertex AI Studio > Freeform`
* **Model:** `gemini-2.5-flash` (**PENTING:** Harus versi 2.5 Flash, jangan salah pilih model lain).
* **Media:** Upload gambar dari Cloud Storage (`cymbal-product-image.png` dari bucket lab).
* **Prompt:**
```text
Analyze this image and generate:
1. Short, descriptive text inspired by the image.
2. Catchy phrases suitable for advertisements.
3. A poetic description for a nature-focused campaign.

```


* **Save Settings:**
* **Name:** `Cymbal Product Analysis`
* **Region:** `europe-west4` (Wajib sesuai instruksi lab).





### 🔹 Task 2: Build a Gemini tagline generator

* **Deskripsi:** Membuat generator tagline dengan memberikan contoh (examples) kepada model.
* **Konfigurasi GUI:**
* **Menu:** `Vertex AI > Vertex AI Studio > Freeform`
* **Model:** `gemini-2.5-flash`
* **System Instructions:**
```text
Cymbal Direct is partnering with an outdoor gear retailer. They're launching a new line of products designed to encourage young people to explore the outdoors. Help them create catchy taglines for this product line.

```


* **Examples (Few-shot):**
* *Input:* `Write a tagline for a durable backpack designed for hikers that makes them feel prepared. Consider styles like minimalist.`
* *Output:* `Built for the Journey: Your Adventure Essentials.`


* **Save Settings:**
* **Name:** `Cymbal Tagline Generator Template`
* **Region:** `europe-west4`





### 🔹 Task 3: Experiment with image analysis code

* **Deskripsi:** Membuka notebook `image-analysis.ipynb`, memperbaiki autentikasi, mengubah prompt agar output < 10 kata, dan menaikkan parameter temperature.
* **Lokasi File:** `image-analysis.ipynb`
* **Kode Solusi (Final):**
*(Perhatikan perubahan pada `client` auth, `location`, `temperature=2.0`, model `gemini-2.5-flash`, dan prompt text)*

```python
from google import genai
from google.genai import types
import base64
import os

def generate():
  # 1. SETUP CLIENT (Auth menggunakan Project ID & Region europe-west4)
  client = genai.Client(
      vertexai=True,
      location="europe-west4",
      project=os.environ.get("GOOGLE_CLOUD_PROJECT"),
  )

  # 2. MODIFIKASI PROMPT (Harus < 10 kata & Creative)
  msg1_text1 = types.Part.from_text(text="""Describe this image in less than 10 words. Be extremely creative and unexpected.""")
  
  # 3. SET BUCKET URI YANG BENAR (Sesuai lab session)
  # Pastikan bucket ini sesuai dengan ID project lab saat ini
  msg1_image1 = types.Part.from_uri(
      file_uri="gs://qwiklabs-gcp-03-ea9bf4e3f5d2-bucket/cymbal-product-image.png",
      mime_type="image/png",
  )

  # 4. MEMASTIKAN MODEL YANG BENAR
  model = "gemini-2.5-flash"
  
  contents = [
    types.Content(
      role="user",
      parts=[
        msg1_text1,
        msg1_image1
      ]
    ),
  ]
  
  tools = [
    types.Tool(google_search=types.GoogleSearch()),
  ]

  # 5. MODIFIKASI PARAMETER (Temperature = 2.0 untuk kreativitas maksimal)
  # Menghapus thinking_config karena model Flash 2.5 tidak support
  generate_content_config = types.GenerateContentConfig(
    temperature = 2.0,
    top_p = 0.95,
    max_output_tokens = 8192,
    safety_settings = [
        types.SafetySetting(category="HARM_CATEGORY_HATE_SPEECH", threshold="OFF"),
        types.SafetySetting(category="HARM_CATEGORY_DANGEROUS_CONTENT", threshold="OFF"),
        types.SafetySetting(category="HARM_CATEGORY_SEXUALLY_EXPLICIT", threshold="OFF"),
        types.SafetySetting(category="HARM_CATEGORY_HARASSMENT", threshold="OFF")
    ],
    tools = tools,
  )

  for chunk in client.models.generate_content_stream(
    model = model,
    contents = contents,
    config = generate_content_config,
    ):
    if not chunk.candidates or not chunk.candidates[0].content or not chunk.candidates[0].content.parts:
        continue
    print(chunk.text, end="")

generate()

```

### 🔹 Task 4: Experiment with tagline generation code

* **Deskripsi:** Membuka notebook `tagline-generator.ipynb` dan memodifikasi prompt input terakhir agar menyertakan keyword "nature".
* **Lokasi File:** `tagline-generator.ipynb`
* **Kode Solusi (Final):**
*(Perhatikan penambahan kalimat "Include the keyword nature" di variable `msg1_text1`)*

```python
from google import genai
from google.genai import types
import base64
import os

def generate():
  # 1. SETUP CLIENT (Region europe-west4)
  client = genai.Client(
      vertexai=True,
      location="europe-west4",
      project=os.environ.get("GOOGLE_CLOUD_PROJECT"),
  )

  # 2. MODIFIKASI PROMPT (Menambahkan 'Include the keyword nature' di input terakhir)
  msg1_text1 = types.Part.from_text(text="""Example 1Input: Write a tagline for a lightweight tent designed for solo travelers that makes them feel independent. Consider styles like inspiring.Output: Solitude redefined: Your home, anywhere.Example 2Input: Write a tagline for heavy-duty hiking boots designed for mountaineers that makes them feel unstoppable. Consider styles like bold and rugged.Output: Conquer every peak. Leave no path untaken.Example 3Input: Write a tagline for a portable solar charger designed for digital nomads that makes them feel connected. Consider styles like modern and tech-focused.Output: Unlimited energy for your limitless journey.Example 4Input: Write a tagline for a waterproof jacket designed for rainy city commuters that makes them feel protected. Consider styles like witty.Output: Rain happens. So does your day.Example 5Input: Write a tagline for a recycled material water bottle designed for eco-conscious hikers that makes them feel responsible. Consider styles like wholesome and earthy.Output: Drink deep. Tread light. Nature thanks you.
Test Input: Write a tagline for a warm sleeping bag designed for winter campers that makes them feel cozy and safe. Consider styles like comforting. Include the keyword nature.
Output""")
  
  si_text1 = """Cymbal Direct is partnering with an outdoor gear retailer. They're launching a new line of products designed to encourage young people to explore the outdoors. Help them create catchy taglines for this product line."""

  # 3. MEMASTIKAN MODEL YANG BENAR
  model = "gemini-2.5-flash"
  
  contents = [
    types.Content(
      role="user",
      parts=[msg1_text1]
    ),
  ]
  
  tools = [
    types.Tool(google_search=types.GoogleSearch()),
  ]

  # 4. CONFIG CLEANUP (Hapus thinking_config agar tidak conflict dengan Flash 2.5)
  generate_content_config = types.GenerateContentConfig(
    temperature = 1,
    top_p = 1,
    max_output_tokens = 8192,
    safety_settings = [
        types.SafetySetting(category="HARM_CATEGORY_HATE_SPEECH", threshold="OFF"),
        types.SafetySetting(category="HARM_CATEGORY_DANGEROUS_CONTENT", threshold="OFF"),
        types.SafetySetting(category="HARM_CATEGORY_SEXUALLY_EXPLICIT", threshold="OFF"),
        types.SafetySetting(category="HARM_CATEGORY_HARASSMENT", threshold="OFF")
    ],
    tools = tools,
    system_instruction=[types.Part.from_text(text=si_text1)],
    # thinking_config DIHAPUS
  )

  for chunk in client.models.generate_content_stream(
    model = model,
    contents = contents,
    config = generate_content_config,
    ):
    if not chunk.candidates or not chunk.candidates[0].content or not chunk.candidates[0].content.parts:
        continue
    print(chunk.text, end="")

generate()

```

---

## 🐛 3. Troubleshooting / Masalah yang Dihadapi

**Error 1: Salah Pemilihan Model (Wrong Model Selection)**

* **Pesan Error:** *Biasanya tidak muncul error eksplisit, tapi "Check My Progress" akan GAGAL (masih merah) atau hasil output tidak sesuai ekspektasi.*
* **Penyebab:** Pada langkah awal di Vertex AI Studio atau di dalam kode Python, model yang dipilih adalah `gemini-pro`, `gemini-1.5-flash`, atau `gemini-2.0-flash-thinking` (default system).
* **Solusi:** Instruksi lab secara spesifik mewajibkan model **`gemini-2.5-flash`**.
* **Di GUI:** Pastikan dropdown "Model" di kiri atas dipilih ke `gemini-2.5-flash`.
* **Di Python:** Pastikan variabel `model = "gemini-2.5-flash"`. Jangan gunakan versi lain meskipun terlihat lebih baru.



**Error 2: ClientError 400 INVALID_ARGUMENT (Thinking Level)**

* **Pesan Error:** `Unable to submit request because thinking_level is not supported by this model.`
* **Penyebab:** Kode Python hasil generate otomatis (Get Code) menyertakan parameter `thinking_config`. Karena kita wajib menggunakan `gemini-2.5-flash` (yang merupakan model standar, bukan model *reasoning*), fitur thinking ini tidak didukung.
* **Solusi:** Hapus manual blok `thinking_config=types.ThinkingConfig(...)` dari dalam konfigurasi `generate_content_config` di kode Python.

**Error 3: Check Progress Failed (Resource Check)**

* **Penyebab:**
1. **Salah Region:** Region default pada kode adalah `us-central1` atau kosong, sedangkan lab mewajibkan resource dibuat di region `europe-west4`.
2. **Salah Bucket:** URI Bucket pada kode contoh di notebook masih menggunakan placeholder/bucket lama (`gs://qwiklabs-gcp-xx...`), sehingga kode mengakses file yang salah atau tidak memiliki izin akses (403 Forbidden).


* **Solusi:**
1. Ubah inisialisasi client menjadi: `client = genai.Client(vertexai=True, location="europe-west4", ...)`
2. Ganti `file_uri` dengan nama bucket spesifik yang didapat di panel kiri saat lab dimulai (misal: `qwiklabs-gcp-03-ea9bf4e3f5d2-bucket`).



---

## 📝 4. Catatan Penting (Key Takeaways)

Hal-hal baru yang saya pelajari dari lab ini:

1. **Ketelitian Memilih Model:** Memilih versi model yang tepat (`gemini-2.5-flash`) sangat krusial. Versi model yang berbeda memiliki kapabilitas parameter yang berbeda (contoh: support `thinking_config`).
2. **Region Sensitivity:** Di lingkungan Vertex AI, region pada Client SDK Python harus sama persis dengan region di mana resource (seperti prompt yang disimpan) berada.
3. **Auth Management:** Saat menggunakan Vertex AI Workbench (JupyterLab) di dalam Google Cloud, kita tidak perlu menggunakan API Key. Praktik terbaiknya adalah menggunakan `PROJECT_ID` dan kredensial environment (ADC) yang sudah terintegrasi.
4. **Temperature Parameter:** Mengubah nilai `temperature` (misal ke 2.0) secara drastis mempengaruhi kreativitas output model, membuatnya lebih variatif dan "liar" dibanding nilai default (1.0).

---

[⬅️ Kembali ke Menu Utama](../README.md)