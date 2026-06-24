# Resume Diskusi: Kotak Obat Cerdas untuk Monitoring Kepatuhan Minum OAT-KDT
**Tanggal Diskusi:** 24 Juni 2026  
**Status:** Dalam pengembangan — fase eksperimen hardware awal

---

## 1. Latar Belakang & Motivasi

Indonesia menempati posisi **ke-2 dunia** dalam jumlah kasus TBC. Salah satu masalah utama penanganan TB adalah **putus pengobatan (default)** — pasien berhenti minum obat di tengah siklus, yang mengakibatkan harus mengulang dari awal.

**Faktor penyebab putus obat:**
- Lupa minum obat karena kesibukan
- Tidak ada pengingat yang efektif
- Pengawas Minum Obat (PMO) konvensional terkendala jumlah tenaga, biaya operasional, dan aksesibilitas

**Solusi yang sudah ada:**
- PMO langsung → kendala SDM dan biaya
- Video call PMO → butuh literasi digital dan koneksi stabil

---

## 2. Konsep Solusi: Kotak Obat Cerdas

Sebuah sistem terintegrasi terdiri dari tiga komponen:

```
[Kotak Obat Cerdas] ←→ [Aplikasi Mobile] ←→ [Web Dashboard Nakes/PMO]
```

**Fokus pengembangan saat ini:** Kotak Obat Cerdas (hardware) sebagai komponen utama yang paling dekat dengan pasien.

### Kemampuan Target Kotak Obat:
1. Menghitung jumlah obat yang tersisa secara mandiri
2. Memberi notifikasi bunyi sesuai jadwal minum obat
3. Mendeteksi apakah pasien mengambil obat sesuai dosis
4. Memberi notifikasi saat obat hampir habis

---

## 3. Spesifikasi Obat (OAT-KDT)

Obat yang digunakan adalah **OAT-KDT (Obat Anti Tuberkulosis — Komposisi Dosis Tetap)** — 1 jenis obat dengan kandungan gabungan beberapa zat aktif.

| Parameter | Nilai |
|-----------|-------|
| Jenis obat | OAT-KDT (1 jenis, multi-kandungan) |
| Berat per tablet | **1.6g – 1.8g** (variatif antar tablet) |
| Dosis umum dewasa | **3 tablet per sekali minum** |
| Kapasitas target kotak | **42 tablet** (14 hari × 3 tablet/hari) |
| Frekuensi isi ulang | 2 minggu sekali |

**Catatan penting:** Variasi berat 1.6–1.8g per tablet bersifat inheren dari proses manufaktur obat, bukan cacat produksi.

---

## 4. Desain Hardware

### Filosofi Desain
- **Compact** — bisa digenggam dengan 1½ tangan manusia
- **Keyless** — tidak ada tombol fisik, tampilan bersih
- **Embedded-first** — tidak menggunakan SBC (Raspberry Pi), murni mikrokontroler
- **Mekanisme buka-tutup** — bukan dispenser, pasien ambil obat secara manual

### Komponen Hardware Terpilih

| Komponen | Spesifikasi | Fungsi |
|----------|-------------|--------|
| Mikrokontroler | ESP32 Wrover Module | Main controller, WiFi, PSRAM 4MB |
| Sensor berat | Load Cell + HX711 | Deteksi jumlah obat |
| RTC | DS3231 | Penjadwalan alarm minum obat |
| EEPROM | AT24C32 (onboard DS3231) | Penyimpanan data persisten |
| Display | TFT ILI9341 2.8" | Antarmuka visual |
| Buzzer | Aktif, GPIO 32 | Notifikasi suara |

### Pin Mapping

| Komponen | Pin |
|----------|-----|
| HX711 SCK | GPIO 14 |
| HX711 DT | GPIO 13 |
| DS3231 SCL | GPIO 21 |
| DS3231 SDA | GPIO 22 |
| Buzzer | GPIO 32 |
| TFT MOSI | GPIO 23 |
| TFT MISO | GPIO 19 |
| TFT SCK | GPIO 18 |
| TFT DC | GPIO 25 |
| TFT CS | GPIO 26 |
| TFT RESET | GPIO 27 |

