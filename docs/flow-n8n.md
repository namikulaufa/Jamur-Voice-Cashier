# Dokumentasi Alur n8n – Jamur Bot

Dokumen ini menjelaskan workflow n8n secara lebih detail, node per node, agar juri dan developer lain dapat memahami dan mengembangkan lebih lanjut. Workflow yang dimaksud berasal dari file `jamur bot` di repository ini.

---

## 1. Trigger & Validasi Input

### 1.1. `Telegram Trigger`
- Menerima semua update `message` dari bot Telegram.
- Output JSON memiliki struktur `message.chat`, `message.text`, `message.voice`, dsb. :contentReference[oaicite:13]{index=13}  

### 1.2. `If` – Hanya proses pesan voice
- Kondisi: `{{ $json.message.voice }}` **is not empty**.
- Cabang:
  - **True** → lanjut ke proses audio.
  - **False** → diarahkan ke `Send a text message`.

### 1.3. `Send a text message`
- Mengirim pesan: **“Harus berupa voice”** jika user mengirim teks/gambar/sticker.
- `appendAttribution` dimatikan agar tidak muncul footer otomatis n8n. :contentReference[oaicite:14]{index=14}  

---

## 2. Download & Transcribe Voice

### 2.1. `Get a file`
- Resource: `file` pada node Telegram.
- Mengambil `file_id` dari `message.voice.file_id`.
- Mendapatkan informasi `file_path` dari Telegram.

### 2.2. `HTTP Request` – Download audio
- URL: `https://api.telegram.org/file/bot<TELEGRAM_BOT_TOKEN>/{file_path}`  
  (di repo publik token diganti placeholder).
- Response format: file binary (property `data`). :contentReference[oaicite:15]{index=15}  

### 2.3. `Code in JavaScript`
- Mengubah MIME type binary dari `application/octet-stream` menjadi `audio/ogg` agar diterima Gemini.
- Script:
  - Loop semua `items`.
  - Jika `item.binary.data` ada, set `mimeType = 'audio/ogg'`. :contentReference[oaicite:16]{index=16}  

### 2.4. `Transcribe a recording`
- Node Google Gemini (Audio).
- Model: `gemini-2.5-flash`.
- Input: binary audio dari node sebelumnya.
- Output: teks di `content.parts[0].text`. :contentReference[oaicite:17]{index=17}  

---

## 3. Parsing Pesanan dengan Gemini

### 3.1. `Message a model`
- Node Google Gemini (Text).
- System message: asisten keuangan yang hanya mengembalikan JSON valid.
- User message berisi:
  - Teks hasil transkripsi.
  - Instruksi format output:
    ```json
    {
      "items": [
        { "nama": "nama produk", "qty": number }
      ]
    }
    ```
- `jsonOutput` diaktifkan agar output mudah diparsing.

### 3.2. `Items Parsed` (Code)
- Mengambil `content.parts[0].text` (string JSON) dari output Gemini.
- `JSON.parse` → object.
- Mengecek keberadaan `items` (array).
- Menghasilkan banyak item n8n, masing‑masing:
  ```json
  { "nama": "...", "qty": 2 }
  ``` :contentReference[oaicite:18]{index=18}  

---

## 4. Lookup Harga & Perhitungan Transaksi

### 4.1. `Get row(s) in sheet`
- Node Google Sheets.
- Target: sheet **Daftar Harga**.
- Filter:
  - `lookupColumn = "lower"`.
  - `lookupValue = {{ $json["nama"] }}`.
- Karena kolom `lower` diisi dengan `LOWER(Nama Produk)`, pencarian jadi case-insensitive. :contentReference[oaicite:19]{index=19}  

### 4.2. `Merge`
- Mode: `combineByPosition`.
- Input 1: `Items Parsed` (nama, qty).
- Input 2: `Get row(s) in sheet` (Nama Produk, Harga, Stok, lower, row_number).
- Output item memiliki kombinasi:
  ```json
  {
    "nama": "roti",
    "qty": 2,
    "Nama Produk": "roti",
    "Harga": 4000,
    "Stok": 60,
    "row_number": 2
  }
