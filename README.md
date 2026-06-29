# 🌡️ Analisis & Prediksi Cuaca IoT — Random Forest + Explainable AI (SHAP)

> Proyek Mata Kuliah **Kecerdasan Buatan** — Kelompok AtmoMind, Kelas TI-1C

Analisis data time series dari sensor IoT yang dipasang di **teras outdoor** menggunakan Python. Mencakup eksplorasi data (EDA), penanganan missing values akibat gangguan jaringan dengan **imputasi data eksternal**, hingga prediksi suhu, kelembapan, dan cahaya menggunakan **Random Forest**, diperkuat dengan **Explainable AI (XAI) berbasis SHAP** untuk menjelaskan alasan di balik setiap prediksi.

---

## 👥 Anggota Kelompok

| No | Nama | NIM |
|----|------|-----|
| 1  | *Azka Galuh Basuki* | *4.33.25.2.04* |
| 2  | *Dela Fajar Mulia* | *4.33.25.2.07* |
| 3  | *Jatmiko Satrio Wibowo* | *4.33.25.2.11* |
| 4  | *Muhammad Fadhil* | *4.33.25.2.15* |
| 5  | *Ulfan Nayaka Dipta* | *4.33.25.2.23* |

> **Kelas:** TI-1C &nbsp;|&nbsp; **Kelompok:** AtmoMind &nbsp;|&nbsp; **Mata Kuliah:** Kecerdasan Buatan

---

## 📁 Struktur Repository

```
├── 📁 dataset
│   ├── 📄 open_meteo_tubes.csv
│   └── 📄 sensor_data_tubes.csv
├── 📝 README.md
└── 📄 weather_analysis_rf_shap.ipynb
```

---

## 🔧 Spesifikasi Sensor

| Komponen | Keterangan |
|---|---|
| Sensor suhu & kelembapan | DHT11 |
| Sensor cahaya | LDR Module |
| Interval pengambilan data | 5 menit |
| Penyimpanan | PostgreSQL (Supabase) |
| Lokasi sensor | Teras |

---

## 📊 Dataset

### Dataset Primer — `sensor_data_tubes.csv`

| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| `id` | int | ID unik record |
| `created_at` | datetime | Timestamp UTC (GMT+0) dari backend Supabase |
| `suhu` | float | Suhu udara (°C) |
| `kelembapan` | float | Kelembapan relatif (%) |
| `cahaya` | int | Nilai ADC sensor LDR (0–4095) |
| `kondisi` | string | `TERANG` / `GELAP` — ditentukan firmware |

### Dataset Eksternal — `open_meteo_tubes.csv`

| Sumber | Isi | Resolusi |
|--------|-----|----------|
| Open-Meteo API | Suhu udara (`temperature_2m`) | Hourly, GMT+7 |

> File ini berisi 3 blok di dalam satu CSV: metadata lokasi, snapshot kelembapan/tekanan "current" (1 baris saja, bukan time-series), dan time-series suhu hourly. **Hanya blok hourly suhu yang dipakai** untuk imputasi — kelembapan dan cahaya tidak punya sumber eksternal historis yang sesuai pada data ini.

### ⚠️ Catatan Penting Sensor

- **Lokasi:** Teras outdoor — terpapar sinar matahari langsung
- **Periode:** 10–29 Juni 2026 · 4.807 record asli · interval target ~5 menit
- **Timezone:** Kolom `created_at` dalam **UTC (GMT+0)**, dikonversi ke **WIB (GMT+7)** saat preprocessing. Open-Meteo hourly sudah dalam local time GMT+7 sejak sumbernya, jadi cukup di-*localize*, bukan dikonversi
- **Sensor LDR terbalik:** Nilai `cahaya` raw **tinggi = gelap**, **rendah = terang** (karakteristik fisika LDR). Koreksi: `cahaya_terkoreksi = 4095 - cahaya_raw`
- **Audit gap:** dari total ±5.388 slot grid 5-menit, 503 slot (9,3%) kosong akibat gangguan pengiriman — gap terbesar 169 menit (2,8 jam), 4 gap di atas 60 menit
- **Strategi imputasi:** seluruh gap suhu diisi dari Open-Meteo hourly (di-upsample ke 5 menit, dikalibrasi dengan offset median +1,94 °C terhadap data sensor asli yang overlap). Kelembapan & cahaya diisi interpolasi linear karena tidak ada sumber eksternal historis untuk kedua variabel ini