### Development Environment
- **Editor:** VS Code + PlatformIO Extension
- **Platform:** pioarduino (fork komunitas untuk support ESP32 Arduino Core 3.x)
- **Platform string:** `https://github.com/pioarduino/platform-espressif32/releases/download/stable/platform-espressif32.zip`

---

## 5. Pendekatan Deteksi Obat

### 5.1 Mengapa Load Cell + Raw ADC

Pendekatan yang dipilih adalah membaca **nilai raw ADC langsung dari HX711** tanpa konversi ke satuan gram. Alasan:

- Konversi ke gram membutuhkan faktor kalibrasi yang **tidak konsisten antar booting**
- Raw ADC lebih jujur dan fleksibel untuk dimanipulasi secara algoritma
- Untuk use case ini, yang dibutuhkan adalah **perubahan relatif**, bukan nilai absolut

### 5.2 Arsitektur KNN Dua Lapis

Sistem deteksi menggunakan **dua KNN yang bekerja paralel:**

#### KNN Layer 1 — Event Detector (Delta ADC)
- **Input:** Delta ADC (selisih sebelum dan sesudah pengambilan obat)
- **Output:** Jumlah tablet yang diambil (kelas 1, 2, 3, 4, 5+)
- **Keunggulan:** Tidak terpengaruh drift booting karena berbasis perubahan relatif
- **Dataset:** Sampel delta ADC dari berbagai kondisi pengambilan

#### KNN Layer 2 — Stock Verifier (ADC Absolut + Re-anchor)
- **Input:** Nilai ADC absolut setelah event, dikoreksi dengan offset booting
- **Output:** Estimasi jumlah tablet yang tersisa di kotak
- **Mekanisme re-anchor:**
  ```
  OFFSET = ADC_BOOT - ADC_REFERENSI_TERSIMPAN
  Dataset Layer 2 digeser semua sebesar OFFSET
  ```
- **Keunggulan:** Bisa monitor stok secara pasif tanpa deteksi event

#### Cross-check antara Layer 1 dan Layer 2
```
Layer 1 hasil: "3 tablet diambil" → stok estimasi = stok_lama - 3
Layer 2 hasil: "tersisa 39 tablet"

|estimasi_layer1 - hasil_layer2| ≤ 1 → KONSISTEN ✓
|estimasi_layer1 - hasil_layer2| > 1 → ANOMALI ⚠️ → flag & notifikasi
```

### 5.3 Asumsi Pengembangan Awal (MVP)
- Setiap booting dianggap kotak **kosong** — user wajib isi ulang dan sistem mulai fresh
- Ini disederhanakan untuk prototipe awal, akan diupgrade di iterasi berikutnya
- Jumlah tablet saat pengisian dikonfirmasi via **aplikasi mobile** (bukan tombol fisik)

### 5.4 Mekanisme Deteksi Event (FSM)

```
IDLE
  │ delta ADC > INTERACT_THRESHOLD
  ▼
DETECTING — catat ADC_BEFORE, pantau fluktuasi (phantom weight diabaikan)
  │ ADC mulai stabil di nilai baru
  ▼
SETTLING — tunggu N sampel stabil berturut-turut
  │ stabil terkonfirmasi
  ▼
VALIDATING — hitung REAL_DELTA = ADC_BEFORE - ADC_AFTER
  │          feed ke KNN Layer 1
  ▼
LOGGING — update stok, cross-check Layer 2, kirim ke cloud
  │
  ▼
IDLE
```

**Event Window:** Semua perubahan ADC dalam window 5–10 detik dihitung sebagai 1 event pengambilan.

### 5.5 Mitigasi Phantom Weight & Hysteresis

