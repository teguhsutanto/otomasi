Pilihan yang sangat bagus\! **Automasi OCR (Optical Character Recognition)** adalah jembatan antara dunia fisik (kertas) dan digital. Ini adalah salah satu proyek yang paling memuaskan karena hasilnya langsung terasa manfaatnya.

Berikut adalah panduan lengkap dan kode *prototype* untuk membuat **Scanner Struk Belanja**.

-----

### 🛠️ Persiapan (Wajib)

Sebelum menulis kode, ada satu hal penting: Python butuh "mesin" untuk membaca tulisan. Kita menggunakan **Tesseract OCR**.

1.  **Install Software Tesseract:**
      * **Windows:** Download dan install installer-nya (cari "Tesseract at UB Mannheim" di Google). Ingat lokasi installnya (biasanya `C:\Program Files\Tesseract-OCR\tesseract.exe`).
      * **Mac/Linux:** Install lewat terminal (`brew install tesseract` atau `sudo apt-get install tesseract-ocr`).
2.  **Install Library Python:**
    Buka terminal/command prompt dan ketik:
    ```bash
    pip install pytesseract pillow pandas openpyxl
    ```

-----

### 💻 Kode Python: Struk ke Excel

Script ini akan membaca gambar struk, mencari kata kunci "TOTAL" dan "TANGGAL", lalu menyimpannya.

```python
import pytesseract
from PIL import Image
import pandas as pd
import re
import os

# --- KONFIGURASI ---
# Jika di Windows, arahkan ke file .exe tesseract yang sudah diinstall
# Hapus baris di bawah jika Anda menggunakan Mac/Linux
pytesseract.pytesseract.tesseract_cmd = r'C:\Program Files\Tesseract-OCR\tesseract.exe'

def ekstrak_data_struk(image_path):
    """
    Membaca gambar dan mengembalikan dictionary berisi Tanggal dan Total.
    """
    try:
        # 1. Buka Gambar
        img = Image.open(image_path)
        
        # 2. Konversi Gambar ke Teks (OCR Process)
        # Menggunakan bahasa Indonesia/Inggris
        raw_text = pytesseract.image_to_string(img)
        
        # print(raw_text) # Uncomment baris ini untuk debugging melihat hasil pembacaan mentah
        
        data = {'File': os.path.basename(image_path), 'Tanggal': None, 'Total': None}

        # 3. Mencari Pola Menggunakan Regex (Regular Expression)
        
        # Pola Tanggal (Mencari format DD-MM-YYYY atau DD/MM/YYYY)
        match_tanggal = re.search(r'(\d{2}[-/]\d{2}[-/]\d{4})', raw_text)
        if match_tanggal:
            data['Tanggal'] = match_tanggal.group(1)

        # Pola Total Harga (Mencari angka setelah kata Total/Jumlah)
        # Mencari baris yang mengandung 'Total' diikuti angka (misal: Total 50.000)
        # Penjelasan Regex: "Total" (abaikan huruf besar/kecil), diikuti karakter apapun, lalu angka
        match_total = re.search(r'(?i)(total|jumlah)[\s\W]*([\d\.,]+)', raw_text)
        if match_total:
            # Membersihkan simbol mata uang agar jadi angka murni
            angka_bersih = match_total.group(2).replace('.', '').replace(',', '').strip()
            data['Total'] = angka_bersih

        return data

    except Exception as e:
        print(f"Error pada file {image_path}: {e}")
        return None

# --- MAIN PROGRAM ---
# Anggap kita punya list gambar struk di folder
list_gambar = ['struk1.jpg', 'struk2.png'] # Ganti dengan nama file gambar asli Anda

hasil_akhir = []

print("Sedang memproses...")

for gambar in list_gambar:
    # Pastikan file gambarnya ada
    if os.path.exists(gambar):
        hasil = ekstrak_data_struk(gambar)
        if hasil:
            hasil_akhir.append(hasil)
            print(f"✅ Berhasil proses: {gambar}")
    else:
        print(f"❌ File tidak ditemukan: {gambar}")

# 4. Simpan ke Excel
if hasil_akhir:
    df = pd.DataFrame(hasil_akhir)
    df.to_excel("Laporan_Pengeluaran.xlsx", index=False)
    print("\n🎉 Selesai! Data tersimpan di 'Laporan_Pengeluaran.xlsx'")
else:
    print("\nTidak ada data yang berhasil diekstrak.")
```

-----

### 🔍 Penjelasan Logika "Otak" Program

Bagian tersulit dari program ini bukan membaca gambarnya, tapi **memahami teksnya**. Di sinilah peran **Regex (Regular Expression)**:

1.  **`pytesseract.image_to_string(img)`**:
    Ini mengubah gambar piksel menjadi string teks panjang yang berantakan.
2.  **Regex Tanggal `(\d{2}[-/]\d{2}[-/]\d{4})`**:
    Kode ini seperti detektif yang mencari pola: "Cari 2 digit angka, diikuti tanda strip atau garis miring, diikuti 2 digit, lalu 4 digit."
      * *Contoh:* Dia akan menangkap `18-11-2025`.
3.  **Regex Total `(?i)(total|jumlah)[\s\W]*([\d\.,]+)`**:
      * `(?i)`: Tidak peduli huruf besar/kecil (Total/TOTAL).
      * `(total|jumlah)`: Cari kata "Total" ATAU "Jumlah".
      * `[\d\.,]+`: Ambil angka-angka setelahnya (termasuk titik dan koma).

-----

### 🚧 Tantangan & Solusi

Saat Anda menjalankannya nanti, Anda mungkin menemui kendala ini:

  * **Gambar Buram/Gelap:** Tesseract akan gagal membaca.
      * *Solusi:* Gunakan library `OpenCV` untuk mengubah gambar jadi hitam-putih (thresholding) sebelum dibaca Tesseract agar tulisan lebih kontras.
  * **Format Struk Beda-beda:** Posisi "Total" di struk Indomaret beda dengan struk restoran.
      * *Solusi:* Regex harus dibuat lebih fleksibel atau spesifik per jenis vendor.

**Langkah Selanjutnya:**
Apakah Anda ingin saya jelaskan cara **menggunakan OpenCV** untuk memperjelas gambar struk yang buram agar hasil bacanya lebih akurat?
