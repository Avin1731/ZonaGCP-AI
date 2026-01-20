# ☁️ Build Real World AI Applications with Gemini and Imagen: Challenge Lab

**Rute:** Generative AI on Vertex AI
**Topik:** Vertex AI SDK, Gemini 2.5 Flash, Imagen 3, Chat Prompts, Multi-modal
**Tanggal Pengerjaan:** 21 Januari 2026

---

## 📂 Part 1: Build an AI Image Recognition app using Gemini on Vertex AI

### 🎯 1. Overview & Tujuan

Lab ini menggunakan Vertex AI Python SDK untuk mengirim gambar ke model Gemini dan meminta deskripsi teks.

### 🛠️ 2. Langkah-Langkah & Solusi

#### 🔹 Persiapan Environment

```bash
pip install --upgrade google-genai

```

#### 🔹 Implementasi Kode (`genai.py`)

*Catatan: Kita menggunakan konfigurasi Client eksplisit untuk menghindari error pembacaan Env Variable.*

```python
from google import genai
from google.genai.types import HttpOptions, Part

# --- SETUP CLIENT ---
# Masukkan parameter project dan location secara eksplisit
# Ganti project ID di bawah sesuai dengan ID lab saat pengerjaan
client = genai.Client(
    vertexai=True,
    project="qwiklabs-gcp-02-c00eecf44d38", 
    location="us-central1",
    http_options=HttpOptions(api_version="v1")
)

# --- PANGGIL MODEL ---
response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents=[
        "What is shown in this image?",
        Part.from_uri(
            file_uri="https://storage.googleapis.com/cloud-samples-data/generative-ai/image/scones.jpg",
            mime_type="image/jpeg",
        ),
    ],
)

# --- PRINT HASIL ---
print(response.text)

```

#### 🔹 Eksekusi

```bash
python3 genai.py

```

---

## 📂 Part 2: Build an AI Image Generator app using Imagen on Vertex AI

### 🎯 1. Overview & Tujuan

Lab ini menggunakan model **Imagen 3** untuk mengubah teks prompt menjadi file gambar (`.jpeg`).

### 🛠️ 2. Langkah-Langkah & Solusi

#### 🔹 Persiapan Environment

```bash
pip install --upgrade google-cloud-aiplatform

```

#### 🔹 Implementasi Kode (`GenerateImage.py`)

```python
import argparse
import vertexai
from vertexai.preview.vision_models import ImageGenerationModel

def generate_image(
    project_id: str, location: str, output_file: str, prompt: str
) -> vertexai.preview.vision_models.ImageGenerationResponse:
    """Generate an image using a text prompt.
    Args:
      project_id: Google Cloud project ID, used to initialize Vertex AI.
      location: Google Cloud region, used to initialize Vertex AI.
      output_file: Local path to the output image file.
      prompt: The text prompt describing what you want to see."""

    # 1. Inisialisasi Vertex AI
    vertexai.init(project=project_id, location=location)

    # 2. Load Model Imagen 3 (imagen-3.0-generate-002)
    model = ImageGenerationModel.from_pretrained("imagen-3.0-generate-002")

    # 3. Generate Gambar
    images = model.generate_images(
        prompt=prompt,
        # Optional parameters
        number_of_images=1,
        seed=1,
        add_watermark=False,
    )

    # 4. Simpan Gambar ke Output File
    images[0].save(location=output_file)

    return images

# --- EKSEKUSI FUNGSI ---
# Pastikan Project ID sesuai dengan lab session
generate_image(
    project_id='qwiklabs-gcp-02-d20024a85259',
    location='us-central1',
    output_file='image.jpeg',
    prompt='Create an image of a cricket ground in the heart of Los Angeles',
)

```

#### 🔹 Eksekusi

```bash
python3 GenerateImage.py

```

---

## 📂 Part 3: Build an application to send Chat Prompts using the Gemini model

### 🎯 1. Overview & Tujuan

Lab ini mengajarkan pembuatan Chatbot. **Penting:** Lab ini mewajibkan penggunaan library `google-cloud-logging` untuk penilaian otomatis (grading).

### 🛠️ 2. Langkah-Langkah & Solusi

#### 🔹 Persiapan Environment

```bash
pip install --upgrade google-genai google-cloud-logging

```

#### 🔹 Task 1: Chat Tanpa Stream (`SendChatwithoutStream.py`)

```python
from google import genai
from google.genai.types import HttpOptions, ModelContent, Part, UserContent
import logging
from google.cloud import logging as gcp_logging

# ------  Below cloud logging code is for Qwiklab's internal use --------
# Initialize GCP logging
gcp_logging_client = gcp_logging.Client()
gcp_logging_client.setup_logging()

# Setup Client
client = genai.Client(
    vertexai=True,
    project='qwiklabs-gcp-04-42ba5674b9d0', # Ganti sesuai Project ID Lab
    location='europe-west4',
    http_options=HttpOptions(api_version="v1")
)

# Create Chat Session with History
chat = client.chats.create(
    model="gemini-2.5-flash",
    history=[
        UserContent(parts=[Part(text="Hello")]),
        ModelContent(
            parts=[Part(text="Great to meet you. What would you like to know?")],
        ),
    ],
)

# Send Messages
response = chat.send_message("What are all the colors in a rainbow?")
print(response.text)

response = chat.send_message("Why does it appear when it rains?")
print(response.text)

```

#### 🔹 Task 2: Chat Dengan Stream (`SendChatwithStream.py`)