Saat tangan masuk kotak untuk mengambil obat, terjadi dua fenomena:
- **Phantom Weight:** Tekanan jari ke dinding/dasar kotak menambah beban sementara
- **Hysteresis Mekanik:** Setelah tangan keluar, ADC tidak kembali persis ke nilai semula

**Solusi:** Sistem **mengabaikan semua nilai ADC selama fluktuasi** dan hanya mencatat:
- `ADC_BEFORE` = nilai stabil sebelum ada interaksi
- `ADC_AFTER` = nilai stabil setelah interaksi selesai (tangan sudah keluar)

`REAL_DELTA = ADC_BEFORE - ADC_AFTER`

### 5.6 Scaling Dataset Antar Device

Karena nilai ADC absolut berbeda tiap unit hardware, dataset KNN menggunakan pendekatan **normalisasi berbasis ADC_per_tablet:**

```
ADC_per_tablet_baru = karakterisasi 1x saat setup
Scale factor = ADC_per_tablet_baru ÷ ADC_per_tablet_referensi
Dataset = dataset_referensi × scale_factor
```

Asumsi yang perlu divalidasi: **linearitas load cell konsisten antar unit** — ini menjadi salah satu eksperimen yang harus dilakukan.

---

## 6. Mitigasi Drift Load Cell

| Jenis Drift | Penyebab | Mitigasi |
|-------------|----------|---------|
| Drift booting | Offset HX711 tidak konsisten | Re-anchor berbasis EEPROM |
| Thermal drift | Suhu mempengaruhi strain gauge | Periodic re-zero saat idle |
| Long-term creep | Fatigue mekanik | Re-anchor periodik |
| Noise elektronik | Interferensi, ripple supply | Moving average / filter |

**Periodic re-zero** dilakukan saat kondisi:
- Sudah >30 menit sejak jadwal minum obat terakhir
- Belum masuk window 15 menit sebelum jadwal berikutnya
- Tidak ada event terdeteksi dalam 30 menit terakhir

---

## 7. Hasil Eksperimen Hardware Awal

### Eksperimen A — Drift Booting

| Sesi | Baseline ADC |
|------|-------------|
| Sesi 1 | 182,822 |
| Sesi 2 | 187,882 |
| **Selisih drift** | **5,060 ADC units** |

> ⚠️ **Temuan kritis:** Drift antar booting sebesar 5,060 ADC units sangat signifikan. Nilai ini berpotensi lebih besar dari delta ADC 1 tablet — perlu dikonfirmasi dengan eksperimen D.

### Eksperimen B — Noise Idle (60 detik)

| Parameter | Sesi 1 | Sesi 2 |
|-----------|--------|--------|
| ADC min | 182,489 | 188,154 |
| ADC max | 185,363 | 189,974 |
| Range noise | 2,874 | 1,820 |
| Rata-rata | 184,202 | 188,967 |
| Drift dalam 60 detik | +1,665 (warming up) | -55 (stabil) |

> **Catatan:** Sesi 1 menunjukkan load cell masih dalam fase warming up — ADC naik 1,665 units dalam 60 detik. Sesi 2 sudah stabil (load cell sudah warm). Ini mengindikasikan adanya **minimum warming up time** yang perlu dikarakterisasi lebih lanjut.

### Eksperimen yang Belum Dilakukan

| Eksperimen | Tujuan | Status |
|------------|--------|--------|
| D — ADC per Tablet | Karakterisasi delta ADC untuk 1, 2, 3, 5, 10 tablet | **Belum** |
| C — Phantom Weight | Ukur efek phantom weight dan hysteresis secara kuantitatif | **Belum** |

---

## 8. Parameter yang Belum Ditentukan (Menunggu Data)

