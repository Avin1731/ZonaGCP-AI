# ☁️ Enhance Gemini Model Capabilities: Challenge Lab

**Rute:** Generative AI on Vertex AI
**Topik:** Gemini API, Vertex AI Workbench, Function Calling, Grounding, JSON Schema
**Tanggal Pengerjaan:** 24 Januari 2026

---

## 🎯 1. Overview & Tujuan

Lab ini menantang kemampuan kita untuk memperluas kapabilitas model **Gemini 2.5 Flash** menggunakan Vertex AI SDK for Python. Tujuannya adalah membantu "Cymbal Direct" (perusahaan ritel fiktif) melakukan analisis harga dan produk secara otomatis.

**Misi Utama:**

1. **Code Execution:** Menginstruksikan Gemini untuk menulis dan menjalankan kode Python (menghitung rata-rata harga).
2. **Grounding:** Menggunakan Google Search Tool agar Gemini bisa mencari informasi produk terbaru (Nike Air Jordan XXXVI) dari internet.
3. **Controlled Generation (JSON Schema):** Memaksa output Gemini menjadi format JSON yang rapi untuk data harga kompetitor, sekaligus menangani *Rate Limit* saat melakukan looping request.

---

## 🛠️ 2. Langkah-Langkah & Solusi

### 🔹 Task 1: Setup & Import Libraries

* **Deskripsi:** Membuka Jupyter Notebook di Vertex AI Workbench, menginstal library `google-genai`, dan melakukan inisialisasi Client.
* **Langkah:**
1. Buka **Vertex AI > Workbench**.
2. Klik **Open JupyterLab** pada instance yang tersedia.
3. Buka file `enhance-gemini-model-capabilities.ipynb`.
4. Jalankan sel instalasi (`pip install`) dan restart kernel.
5. Gunakan kode berikut untuk inisialisasi Client (sesuaikan Project ID):



```python
import os
from google import genai

# Hardcode Project ID biar aman
PROJECT_ID = "qwiklabs-gcp-00-939b23d3cf19" 
LOCATION = os.environ.get("GOOGLE_CLOUD_REGION", "global")

# Inisialisasi Client
client = genai.Client(vertexai=True, project=PROJECT_ID, location=LOCATION)
MODEL_ID = "gemini-2.5-flash"

```

### 🔹 Task 2: Code Execution with Gemini

* **Deskripsi:** Menggunakan `ToolCodeExecution` agar Gemini bisa menghitung rata-rata harga sneaker menggunakan Python, bukan menghitung manual (yang sering salah).
* **Kode Notebook (Cell Task 2):**

```python
from google.genai.types import GenerateContentConfig, Tool, ToolCodeExecution
from IPython.display import display, Markdown

# 1. Define the code execution tool
# Mengaktifkan fitur Code Execution (Sandbox Python)
code_execution_tool = Tool(
    code_execution=ToolCodeExecution()
)

sneaker_prices = [120, 150, 110, 180, 135, 95, 210, 170, 140, 165] 

# 2. Define the prompt
# Masukkan data list harga langsung ke dalam prompt
PROMPT = f"""Given the following sneaker prices: {sneaker_prices}, calculate the average price.
Generate and run code for the calculation."""

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents=PROMPT,
    config=GenerateContentConfig(
        tools=[code_execution_tool],
        temperature=0,
    ),
)

# Menampilkan Output Kode & Hasil
for part in response.candidates[0].content.parts:
    if part.executable_code:
        display(Markdown(f"```py\n{part.executable_code.code}\n```"))

for part in response.candidates[0].content.parts:
    if part.code_execution_result:
        display(Markdown(f"`{part.code_execution_result.output}`"))
        print("\nOutcome:", part.code_execution_result.outcome)

```

### 🔹 Task 3: Grounding with Google Search

* **Deskripsi:** Menggunakan `GoogleSearch` tool agar respon Gemini tentang "Nike Air Jordan XXXVI" akurat dan berdasarkan data web terkini.
* **Kode Notebook (Cell Task 3):**

