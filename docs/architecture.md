# Arsitektur Jamur Bot – Voice Kasir Telegram + Live Stock

Dokumen ini menjelaskan arsitektur sistem Jamur Bot dari sudut pandang komponen, aliran data, dan alasan pemilihan teknologi.

---

## 1. Gambaran Umum

Jamur Bot adalah kasir berbasis **Telegram + voice** yang:

1. Menerima voice message dari pengguna.
2. Menggunakan **Google Gemini** untuk:
   - Transcribe audio → teks.
   - Parsing teks → daftar item belanja (produk + qty).
3. Mengambil harga & stok produk dari **Google Sheets**.
4. Menghitung subtotal dan grand total, lalu membuat:
   - QR code untuk pembayaran.
   - Ringkasan transaksi untuk Telegram & pencatatan cashflow.
5. Meng-update stok produk di sheet sehingga dapat ditampilkan live di website.

Seluruh orkestrasi dilakukan menggunakan **n8n**.

---

## 2. Komponen Utama

### 2.1. Antarmuka Pengguna

- **Telegram Bot**
  - Menerima input berupa voice message.
  - Mengirimkan:
    - Pesan error jika input bukan voice.
    - QR pembayaran + ringkasan belanja.
    - Notifikasi menunggu pembayaran, pembayaran berhasil, dan ucapan terima kasih.
  - Pesan tertentu (QR & status pembayaran) dihapus kembali agar chat tetap bersih. :contentReference[oaicite:1]{index=1}  

- **Website Live Stock (Front-end)**
  - Menampilkan stok terkini setiap produk.
  - Membaca data stok dari Google Sheets atau dari API backend yang bersumber pada sheet yang sama.

### 2.2. Orkestrator

- **n8n Workflow**
  - Menangani seluruh alur:
    - Trigger dari Telegram.
    - Integrasi dengan Gemini.
    - Integrasi dengan Google Sheets.
    - Integrasi dengan layanan QR code.
    - Logika bisnis (perhitungan, filtering, update stok, dsb.). :contentReference[oaicite:2]{index=2}  

### 2.3. Layanan AI

- **Google Gemini 2.5 Flash**
  - Node `Transcribe a recording`: konversi audio (voice Telegram) menjadi teks.
  - Node `Message a model`: konversi teks natural language menjadi JSON terstruktur:
    ```json
    {
      "items": [
        { "nama": "roti", "qty": 2 },
        ...
      ]
    }
    ```
  - Dipilih karena:
    - Mendukung input audio secara langsung.
    - Mampu memahami bahasa Indonesia dengan baik.
    - Cocok untuk skenario prompt‑based tanpa perlu training khusus.   

### 2.4. Basis Data Ringan

- **Google Sheets**
  - Sheet **Daftar Harga**:
    - Menyimpan data master produk (nama, harga, stok, kolom `lower` untuk pencarian case-insensitive, dan `row_number` untuk update).
  - Sheet **Cashflow**:
    - Menyimpan riwayat transaksi (Nomor nota, Tanggal, Total, Detail).
  - Dipilih karena:
    - Mudah dioperasikan oleh pemilik toko.
    - Langsung bisa diakses website untuk live stock.
    - Integrasi native dengan n8n. :contentReference[oaicite:4]{index=4}  

### 2.5. Layanan Pendukung

- **QR Code API**
  - Node `HTTP Request1` memanggil endpoint `api.qrserver.com` untuk menghasilkan QR code berdasarkan `grand_total`. :contentReference[oaicite:5]{index=5}  

---

## 3. Aliran Data Tingkat Tinggi

Berikut alur *happy path* tinggi:

1. **Voice masuk**
   - `Telegram Trigger` menerima update.
   - Node `If` mengecek apakah payload memiliki `message.voice`. Jika tidak, cabang false mengirim pesan **“Harus berupa voice”** dan flow selesai. :contentReference[oaicite:6]{index=6}  

2. **Download & Transcribe**
   - `Get a file` mengambil `file_id` dari voice Telegram.
   - `HTTP Request` mengunduh file audio dari Telegram CDN.
   - `Code in JavaScript` mengubah MIME type menjadi `audio/ogg`.
   - `Transcribe a recording` (Gemini) menghasilkan teks perintah. :contentReference[oaicite:7]{index=7}  

3. **AI Parsing Order**
   - `Message a model` memberikan prompt ke Gemini untuk mengubah teks menjadi JSON `items`.
   - `Items Parsed` (Code) memecah JSON menjadi beberapa item n8n dengan field `nama` dan `qty`. :contentReference[oaicite:8]{index=8}  

4. **Lookup Harga & Hitung Total**
   - `Get row(s) in sheet` mencari record di sheet `Daftar Harga` berdasarkan kolom `lower` (nama produk lower-case).
   - `Merge` menggabungkan data item (nama, qty) dengan data sheet (nama produk, harga).
   - `Build Caption`:
     - Menghitung `subtotal` per produk dan `grand_total`.
     - Menyusun `caption` untuk Telegram dan `sheet_caption` untuk cashflow. :contentReference[oaicite:9]{index=9}  

5. **QR & Flow Pembayaran**
   - `HTTP Request1` membuat QR code berdasarkan `grand_total`.
   - `Kirim QR` mengirim foto QR + caption ke pengguna.
   - `Menunggu` mengirim pesan “Menunggu Pembayaran”.
   - `Wait` dan `Wait1/2` membuat jeda, lalu:
     - `Pembayaran Berhasil` mengirim konfirmasi (simulasi).
     - `Delete Menunggu`, `Delete Kirim QR`, `Delete Pembayaran Berhasil` membersihkan pesan status.
     - `Syukron` mengirim ucapan terima kasih. :contentReference[oaicite:10]{index=10}  

6. **Pencatatan Cashflow**
   - `Append or update row in sheet` menulis transaksi ke sheet `Cashflow` menggunakan nilai `grand_total` dan `sheet_caption`. :contentReference[oaicite:11]{index=11}  

7. **Update Stok**
   - `Split Lines` memecah `lines` (produk + qty) menjadi satu item per produk.
   - `Sheet Stock` menarik data stok lama & row number per produk dari `Daftar Harga`.
   - `Code in JavaScript1` menghitung `stok_baru = stok_lama - qty`.
   - `Update row in sheet` menyimpan `stok_baru` ke kolom `Stok` dengan kunci `row_number`. :contentReference[oaicite:12]{index=12}  

---

## 4. Keputusan Desain Penting

1. **Telegram + Voice vs. UI Web**
   - Target use case: kasir/owner yang multitasking dan lebih natural bicara daripada mengetik → voice cocok.
   - Telegram mudah diakses dari HP murah.

2. **Google Sheets sebagai “database”**
   - Pemilik toko sudah familiar.
   - Memudahkan integrasi dengan website sederhana (read‑only) tanpa DB server terpisah.

3. **AI di dua tahap (Transcribe + NLP)**
   - Memisahkan **ASR** (Automatic Speech Recognition) dan **NLP** (parsing order) membuat prompt lebih fleksibel dan mudah di-tuning.

4. **Stock-aware Architecture**
   - Semua pembelian langsung mengurangi stok di `Daftar Harga` sehingga:
     - Web stok selalu konsisten.
     - Tidak perlu sinkronisasi tambahan.

---

## 5. Potensi Pengembangan

- Multi-toko / multi-user dengan penambahan kolom `store_id`.
- Penambahan diskon/promo/ongkir dalam perhitungan.
- Integrasi dengan payment gateway yang real (mengganti bagian simulasi).
- Backend kecil (Node/Go) untuk menyediakan API stok yang lebih aman ke website.