| Parameter | Keterangan | Cara Mendapat |
|-----------|------------|---------------|
| `INTERACT_THRESHOLD` | Minimum delta ADC dianggap sebagai event | Eksperimen D: harus > noise & < delta 1 tablet |
| `STABLE_THRESHOLD` | Toleransi fluktuasi untuk dianggap stabil | Dari noise floor eksperimen B |
| `STABLE_NEEDED` | Jumlah sampel stabil berturut-turut | Dari pola settling eksperimen B |
| `ADC_per_tablet` | Delta ADC untuk 1 tablet | Eksperimen D |
| Warming up time | Berapa lama sebelum load cell benar-benar stabil | Eksperimen B durasi panjang |
| Hysteresis magnitude | Seberapa besar dan konsisten error hysteresis | Eksperimen C |

---

## 9. Pertanyaan Terbuka untuk Didiskusikan

1. **Noise ±910 ADC units** saat idle — apakah ini wajar untuk load cell kelas ini, atau ada yang bisa dimitigasi dari sisi hardware (kapasitor decoupling, shielding kabel)?

2. **Drift booting 5,060 ADC units** — apakah mekanisme re-anchor berbasis EEPROM cukup robust untuk mengkompensasi ini, atau perlu pendekatan lain?

3. **Warming up time** — apakah perlu ditambahkan delay mandatory setelah booting sebelum sistem mulai beroperasi? Berapa lama yang acceptable dari sisi UX?

4. **Validasi asumsi linearitas** antar unit load cell — apakah perlu eksperimen multi-unit sebelum dataset KNN dianggap transferable?

5. **Filter yang tepat** — dari karakteristik noise yang ada, apakah kombinasi Median + EMA lebih baik dari Moving Average biasa? Perlu evaluasi setelah data eksperimen D tersedia.

6. **Threshold KNN** — dengan noise sebesar ini, apakah KNN 1 fitur (delta ADC) masih cukup, atau perlu multi-fitur (delta + durasi event + variance selama event)?

---

## 10. Rencana Eksperimen Berikutnya

### Prioritas Segera
1. **Eksperimen D** — Karakterisasi ADC per tablet
   - Letakkan 1, 2, 3, 5, 10 tablet secara bertahap
   - Catat delta ADC tiap penambahan
   - Evaluasi linearitas dan SNR terhadap noise

2. **Eksperimen C** — Phantom Weight & Hysteresis
   - Lakukan dengan ada obat di dalam (kondisi realistis)
   - Ulang minimal 10 kali untuk lihat konsistensi hysteresis

3. **Eksperimen Warming Up** — Durasi B diperpanjang
   - Log ADC selama 5–10 menit setelah booting
   - Tentukan kapan ADC benar-benar plateau

### Setelah Data Terkumpul
4. Tentukan semua parameter threshold berdasarkan data empiris
5. Implementasi filter yang sesuai
6. Implementasi FSM + KNN Layer 1 sederhana
7. Validasi end-to-end dengan skenario simulasi pengambilan obat

---

## 11. Kode yang Sudah Dibuat

### `main.cpp` — Firmware Eksperimen (PlatformIO)
Multi-eksperimen via Serial command:
- `a` → Eksperimen A (baseline & drift booting)
- `b` → Eksperimen B (noise idle 60 detik)
- `c` → Eksperimen C (phantom weight & hysteresis)
- `d` → Eksperimen D (karakterisasi ADC per tablet)
- `s` → Status sistem

### `logger.py` — Python Serial Logger
- Logging otomatis ke file `.txt` dengan timestamp
- Input keyboard untuk kirim perintah ke ESP32
- Output tersimpan di folder `data_eksperimen/`

### `calibration_knn.ino` — Asisten Kalibrasi KNN (Draft Awal)
- Pengumpulan dataset KNN dua arah (naik & turun)
- EMA filter (belum final — filter masih dalam evaluasi)
- Output: array `KNNDataPoint` siap di-hardcode ke firmware

---

*Resume ini dibuat sebagai bahan diskusi dengan pakar. Data eksperimen mentah tersimpan di file log terpisah.*