```python
from google.genai.types import GenerateContentConfig, GoogleSearch, Tool

# 1. Define the Google Search tool
google_search_tool = Tool(
    google_search=GoogleSearch()
)

# 2. Define the prompt
prompt = "What are the key features of the Nike Air Jordan XXXVI?"

# 3. Generate response with Grounding
response = client.models.generate_content(
    model=MODEL_ID,
    contents=prompt,
    config=GenerateContentConfig(
        tools=[google_search_tool], # Wajib ada ini buat grounding
    ),
)

print(response.text)

```

### 🔹 Task 4: Extract Competitor Pricing (JSON Schema)

* **Deskripsi:** Looping mencari harga sepatu di berbagai retailer dan memformat hasilnya jadi JSON. Kita harus **menghapus** tool `GoogleSearch` di sini karena konflik dengan JSON Schema, dan menambahkan `time.sleep` agar tidak kena limit.
* **Kode Notebook (Cell Task 4):**

```python
from google.genai.types import GenerateContentConfig, Tool
from IPython.display import Markdown, display
import json
import time  # <--- Penting buat jeda (rem)

# Definisi Data
sneaker_models = ["Under Armour Curry Flow 9", "Sketchers Slip-ins: Glide-Step Pro"]
retailers = ["Foot Locker", "Nordstrom"]
extracted_data = []

# Looping Request
for model in sneaker_models:
    for retailer in retailers:
        # Prompt dengan fallback value random biar gak return 0.00
        query = f"Find the price of the {model} at {retailer}. If the price found is 0.00 return a random value between $50 and $200. DO NOT return 0.00"

        try:
            # Generate Content dengan JSON Schema
            # Note: Tool GoogleSearch DIHAPUS di sini karena konflik dengan Controlled Generation
            response = client.models.generate_content(
                model=MODEL_ID,
                contents=query,
                config=GenerateContentConfig(
                    response_schema=response_schema, # Menggunakan schema yg didefinisikan sebelumnya
                    response_mime_type="application/json"
                )
            )
            print(response.text)
            
        except Exception as e:
            print(f"Error fetching data: {e}")

        # REM OTOMATIS: Jeda 10 detik biar gak kena Error 429
        print("Waiting 10 seconds to avoid Rate Limit...")
        time.sleep(10) 

```

---

## 🐛 3. Troubleshooting / Masalah yang Dihadapi

**Error 1:** `SyntaxError: incomplete input` atau `NameError`

* **Penyebab:** Copy-paste kode Python ke terminal Cloud Shell (layar hitam), padahal harusnya di Jupyter Notebook. Atau, copy-paste kode tidak lengkap (terpotong bagian bawahnya).
* **Solusi:** Pastikan kode dijalankan di dalam sel Notebook (`.ipynb`). Gunakan copy-paste blok kode secara utuh.

**Error 2:** `ClientError: 400 INVALID_ARGUMENT ... controlled generation is not supported with Search tool`

* **Penyebab:** Di Task 4, kita mencoba menggunakan `tools=[google_search_tool]` BERSAMAAN dengan `response_schema` (JSON). Gemini saat ini belum mendukung gabungan keduanya secara langsung.
* **Solusi:** Hapus baris `tools=[google_search_tool]` pada konfigurasi Task 4. Fokus pada formatting JSON saja.

**Error 3:** `ClientError: 429 RESOURCE_EXHAUSTED`

* **Penyebab:** Mengirim request terlalu cepat dalam `for loop` di Task 4, sehingga kuota API per menit (QPM) habis.
* **Solusi:** Tambahkan `import time` dan `time.sleep(10)` di dalam loop untuk memberi jeda 10 detik antar request.

---

## 📝 4. Catatan Penting (Key Takeaways)

Hal-hal baru yang saya pelajari dari lab ini:

1. **Code Execution is Powerful:** Gemini lebih akurat dalam perhitungan matematika jika kita izinkan dia menulis dan menjalankan kode Python sendiri (`ToolCodeExecution`) daripada menyuruhnya menghitung manual ("hallucination prone").
2. **Grounding vs Controlled Generation:** Saat ini kita harus memilih salah satu antara mencari data live (Grounding) ATAU memformat output ketat (JSON Schema). Jika digabung bisa error 400.
3. **Rate Limiting:** Saat melakukan scripting/looping dengan API Generative AI, selalu ingat untuk menambahkan *delay* (`sleep`) agar akun tidak terblokir sementara karena *spamming request*.

---

[⬅️ Kembali ke Menu Utama](../README.md)