```python
from google import genai
from google.genai.types import HttpOptions
import logging
from google.cloud import logging as gcp_logging

# ------  Below cloud logging code is for Qwiklab's internal use --------
# Initialize GCP logging
gcp_logging_client = gcp_logging.Client()
gcp_logging_client.setup_logging()

# Setup Client
client = genai.Client(
    vertexai=True,
    project='qwiklabs-gcp-04-42ba5674b9d0', # Ganti sesuai Project ID Lab
    location='europe-west4',
    http_options=HttpOptions(api_version="v1")
)

# Create Chat Session
chat = client.chats.create(model="gemini-2.5-flash")

# Send Message with Streaming
response_text = ""
for chunk in chat.send_message_stream("What are all the colors in a rainbow?"):
    print(chunk.text, end="")
    response_text += chunk.text

```

#### 🔹 Eksekusi

```bash
python3 SendChatwithoutStream.py
python3 SendChatwithStream.py

```

---

## 📂 Part 4: Build a Multi-Modal GenAI Application: Challenge Lab

### 🎯 1. Overview & Tujuan

Challenge Lab ini menggabungkan Imagen (untuk membuat gambar) dan Gemini (untuk membaca gambar). Aplikasi akan membuat gambar buket bunga, lalu meminta Gemini membuatkan ucapan ulang tahun berdasarkan gambar tersebut.

### 🛠️ 2. Langkah-Langkah & Solusi

#### 🔹 Persiapan Environment (VS Code)

```bash
python3 -m pip install --upgrade google-genai

```

#### 🔹 Implementasi Kode Gabungan (`main.py`)

Kode ini menggunakan `from_bytes` untuk mengirim data gambar ke Gemini (sesuai update SDK terbaru) dan membaca `keys.json` untuk autentikasi otomatis di VS Code.

```python
import os
import json
from google import genai
from google.genai import types

# --- SETUP AUTENTIKASI ---
# Menggunakan file keys.json yang tersedia di environment lab
os.environ["GOOGLE_APPLICATION_CREDENTIALS"] = "keys.json"

# Mengambil Project ID dari file keys.json
with open("keys.json", "r") as f:
    keys = json.load(f)
    PROJECT_ID = keys["project_id"]

LOCATION = "us-central1" # Region standar untuk Imagen di lab ini

print(f"Menggunakan Project ID: {PROJECT_ID}")
client = genai.Client(vertexai=True, project=PROJECT_ID, location=LOCATION)

# --- TASK 1: GENERATE IMAGE FUNCTION ---
def generate_bouquet_image(prompt):
    print(f"Sedang membuat gambar dengan prompt: {prompt}...")
    try:
        # Memanggil model Imagen 3 (imagen-3.0-generate-002)
        response = client.models.generate_images(
            model='imagen-3.0-generate-002',
            prompt=prompt,
            config=types.GenerateImagesConfig(
                number_of_images=1,
                aspect_ratio="1:1"
            )
        )
        
        # Menyimpan hasil gambar ke file lokal
        if response.generated_images:
            image_bytes = response.generated_images[0].image.image_bytes
            output_file = "bouquet.png"
            with open(output_file, "wb") as f:
                f.write(image_bytes)
            print(f"Sukses! Gambar disimpan di: {output_file}")
            return output_file
        else:
            print("Tidak ada gambar yang dihasilkan.")
            return None
            
    except Exception as e:
        print(f"Error generate gambar: {e}")
        return None

# --- TASK 2: ANALYZE IMAGE FUNCTION ---
def analyze_bouquet_image(image_path):
    if not image_path:
        print("Tidak ada file gambar untuk dianalisis.")
        return

    print(f"\nMenganalisis gambar di path: {image_path}...")
    
    try:
        # Membaca file gambar sebagai bytes
        with open(image_path, "rb") as f:
            img_bytes = f.read()
            
        text_prompt = "Generate birthday wishes based on the image passed"
        
        print("Respon Gemini (Streaming):")
        
        # Memanggil model Gemini 2.5 Flash dengan streaming
        # PENTING: Menggunakan from_bytes, bukan from_data untuk SDK terbaru
        for chunk in client.models.generate_content_stream(
            model='gemini-2.5-flash',
            contents=[
                types.Content(
                    role="user",
                    parts=[
                        types.Part.from_text(text=text_prompt),
                        types.Part.from_bytes(data=img_bytes, mime_type="image/png")
                    ]
                )
            ]
        ):
            print(chunk.text, end="")
            
    except Exception as e:
        print(f"Error analisis gambar: {e}")

# --- EKSEKUSI UTAMA ---
if __name__ == "__main__":
    # Eksekusi Task 1: Generate Gambar
    path = generate_bouquet_image("Create an image containing a bouquet of 2 sunflowers and 3 roses.")
    
    # Eksekusi Task 2: Analisis Gambar (jika generate berhasil)
    if path:
        analyze_bouquet_image(path)

```

#### 🔹 Eksekusi

```bash
python3 main.py

```

---

### 🐛 3. Troubleshooting Umum (Rekap Masalah & Solusi)

1. **Error:** `ModuleNotFoundError`
* **Penyebab:** Library terinstall di Python 2, tapi kita pakai Python 3.
* **Solusi:** Gunakan `python3 -m pip install ...` dan jalankan script dengan `python3 namafile.py`.


2. **Error:** `ValueError: Missing key inputs`
* **Penyebab:** Environment variables Project ID/Region tidak terbaca oleh SDK.
* **Solusi:** Masukkan `project` dan `location` secara hardcode di dalam `genai.Client()`.


3. **Error:** `AttributeError: type object 'Part' has no attribute 'from_data'`
* **Penyebab:** SDK `google-genai` versi terbaru mengubah nama fungsi.
* **Solusi:** Ganti `from_data` menjadi `from_bytes` saat mengupload gambar raw.



---

[⬅️ Kembali ke Menu Utama](../README.md)