---

## 📓 Notebook

### `weather_analysis_rf_shap.ipynb` — Analisis Utama

| # | Bagian | Deskripsi |
|---|--------|-----------|
| 1 | Import Library | pandas, numpy, matplotlib, seaborn, scikit-learn, **SHAP** |
| 2 | Load & Inspect | Muat sensor IoT + Open-Meteo (parsing CSV multi-section) |
| 3 | Preprocessing | Konversi UTC→WIB, koreksi nilai LDR |
| 4 | Audit Kualitas Data | Deteksi gap pengiriman, distribusi interval |
| 5 | **Imputasi** | Suhu → Open-Meteo terkalibrasi; Kelembapan & Cahaya → interpolasi linear |
| 6 | Feature Engineering | Fitur temporal, siklus sin/cos, flag siang/malam, lag & rolling (dihitung di grid 5 menit kontinu agar tidak melompat gap) |
| 7 | EDA | Univariat · Bivariat · Time-Series Story · Pola Diurnal |
| 8 | Deteksi Outlier | IQR + Z-Score, visualisasi pada timeline |
| 9 | **Random Forest** | 3 model terpisah: Suhu, Kelembapan, Cahaya |
| 10 | **Explainable AI (SHAP)** | Summary plot, dependence plot, waterfall & force plot per-prediksi |
| 11 | Kesimpulan | Ringkasan temuan, insight SHAP, keterbatasan, rekomendasi |

---

## 🔍 Temuan EDA

| Variabel | Temuan |
|----------|--------|
| **Suhu** | Rentang 23,2–39,3°C · Puncak rata-rata ~14:00 WIB (±36°C) · Terendah ~05:00–06:00 WIB (±25,5°C) |
| **Kelembapan** | Korelasi negatif kuat dengan suhu (r ≈ −0,92) · Malam ~87% · Siang terendah ~43% (pukul 14:00) |
| **Cahaya** | Setelah koreksi LDR: korelasi positif dengan suhu (r ≈ +0,70), negatif dengan kelembapan (r ≈ −0,73) — masuk akal secara fisik |
| **Transisi** | Sensor GELAP→TERANG ~06:00 WIB, TERANG→GELAP ~18:00 WIB |
| **Kalibrasi Eksternal** | Sensor vs Open-Meteo: offset sistematis +1,94°C — wajar karena sensor teras terpapar langsung matahari, sedangkan Open-Meteo bersifat reanalysis area luas |

---

## 🤖 Hasil Prediksi — Random Forest

| Target | MAE | RMSE | R² |
|--------|-----|------|-----|
| Suhu (°C) | 0,168 | 0,255 | 0,994 |
| Kelembapan (%) | 0,700 | 0,984 | 0,997 |
| Intensitas Cahaya | 51,08 | 94,67 | 0,996 |

Ketiga model memperoleh **R² > 0,99**, didorong utamanya oleh autokorelasi alami sensor IoT pada skala 5 menit — nilai 5 menit lalu adalah prediktor yang sangat kuat untuk nilai sekarang. Evaluasi dilakukan terhadap data sensor **asli** (bukan hasil imputasi), agar metrik mencerminkan kemampuan model yang sesungguhnya.

---

## 🧠 Explainable AI (XAI) dengan SHAP

Random Forest akurat, tapi secara default ia adalah *black box*. **SHAP (SHapley Additive exPlanations)** dipakai untuk menjelaskan *mengapa* model memprediksi nilai tertentu — bukan cuma *seberapa penting* suatu fitur secara umum (seperti `feature_importances_` bawaan), tapi kontribusi pasti tiap fitur (dalam satuan asli: °C, %, dst.) pada **setiap prediksi individual**.

### Insight Global (Mean |SHAP value|)

| Target | Fitur Paling Dominan | Kontribusi | Fitur Berikutnya |
|--------|----------------------|------------|-------------------|
| Suhu | `suhu_lag1` (suhu 5 menit lalu) | ~45% | `suhu_roll6_mean`, `suhu_lag2` |
| Kelembapan | `lembap_lag1` | ~96% | `cahaya_lag1`, `hari_ke` |
| Cahaya | `cahaya_lag1` | ~75% | `is_malam`, `hari_ke` |

