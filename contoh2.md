Pilihan tepat\! Mengandalkan gambar mentah dari kamera HP seringkali membuat OCR gagal karena bayangan, kertas kusut, atau pencahayaan buruk.

Kita akan menggunakan **OpenCV** untuk melakukan *Pre-processing*. Tujuannya adalah mengubah gambar menjadi **Biner (Hitam Putih Murni)** agar mesin Tesseract bisa membaca teks dengan sangat jelas.

### 1\. Install Library Tambahan

Anda perlu menginstall library OpenCV untuk Python:

```bash
pip install opencv-python
```

### 2\. Kode Python: Integrasi OpenCV + Tesseract

Berikut adalah kode yang sudah di-upgrade. Saya menambahkan fungsi `perbaiki_kualitas_gambar` yang bertugas sebagai "filter penjernih" sebelum gambar dibaca.

```python
import cv2 # Library OpenCV
import pytesseract
from PIL import Image
import pandas as pd
import re
import os
import numpy as np

# --- KONFIGURASI TESSERACT (Windows) ---
# Hapus baris ini jika di Mac/Linux
pytesseract.pytesseract.tesseract_cmd = r'C:\Program Files\Tesseract-OCR\tesseract.exe'

def perbaiki_kualitas_gambar(image_path):
    """
    Menggunakan OpenCV untuk mengubah gambar menjadi hitam-putih kontras tinggi (Thresholding).
    """
    # 1. Baca gambar dengan OpenCV
    img = cv2.imread(image_path)
    
    if img is None:
        return None

    # 2. Ubah ke Grayscale (Abu-abu)
    # OCR bekerja lebih baik tanpa warna-warni
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)

    # 3. Adaptive Thresholding (Teknik Kunci!)
    # Ini mengubah gambar jadi Hitam & Putih murni, tapi cerdas menangani bayangan.
    # Parameter: (Source, MaxVal, AdaptiveMethod, ThresholdType, BlockSize, Constant)
    processed_img = cv2.adaptiveThreshold(
        gray, 
        255, 
        cv2.ADAPTIVE_THRESH_GAUSSIAN_C, 
        cv2.THRESH_BINARY, 
        11, 
        2
    )

    # 4. Denoising (Opsional - Menghapus bintik-bintik kecil/noise)
    # kernel = np.ones((1, 1), np.uint8)
    # processed_img = cv2.dilate(processed_img, kernel, iterations=1)
    # processed_img = cv2.erode(processed_img, kernel, iterations=1)

    # 5. Konversi kembali ke format yang dimengerti PIL/Tesseract
    return Image.fromarray(processed_img)

def ekstrak_data_struk(image_path):
    try:
        # --- STEP BARU: PRE-PROCESSING ---
        # Gambar diproses dulu lewat OpenCV sebelum dibaca
        img_bersih = perbaiki_kualitas_gambar(image_path)
        
        if img_bersih is None:
            return None

        # (Opsional) Simpan gambar hasil proses untuk dicek manusia
        # img_bersih.save(f"debug_{os.path.basename(image_path)}") 

        # --- PROSES OCR SEPERTI BIASA ---
        raw_text = pytesseract.image_to_string(img_bersih)
        
        data = {'File': os.path.basename(image_path), 'Tanggal': None, 'Total': None}

        # Regex Tanggal (Mencari DD-MM-YYYY atau DD/MM/YYYY)
        match_tanggal = re.search(r'(\d{2}[-/]\d{2}[-/]\d{4})', raw_text)
        if match_tanggal:
            data['Tanggal'] = match_tanggal.group(1)

        # Regex Total (Mencari kata Total/Jumlah/Grand Total diikuti angka)
        match_total = re.search(r'(?i)(total|jumlah|grand total)[\s\W]*([\d\.,]+)', raw_text)
        if match_total:
            clean_num = match_total.group(2).replace('.', '').replace(',', '').strip()
            data['Total'] = clean_num

        return data

    except Exception as e:
        print(f"Error: {e}")
        return None

# --- MAIN PROGRAM ---
list_gambar = ['struk_kusut.jpg', 'struk_gelap.png'] # Ganti nama file Anda

hasil_akhir = []
print("Mulai memindai dengan OpenCV Enhanced Mode...")

for gambar in list_gambar:
    if os.path.exists(gambar):
        hasil = ekstrak_data_struk(gambar)
        if hasil:
            hasil_akhir.append(hasil)
            print(f"✅ {gambar} : {hasil}")
    else:
        print(f"❌ File {gambar} tidak ada.")

if hasil_akhir:
    pd.DataFrame(hasil_akhir).to_excel("Laporan_Canggih.xlsx", index=False)
```

-----

### 💡 Apa yang Sebenarnya Dilakukan OpenCV?

Teknik yang saya gunakan di atas disebut **Adaptive Gaussian Thresholding**.

1.  **Grayscale:** Membuang informasi warna yang tidak perlu. Bagi komputer, teks merah di kertas putih sama saja dengan teks hitam di kertas putih.
2.  **Adaptive Thresholding:**
      * Jika kita pakai *Simple Threshold*, komputer memukul rata: "Semua piksel di atas nilai 127 jadi putih, di bawah jadi hitam". Ini **gagal total** jika ada bayangan tangan di atas struk (bagian bayangan akan jadi hitam semua).
      * **Adaptive Threshold** melihat area kecil di sekitar setiap huruf. Dia menghitung: "Apakah piksel ini lebih gelap dari tetangganya?". Jika ya, jadi hitam (teks). Jika tidak, jadi putih (kertas).
      * **Hasilnya:** Tulisan tetap terbaca jelas meskipun ada bagian kertas yang terkena bayangan atau pencahayaan tidak rata.

### Langkah Berikutnya yang Bisa Saya Lakukan

Agar program ini benar-benar otomatis, saya bisa bantu Anda menggabungkan skrip ini dengan **Ide No. 9 (File Organizer)**.

Jadi skenarionya:
**Setiap kali Anda meletakkan foto struk di folder "Input", script otomatis bangun, memprosesnya, update Excel, lalu memindahkan gambarnya ke folder "Sudah Diproses".**

Apakah Anda mau script **"Folder Watcher"** tersebut?
