
---

## `stock-sync.md`

```markdown
# Sinkronisasi Stok – Jamur Bot & Website

Dokumen ini menjelaskan cara Jamur Bot mengelola stok produk secara live menggunakan Google Sheets, dan bagaimana stok tersebut dapat ditampilkan di website.

---

## 1. Desain Data Stok

### 1.1. Sheet `Daftar Harga`

Struktur kolom yang disarankan:

| Nama Produk | Harga | Stok | lower | row_number |
|-------------|-------|------|-------|------------|
| Roti        | 4000  | 60   | roti  | 2          |
| Wafer       | 2000  | 30   | wafer | 3          |
| Teh Botol   | 12000 | 35   | teh botol | 4    |

Penjelasan:

- **Nama Produk**  
  Nama asli produk yang akan dibaca manusia.

- **Harga**  
  Harga satuan produk.

- **Stok**  
  Jumlah stok terkini yang akan ditampilkan di website.

- **lower**  
  `=LOWER(A2)` – berisi nama produk dalam huruf kecil.  
  Dipakai untuk *case-insensitive lookup* (misalnya "roti", "ROTI", atau "Roti" tetap cocok). :contentReference[oaicite:25]{index=25}  

- **row_number**  
  Diisi otomatis oleh n8n Google Sheets node (`readOnly`), dan digunakan sebagai primary key ketika melakukan update row.

---

## 2. Alur Update Stok di n8n

Berikut alur node yang terlibat dalam sinkronisasi stok setelah transaksi berhasil:

1. **`Build Caption`**
   - Menghasilkan satu item berisi:
     - `lines`: array `{ nama, qty, harga, subtotal }`.
     - `grand_total`.
   - Contoh:
     ```json
     {
       "lines": [
         { "nama": "roti", "qty": 2, "harga": 4000,  "subtotal": 8000 },
         { "nama": "wafer","qty": 3, "harga": 2000,  "subtotal": 6000 }
       ],
       "grand_total": 14000
     }
     ```
     :contentReference[oaicite:26]{index=26}  

2. **`Split Lines` (Code)**
   - Mengubah struktur di atas menjadi banyak item:
     ```json
     { "nama": "roti",  "qty": 2 }
     { "nama": "wafer", "qty": 3 }
     ```
   - Script mengambil `items[0].json.lines` dan mem‑`map` ke item baru. :contentReference[oaicite:27]{index=27}  

3. **`Sheet Stock` (Google Sheets – Get row(s) in sheet)**
   - Target: sheet `Daftar Harga`.
   - Filter:
     - `lookupColumn = "lower"`.
     - `lookupValue = {{ $json["nama"] }}` (nama dari `Split Lines`, diharapkan sudah dalam lower-case).
   - Opsi `returnFirstMatch: true` memastikan satu baris per produk.
   - Output per item mengandung:
     - `Nama Produk`
     - `Harga`
     - `Stok`
     - `lower`
     - `row_number`. :contentReference[oaicite:28]{index=28}  

4. **`Code in JavaScript1`**
   - Menggabungkan hasil `Sheet Stock` dengan qty pembelian berdasarkan index:

     ```js
     for (let i = 0; i < items.length; i++) {
       const row = items[i].json;  // hasil Sheet Stock
       const stokLama = Number(row.Stok || 0);
       const qtyBeli  = Number($item(i).$node["Split Lines"].json.qty || 0);
       const stokBaru = stokLama - qtyBeli;
       ...
     }
     ```

   - Hasil akhir item:
     ```json
     {
       "nama": "roti",
       "stok_lama": 60,
       "qty_beli": 2,
       "stok_baru": 58
     }
     ```

5. **`Update row in sheet`**
   - Operation: `update`.
   - Matching column: `row_number`.
   - `row_number` diisi dari output `Sheet Stock`.
   - Kolom yang di-update:
     - `Stok = {{ $json["stok_baru"] }}`.
   - Dengan cara ini, hanya baris produk yang relevan yang berubah stoknya. :contentReference[oaicite:29]{index=29}  

---

## 3. Integrasi dengan Website Live Stock

Ada beberapa opsi integrasi:

### 3.1. Website Membaca Sheet Langsung

Paling sederhana:

1. Publish Google Sheet sebagai **web** / CSV.
2. Front-end (misalnya `web/index.html`) melakukan `fetch` ke URL publik.
3. `script.js`:
   - Parse data sheet.
   - Render tabel stok dengan kolom Nama Produk & Stok.
   - Tambahkan logika UI:
     - Jika `stok <= 0` → tampilkan “Habis”.
     - Jika `stok < 5` → highlight merah sebagai “Hampir habis”.

### 3.2. Backend Kecil Sebagai Proxy (Opsional)

Untuk keamanan lebih baik:

1. Buat endpoint kecil (Node.js/Express / Cloud Function) yang:
   - Membaca data dari Google Sheets menggunakan service account.
   - Mengembalikan JSON ke frontend.
2. Website memanggil endpoint tersebut dan menampilkan stok.

Arsitektur ini menjaga konsistensi karena:
- n8n hanya menulis stok ke `Daftar Harga`.
- Website hanya membaca dari `Daftar Harga`.
- Tidak ada sumber data stok ganda.

---

## 4. Edge Cases & Catatan

- Jika produk diucapkan tapi tidak ditemukan di `Daftar Harga`:
  - Bisa ditandai error di node Code (misalnya menambahkan field `error`).
  - Ke depan bisa ditambahkan log khusus “produk tidak dikenal”.

- Untuk menghindari stok negatif:
  - Pada `Code in JavaScript1` bisa menggunakan:
    ```js
    const stokBaru = Math.max(stokLama - qtyBeli, 0);
    ```
- Stok menjadi sumber kebenaran untuk:
  - Website live stock.
  - Keputusan kasir (misal: kalau stok 0, bot bisa memperingatkan di masa depan).

Dengan desain ini, Jamur Bot tidak hanya menjadi kasir voice, tetapi juga sistem inventori ringan yang selalu sinkron dengan website.

