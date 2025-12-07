# Jamur Bot – Voice Kasir Telegram dengan AI & Live Stock

## 🧩 Problem

UMKM/toko kecil sering kesulitan:
- Mencatat transaksi dengan rapi.
- Menghitung total belanja dari banyak item.
- Mengupdate stok secara real-time ke sistem/website.

Semua ini biasanya masih manual (tulis di buku / kalkulator).

## 💡 Solution

**Jamur Bot** adalah kasir berbasis **Telegram + voice**:

1. Pembeli/penjaga kasir cukup **mengucapkan** pesanan (contoh:  
   `beli roti 2, wafer 3, teh botol 1`).
2. Bot:
   - Transcribe suara → teks (Google Gemini).
   - Parse teks → daftar item + qty.
   - Cek harga & stok di **Google Sheets** (daftar harga).
   - Hitung subtotal & total.
   - Kirim **QR pembayaran** + ringkasan belanja.
   - Setelah “pembayaran diterima”, bot:
     - Menghapus pesan QR.
     - Mengupdate stok di Google Sheets.
     - Mengirim pesan terima kasih.

3. Website menampilkan **stok live** berdasarkan Sheet yang sama, sehingga pelanggan bisa lihat ketersediaan barang.

## 🚀 Tech Stack

- **n8n** sebagai orkestrator workflow.
- **Telegram Bot** sebagai antarmuka pengguna.
- **Google Gemini 2.5 Flash** untuk:
  - Transcribe voice.
  - Parsing teks menjadi JSON item belanja.
- **Google Sheets**:
  - Sheet `Daftar Harga` (harga & stok, dengan kolom `lower` untuk pencocokan case-insensitive).
  - Sheet `Cashflow` (riwayat transaksi).
- **QR Server API** untuk generate QR pembayaran.
- **Static Web (HTML/JS)** untuk menampilkan stok live dari Sheet.

## 🏗️ Arsitektur Singkat

1. **User → Telegram Bot (Voice)**  
   Voice ditangkap oleh `Telegram Trigger` dan difilter oleh node `If` agar hanya voice yang diproses.

2. **Download & Transcribe**  
   Node `Get a file` + `HTTP Request` mengunduh file voice, `Code in JS` mengatur MIME `audio/ogg`, lalu `Transcribe a recording` (Gemini) menghasilkan teks. :contentReference[oaicite:8]{index=8}  

3. **AI Parsing Order**  
   Node `Message a model` (Gemini) mengubah teks menjadi JSON `items` (nama produk + qty). Node `Items Parsed` memecahnya menjadi banyak item. :contentReference[oaicite:9]{index=9}  

4. **Lookup Harga & Hitung Total**  
   Node `Get row(s) in sheet` mengambil harga dari sheet `Daftar Harga` berdasarkan kolom `lower`, kemudian `Merge` dan `Build Caption` menghitung subtotal, total, menghasilkan:
   - `caption` (untuk Telegram).
   - `sheet_caption` (untuk Cashflow).
   - `grand_total` (nilai pembayaran). :contentReference[oaicite:10]{index=10}  

5. **QR + Flow Pembayaran**  
   - Node `HTTP Request1` membuat QR dari `grand_total`.  
   - `Kirim QR` mengirim gambar QR + caption ke Telegram.  
   - `Menunggu` + `Wait` + `Pembayaran Berhasil` + `Delete ...` mensimulasikan alur pembayaran dan menghapus pesan yang tidak relevan. :contentReference[oaicite:11]{index=11}  

6. **Pencatatan Cashflow**  
   Node `Append or update row in sheet` menulis ke sheet `Cashflow` (tanggal, nomor nota, total, dan detail pesanan). :contentReference[oaicite:12]{index=12}  

7. **Update Stok Live**  
   - Node `Split Lines` membagi item untuk stok.  
   - Node `Sheet Stock` mengambil stok lama per produk.  
   - `Code in JavaScript1` menghitung `stok_baru`.  
   - Node `Update row in sheet` menyimpan stok baru ke `Daftar Harga`. :contentReference[oaicite:13]{index=13}  

8. **Website Stok**  
   Folder `web/` berisi halaman yang membaca Sheet/endpoint untuk menampilkan stok terkini.

Detail arsitektur dan diagram lengkap ada di [`docs/architecture.md`](./docs/architecture.md).

## ✨ Fitur Utama

- Input **voice** → AI → otomatis jadi transaksi.
- Parsing natural language pesanan (bahasa Indonesia santai).
- Lookup harga & stok dari **sheet yang sama dengan website**.
- Generate QR pembayaran + flow simulasi pembayaran (menunggu, diterima, hapus QR).
- Pencatatan cashflow kasir otomatis di Google Sheets.
- Update stok real-time yang langsung tercermin di website.

## 📦 Cara Menjalankan

### 1. Prasyarat

- Akun:
  - Telegram Bot (BotFather).
  - Google Cloud / AI Studio (API key Gemini).
  - Google Sheets + Service Account (kalau pakai).
- n8n (Cloud/self-host).

### 2. Setup Environment

1. Duplikasi `.env.example` menjadi `.env` dan isi semua nilai:
   ```bash
   cp .env.example .env
