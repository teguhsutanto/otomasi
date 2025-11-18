Ini adalah tahap akhir untuk membuat "robot" administrasi Anda. Kita akan menggunakan library bernama **`watchdog`**.

Sesuai namanya, library ini bertugas "menjaga anjing" (memantau) folder tertentu. Jika ada file baru masuk (Anda *drop* file gambar ke folder tersebut), dia akan langsung menyalak (menjalankan script OCR), mencatat ke Excel, lalu memindahkan file tersebut ke folder "Selesai".

### 1\. Install Library Watchdog

Buka terminal/command prompt:

```bash
pip install watchdog
```

### 2\. Kode Lengkap: Folder Watcher + OCR + Excel

Simpan kode ini dengan nama file, misalnya `robot_admin.py`.

Pastikan Anda membuat 2 folder di lokasi yang sama dengan script ini:

1.  Folder bernama **`Input_Struk`** (Tempat menaruh foto baru).
2.  Folder bernama **`Sudah_Diproses`** (Tempat file dipindahkan setelah selesai).

<!-- end list -->

```python
import time
import sys
import os
import shutil
import pandas as pd
import cv2
import pytesseract
import re
from PIL import Image
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

# --- KONFIGURASI ---
FOLDER_INPUT = "Input_Struk"
FOLDER_OUTPUT = "Sudah_Diproses"
FILE_EXCEL = "Laporan_Keuangan.xlsx"
TESSERACT_PATH = r'C:\Program Files\Tesseract-OCR\tesseract.exe' # Sesuaikan path

# Setup Tesseract
pytesseract.pytesseract.tesseract_cmd = TESSERACT_PATH

# --- FUNGSI PEMROSESAN GAMBAR (DARI DISKUSI SEBELUMNYA) ---
def perbaiki_kualitas_gambar(image_path):
    img = cv2.imread(image_path)
    if img is None: return None
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
    processed_img = cv2.adaptiveThreshold(
        gray, 255, cv2.ADAPTIVE_THRESH_GAUSSIAN_C, cv2.THRESH_BINARY, 11, 2
    )
    return Image.fromarray(processed_img)

def ekstrak_data_struk(image_path):
    try:
        img_bersih = perbaiki_kualitas_gambar(image_path)
        if img_bersih is None: return None
        
        raw_text = pytesseract.image_to_string(img_bersih)
        
        data = {'File': os.path.basename(image_path), 'Tanggal': '-', 'Total': '0'}
        
        # Regex
        match_tanggal = re.search(r'(\d{2}[-/]\d{2}[-/]\d{4})', raw_text)
        if match_tanggal: data['Tanggal'] = match_tanggal.group(1)
        
        match_total = re.search(r'(?i)(total|jumlah|grand total)[\s\W]*([\d\.,]+)', raw_text)
        if match_total:
            clean_num = match_total.group(2).replace('.', '').replace(',', '').strip()
            data['Total'] = clean_num
            
        return data
    except Exception as e:
        print(f"Error Ekstraksi: {e}")
        return None

def update_excel(data_baru):
    """Menambahkan satu baris data ke Excel tanpa menghapus data lama"""
    if not os.path.exists(FILE_EXCEL):
        # Buat baru jika belum ada
        df = pd.DataFrame([data_baru])
        df.to_excel(FILE_EXCEL, index=False)
    else:
        # Baca yang lama, gabung dengan yang baru
        df_lama = pd.read_excel(FILE_EXCEL)
        df_baru = pd.DataFrame([data_baru])
        df_final = pd.concat([df_lama, df_baru], ignore_index=True)
        df_final.to_excel(FILE_EXCEL, index=False)
    print(f"💾 Data tersimpan: {data_baru}")

# --- FUNGSI WATCHDOG (ROBOT PEMANTAU) ---
class PenjagaFolder(FileSystemEventHandler):
    def on_created(self, event):
        # Hanya proses file gambar, abaikan folder
        if not event.is_directory and event.src_path.endswith(('.jpg', '.jpeg', '.png')):
            print(f"\n👀 Mendeteksi file baru: {event.src_path}")
            
            # Tunggu 1 detik agar file selesai di-copy/download sepenuhnya
            time.sleep(1)
            
            # 1. Proses OCR
            hasil = ekstrak_data_struk(event.src_path)
            
            if hasil:
                # 2. Update Excel
                update_excel(hasil)
                
                # 3. Pindahkan File
                nama_file = os.path.basename(event.src_path)
                tujuan = os.path.join(FOLDER_OUTPUT, nama_file)
                
                # Cek jika file dengan nama sama sudah ada di folder tujuan
                if os.path.exists(tujuan):
                    base, ext = os.path.splitext(nama_file)
                    tujuan = os.path.join(FOLDER_OUTPUT, f"{base}_{int(time.time())}{ext}")

                shutil.move(event.src_path, tujuan)
                print(f"✅ File dipindahkan ke folder '{FOLDER_OUTPUT}'")
            else:
                print("❌ Gagal membaca struk.")

# --- MAIN LOOP ---
if __name__ == "__main__":
    # Pastikan folder tersedia
    if not os.path.exists(FOLDER_INPUT): os.makedirs(FOLDER_INPUT)
    if not os.path.exists(FOLDER_OUTPUT): os.makedirs(FOLDER_OUTPUT)

    event_handler = PenjagaFolder()
    observer = Observer()
    observer.schedule(event_handler, FOLDER_INPUT, recursive=False)
    observer.start()

    print(f"🤖 Robot Admin aktif...")
    print(f"📂 Memantau folder: '{FOLDER_INPUT}'")
    print("Tekan Ctrl+C untuk berhenti.\n")

    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        observer.stop()
    observer.join()
```

-----

### 🚀 Cara Menjalankan Automasi Ini

1.  **Buat Struktur Folder:**
    Pastikan di folder proyek Anda ada script Python ini, dan folder kosong bernama `Input_Struk`.
2.  **Jalankan Script:**
    Run scriptnya di terminal. Anda akan melihat pesan `🤖 Robot Admin aktif...`.
3.  **Lakukan Aksi:**
      * Biarkan terminal tetap terbuka.
      * Buka File Explorer.
      * *Copy-Paste* atau *Drag-Drop* gambar struk ke dalam folder `Input_Struk`.
4.  **Lihat Hasilnya:**
      * Dalam hitungan detik, gambar di `Input_Struk` akan hilang (dipindahkan ke `Sudah_Diproses`).
      * File `Laporan_Keuangan.xlsx` akan muncul/terupdate otomatis dengan data tanggal dan total harga.

### 💡 Tips Pengembangan Lanjut

Sekarang Anda memiliki sistem dasar yang solid. Jika ingin dikembangkan lebih jauh untuk **Skala Kantor**, Anda bisa menambahkan:

1.  **Validasi Mata Uang:** Menambahkan logika untuk membedakan IDR (Rp) atau USD ($).
2.  **Bot Notifikasi (Ide No. 8):** Tambahkan fungsi kirim pesan ke Telegram/WhatsApp: *"Bos, struk baru senilai Rp 500.000 baru saja direkap."*
3.  **Integrasi Google Sheets:** Gunakan `gspread` daripada Excel lokal, agar data bisa diakses tim secara online real-time.

Bagaimana? Apakah ini sudah sesuai dengan bayangan "automasi kerjaan" yang Anda inginkan?
