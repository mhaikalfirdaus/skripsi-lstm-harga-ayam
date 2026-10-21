# Prediksi Harga Ayam Ras Kota Bogor dengan LSTM

Sistem prediksi harga ayam ras harian di Kota Bogor menggunakan LSTM multivariat, lengkap dengan dashboard Streamlit. Proyek ini berasal dari skripsi S1 Ilmu Komputer, Universitas Pakuan Bogor.

## Masalah

Harga komoditas pangan naik turun, terutama menjelang hari raya, sehingga sulit diperkirakan oleh pedagang dan pengambil kebijakan. Proyek ini membangun model deret waktu untuk memprediksi harga ayam ras berikutnya dari riwayat harga, harga pakan jagung, dan kalender hari raya.

## Data

| Item | Keterangan |
|---|---|
| Periode | 1 Januari 2021 sampai 17 Juli 2026 (2.024 hari) |
| Harga ayam ras | Data harga harian Kota Bogor, file mentah di `datamentah/` |
| Harga pakan | Jagung pipilan per kg, `Dataset_Harga_Pakan_Jagung_2021_2026.xlsx` |
| Fitur kalender | `Is_Hari_Raya` dan `Efek_Lebaran`, dibuat dari `holiday_calendar.py` |
| Pembagian data | 70% latih, 15% validasi, 15% uji (urut waktu, tanpa diacak) |

Harga ayam pada sekitar 66% hari sama persis dengan hari sebelumnya, karena sumber data jarang memperbarui harga. Ini penting saat membaca hasil di bawah.

## Metode

1. **Pra-pemrosesan** (`preprocess.py`): menggabungkan data mentah, menyusun kalender harian, dan mengisi tanggal kosong dengan interpolasi linear.
2. **Pemeriksaan data macet** (`locf_reinterpolasi.py`): mendeteksi runtun nilai identik minimal 60 hari lalu menginterpolasi ulang. Pada berkas yang ada di repo ini, `dataset_bersih_lstm.csv` identik dengan `dataset_bersih_lstm_final.csv` dan `perbandingan_locf.csv` kosong, artinya tidak ada runtun yang diubah.
3. **Fitur** (`feature_engineering.py`): harga ayam, harga jagung, lag 1 hari, lag 7 hari, rata-rata bergerak 7 hari, dan dua fitur kalender.
4. **Model** (`train.py`): LSTM dengan jendela 14 hari. Targetnya adalah return harian, lalu harga direkonstruksi dengan `harga_t = harga_(t-1) x (1 + return_prediksi)`.
5. **Optimasi** (`optimize.py`, `tune.py`): TimeSeriesCV, grid search ukuran jendela (7, 14, 21, 30), dan grid search hyperparameter.

## Hasil

Metrik model akhir (`models/metrics.json`), dalam rupiah per kg, prediksi satu hari ke depan:

| Subset | MAE | RMSE | MAPE |
|---|---|---|---|
| Latih | Rp181 | Rp307 | 0,48% |
| Validasi | Rp118 | Rp227 | 0,32% |
| Uji | Rp180 | Rp441 | 0,45% |

**Pembanding sederhana.** Pada data uji yang sama, menebak harga hari ini sama dengan kemarin menghasilkan MAE Rp165, RMSE Rp454, dan MAPE 0,41%. Jadi LSTM hanya sedikit lebih baik pada RMSE (Rp441 vs Rp454) dan sedikit lebih buruk pada MAE dan MAPE. Artinya model belum terbukti mengungguli tebakan harga kemarin untuk prediksi satu hari. Angka pembanding ini dihitung ulang dari dataset dengan pembagian yang sama dengan model.

Hasil grid search ukuran jendela dan hyperparameter menunjukkan selisih RMSE yang sangat kecil (kisaran beberapa rupiah), sehingga pengaruhnya dianggap berada dalam batas noise.

## Cara menjalankan

```bash
pip install -r requirements.txt

# Opsional, hanya jika data mentah berubah
python preprocess.py
python locf_reinterpolasi.py

python train.py            # latih model, simpan ke models/
python predict.py          # forecast lewat terminal
streamlit run app.py       # dashboard web
```

## Struktur file

| File | Fungsi |
|---|---|
| `preprocess.py`, `locf_reinterpolasi.py` | Pembersihan data tahap 1 dan 2 |
| `feature_engineering.py`, `holiday_calendar.py` | Fitur turunan dan kalender hari libur |
| `train.py`, `predict.py` | Pelatihan dan forecast |
| `optimize.py`, `tune.py` | TimeSeriesCV dan tuning |
| `app.py` | Dashboard Streamlit |
| `verifikasi_prediksi.py` | Memeriksa bahwa prediksi uji berasal dari model |
| `models/` | Model terlatih, scaler, metrik, dan hasil tuning |

## Keterbatasan

- Forecast ke depan bersifat autoregresif, sehingga galat menumpuk pada horizon lebih dari 14 hari.
- Harga pakan untuk hari mendatang adalah asumsi, bukan data aktual.
- Sebelum melatih ulang dengan data setelah 2026, tambahkan tanggal hari libur tahun tersebut ke `holiday_calendar.py`.
- `optimize.py` dan `tune.py` memakai fitur turunan yang sedikit lebih banyak daripada `feature_engineering.py`, jadi hasil tuning tidak persis sebanding dengan model akhir.
- Hasil ini bukan satu-satunya dasar keputusan bisnis atau keuangan.