- **Suhu**: prediksi didominasi momentum jangka pendek — kombinasi nilai 5 menit lalu dan rata-rata 30 menit terakhir
- **Kelembapan**: berubah sangat halus antar 5 menit, jadi nilai sebelumnya hampir cukup sendirian untuk memprediksi nilai berikutnya
- **Cahaya**: lag tetap dominan, tapi `is_malam` ikut berperan besar — masuk akal karena cahaya berubah drastis & cepat saat transisi siang-malam, sehingga model butuh sinyal konteks waktu, bukan cuma lag

### Studi Kasus: Menjelaskan Prediksi Individual

| Kasus | Waktu | Aktual | Prediksi | Error | Pendorong Utama |
|-------|-------|--------|----------|-------|-------------------|
| Paling akurat | 27/06 05:50 | 25,80°C | 25,80°C | 0,00 | `suhu_lag1`, `suhu_roll6_mean`, `suhu_lag2` — semua menurunkan prediksi searah (konsisten) |
| Paling meleset | 26/06 14:10 | 36,90°C | 35,66°C | 1,24 | `suhu_roll6_mean` & `suhu_lag1` menaikkan, tapi rolling mean masih "tertinggal" dari kenaikan suhu yang tajam |

Waterfall & force plot pada notebook memvisualisasikan kedua kasus ini secara rinci — termasuk nilai SHAP per fitur dan arah dorongannya terhadap *base value* (rata-rata prediksi model).

---

## 🚀 Cara Menjalankan

### 1. Clone repository

```bash
git clone https://github.com/delafajarmulia/AtmoMind-AI.git
cd AtmoMind-AI
```

### 2. Install dependencies

```bash
pip install pandas numpy matplotlib seaborn scikit-learn shap jupyter
```

### 3. Jalankan notebook

```bash
jupyter notebook weather_analysis_rf_shap.ipynb
```

> Pastikan kedua file dataset (`sensor_data_tubes.csv` dan `open_meteo_tubes.csv`) berada di direktori yang sama dengan notebook.

---

## 🛠️ Tech Stack

![Python](https://img.shields.io/badge/Python-3.9+-3776AB?logo=python&logoColor=white)
![Jupyter](https://img.shields.io/badge/Jupyter-Notebook-F37626?logo=jupyter&logoColor=white)
![pandas](https://img.shields.io/badge/pandas-150458?logo=pandas&logoColor=white)
![scikit--learn](https://img.shields.io/badge/scikit--learn-F7931E?logo=scikit-learn&logoColor=white)
![SHAP](https://img.shields.io/badge/SHAP-Explainable%20AI-FF0D57?logo=python&logoColor=white)

| Library | Kegunaan |
|---------|---------|
| `pandas` | Manipulasi data, reindex, resample |
| `numpy` | Komputasi numerik, fitur siklus |
| `matplotlib` & `seaborn` | Visualisasi statis |
| `scikit-learn` | Random Forest, evaluasi model |
| `shap` | Explainable AI — TreeExplainer, summary/dependence/waterfall/force plot |

---

## ⚠️ Keterbatasan

- Gap suhu diisi estimasi data eksternal — bukan data pengukuran sensor nyata
- Sensor teras terpapar langsung sinar matahari (offset suhu sistematis ~1,9°C di atas Open-Meteo)
- Open-Meteo hanya menyediakan suhu historis pada dataset ini; kelembapan & cahaya saat gap sepenuhnya bergantung pada interpolasi linear
- 19 hari pengamatan belum cukup untuk menangkap pola mingguan/musiman
- Data eksternal hourly di-upsample ke 5 menit → ada efek smoothing pada periode gap
- SHAP dihitung pada sampel test set (bukan seluruh data) untuk efisiensi komputasi
- Evaluasi memakai split waktu 80/20 sederhana (bukan walk-forward), cukup untuk skala data 19 hari namun kurang merepresentasikan stabilitas performa pada periode lebih panjang

---

## 📌 Lisensi

Proyek ini dibuat untuk keperluan akademik. Bebas digunakan sebagai referensi pembelajaran.

---

<p align="center">
  Dibuat dengan ❤️ oleh <strong>Kelompok AtmoMind · Kelas TI-1C</strong> · 2026
</p>
