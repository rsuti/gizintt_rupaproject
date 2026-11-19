# Dokumentasi Alur Pengolahan Data Dasbor Gizi NTT 2045

Riset PRIS RUPA Future Society 2026, FKM UI dan OR IPSH BRIN. Pengelola data: R. Sutiawan.
Versi 0.1, 19 November 2026. Dokumen kerja tim; diperbarui bersama kode.

Dokumen ini menjelaskan logika pengolahan data di balik setiap menu dasbor: dari mana data berasal, langkah olahnya, metode yang dipakai, dan hasil yang tampil. Mockup dasbor saat ini memakai angka ilustrasi; alur di bawah adalah rancangan untuk dasbor riset.

Tanda yang dipakai:

- **[KEPUTUSAN TIM]**: pilihan teknis yang belum diputuskan dan perlu disepakati tim.
- **[VERIFIKASI]**: sumber atau definisi perlu dicek ke penyedia data atau kamus variabel.

## Daftar isi

0. [Alur umum](#0-alur-umum)
1. [Menu 1 Ringkasan](#menu-1-ringkasan)
2. [Menu 2 Peta dan lintasan](#menu-2-peta-dan-lintasan)
3. [Menu 3 Kelompok kabupaten](#menu-3-kelompok-kabupaten)
4. [Menu 4 Skenario 2045](#menu-4-skenario-2045)
5. [Menu 5 Suara warga](#menu-5-suara-warga)
6. [Menu 6 Metadata](#menu-6-metadata)
7. [Pemutakhiran tahunan](#7-pemutakhiran-tahunan)
8. [Rujukan metode](#8-rujukan-metode)

---

## 0. Alur umum

Semua menu dibangun dari satu rangkaian skrip R yang dijalankan dengan paket `targets`. Satu perintah (`targets::tar_make()`) menjalankan ulang seluruh tahap, dan hanya tahap yang inputnya berubah yang dihitung ulang.

```mermaid
flowchart TD
    subgraph S["Sumber data"]
        A1["SSGI dan SKI<br/>stunting, wasting, overweight"]
        A2["Susenas Maret<br/>pangan olahan, protein"]
        A3["Podes<br/>minimarket, posyandu, jalan, sinyal"]
        A4["CHIRPS<br/>curah hujan bulanan"]
        A5["VIIRS dan WorldPop<br/>cahaya malam, sebaran penduduk"]
        A6["Data rutin dan harga<br/>TTD, harga pangan"]
        A7["Data warga PK3<br/>narasi, peta, indikator kader"]
    end
    S --> B["data/raw<br/>file asli, tidak diubah"]
    B --> C["Harmonisasi<br/>kode wilayah 22 kab/kota, tahun, satuan"]
    C --> D["Pemeriksaan mutu<br/>nilai kosong, nilai ekstrem, RSE"]
    D --> E["Panel kab/kota 2013-2024<br/>22 kovariat, 7 kelompok"]
    E --> F["Model dan analisis<br/>BYM2+RW2, kml, panel FE, skenario"]
    A7 --> G["Analisis kualitatif<br/>Framework Method"]
    F --> H["data/output<br/>tabel agregat CSV/JSON"]
    G --> H
    H --> I["Cek anonimisasi"]
    I --> J["Quarto dashboard<br/>6 menu"]
    J --> K["GitHub Pages<br/>rsuti.github.io/gizintt_rupaproject"]
```

| Tahap | Isi | Folder atau berkas |
|---|---|---|
| Data mentah | File asli dari penyedia, disimpan apa adanya | `data/raw/` (tidak diunggah ke repositori publik) |
| Harmonisasi | Kode wilayah BPS, penyesuaian pemekaran, satuan | `R/01_harmonisasi.R` |
| Panel | Satu baris per kab/kota per tahun | `data/panel/panel_ntt_2013_2024.csv` |
| Model | Skrip per metode | `R/02_bym2.R`, `R/03_kml.R`, `R/04_panel_fe.R`, `R/05_skenario.R` |
| Keluaran | Hanya angka agregat | `data/output/` |
| Dasbor | Halaman Quarto | `dasbor/index.qmd` |

Nama berkas dan folder di atas adalah usulan struktur [KEPUTUSAN TIM].

---

## Menu 1 Ringkasan

Menampilkan nilai provinsi tahun terpilih, perubahan dari tahun sebelumnya, tren singkat, status warna, dan kotak skenario tempat NTT berada.

```mermaid
flowchart TD
    A["Panel kab/kota<br/>nilai per indikator per tahun"] --> B["Agregasi ke provinsi<br/>rata-rata tertimbang penduduk sasaran"]
    P["Proyeksi penduduk BPS<br/>balita, remaja putri"] --> B
    B --> C["Hitung perubahan<br/>nilai tahun t dikurangi tahun t-1"]
    B --> D["Tren singkat<br/>deret 2013 sampai tahun t"]
    B --> E{"Batas peringatan<br/>sudah disepakati FGD II?"}
    E -- "ya" --> F["Bandingkan nilai dengan batas<br/>aman, waspada, perlu tindakan"]
    E -- "belum" --> G["Status abu-abu<br/>batas belum disepakati"]
    C --> H["Kartu indikator"]
    D --> H
    F --> H
    G --> H
    I["Indeks penanda arah<br/>dari Menu 4"] --> J["Kotak skenario A-D<br/>dan arah perubahan"]
```

| Unsur | Logika |
|---|---|
| Nilai provinsi | Rata-rata kab/kota ditimbang jumlah penduduk sasaran (balita untuk stunting dan wasting; remaja putri untuk TTD; rumah tangga untuk indikator Susenas). Untuk indikator survei gizi, dipakai angka provinsi resmi bila tersedia. |
| Arah baik/buruk | Tiap indikator punya arah "buruk bila naik" (misalnya stunting) atau "buruk bila turun" (misalnya protein, posyandu, TTD). Warna teks perubahan mengikuti arah ini. |
| Status warna | Hanya aktif bila batas peringatan sudah ditetapkan di FGD II (Mei 2027). Sebelum itu tampil "batas belum disepakati". |
| Anemia | Hanya data konteks (SKI, sekitar 5 tahunan, tingkat provinsi); tidak dipakai untuk pemantauan tahunan. |

---

## Menu 2 Peta dan lintasan

Menampilkan peta 22 kab/kota untuk indikator terpilih dan lintasan satu kabupaten dari 2013 beserta rentang ketidakpastian. Cara olah berbeda menurut jenis sumber data.

### 2a. Indikator survei gizi (stunting, wasting)

```mermaid
flowchart TD
    A["Prevalensi kab/kota per survei<br/>2013, 2018, 2019, 2021, 2022, 2023, 2024"] --> B["Ambil galat baku<br/>dari publikasi atau mikrodata"]
    B --> C["Model BYM2 + RW2 di R-INLA<br/>efek wilayah bertetangga + efek waktu"]
    W["Matriks ketetanggaan<br/>22 kab/kota, pulau dihubungkan manual"] --> C
    X["Kovariat panel"] --> C
    C --> D["Posterior per kab/kota per tahun<br/>median dan interval kredibel 95%"]
    D --> E["Validasi LOPO<br/>sisihkan satu periode, misalnya 2024,<br/>bandingkan prediksi dengan angka survei"]
    E --> F{"Galat prediksi<br/>dapat diterima?"}
    F -- "ya" --> G["Simpan estimasi 2013-2024"]
    F -- "tidak" --> H["Tinjau spesifikasi model"]
    H --> C
    G --> I["Peta: kelas ambang WHO-UNICEF"]
    G --> J["Grafik lintasan: garis median,<br/>pita interval 95%, titik survei"]
```

### 2b. Indikator Susenas (pangan olahan, protein)

```mermaid
flowchart TD
    A["Mikrodata Susenas Maret<br/>modul konsumsi"] --> B["Estimasi langsung per kab/kota<br/>dengan bobot survei"]
    B --> C{"RSE lebih dari 25%?"}
    C -- "tidak" --> D["Pakai estimasi langsung"]
    C -- "ya" --> E["Estimasi area kecil Fay-Herriot<br/>dengan kovariat Podes dan VIIRS"]
    D --> F["Panel indikator"]
    E --> F
```

### 2c. Indikator big data dan Podes

```mermaid
flowchart TD
    A["CHIRPS bulanan"] --> B["Hitung SPI per kab/kota"]
    B --> C["Jumlah bulan kering<br/>dalam 24 bulan terakhir"]
    D["VIIRS bulanan"] --> E["Rata-rata tertimbang penduduk<br/>bobot sel WorldPop"]
    F["Podes 2018, 2021, 2024, 2025"] --> G["Persentase desa per kab/kota"]
    G --> H{"Tahun tanpa Podes?"}
    H -- "ya" --> I["Isi antartahun<br/>KEPUTUSAN TIM: interpolasi linier<br/>atau nilai Podes terakhir"]
    H -- "tidak" --> J["Panel indikator"]
    I --> J
    C --> J
    E --> J
```

| Unsur | Logika |
|---|---|
| Kelas warna peta, stunting dan wasting | Lima kelas ambang prevalensi WHO-UNICEF (de Onis dkk., 2019). Stunting: kurang dari 2,5; 2,5 sampai kurang dari 10; 10 sampai kurang dari 20; 20 sampai kurang dari 30; 30% atau lebih. Wasting: kurang dari 2,5; 2,5 sampai kurang dari 5; 5 sampai kurang dari 10; 10 sampai kurang dari 15; 15% atau lebih. |
| Kelas warna peta, indikator lain | Kuintil antarkabupaten pada tahun terpilih. Hijau selalu kondisi terbaik; untuk indikator "buruk bila turun", urutan warna dibalik. |
| Penimbang VIIRS | Cahaya malam di sel tanpa penduduk tidak ikut dihitung; tiap sel diberi bobot jumlah penduduk WorldPop agar nilai kab/kota mewakili tempat orang tinggal. |
| Overweight balita | Angka kab/kota kurang teliti (SKI 2023: RSE di atas 25% di sebagian besar kab/kota), sehingga hanya ditampilkan di tingkat provinsi. |
| Harga pangan | Komoditas dan ketersediaan per kab/kota NTT [VERIFIKASI]. |
| Cakupan TTD | Sumber dan definisi laporan rutin Dinkes [VERIFIKASI]. |

---

## Menu 3 Kelompok kabupaten

Mengelompokkan kabupaten yang lintasan stuntingnya mirip, lalu menjelaskan faktor yang membedakan kelompok dan kaitannya dengan skenario.

```mermaid
flowchart TD
    A["Lintasan stunting 2013-2024<br/>hasil BYM2 Menu 2"] --> B["Klasterisasi kml<br/>k-means untuk data longitudinal"]
    B --> C["Coba k = 2 sampai 6<br/>ulang 20 kali per k"]
    C --> D["Pilih k<br/>kriteria Calinski-Harabasz<br/>dan keterbacaan pola"]
    D --> E["Uji stabilitas<br/>anggota kelompok tetap saat diulang?"]
    E --> F["Beri label kelompok<br/>misalnya tinggi menetap, turun lambat, turun cepat"]
    F --> G["Profil faktor per kelompok<br/>rata-rata 22 kovariat"]
    H["Panel kovariat"] --> I["Regresi panel efek tetap<br/>kab/kota dan tahun"]
    I --> J["Wild cluster bootstrap<br/>karena hanya 22 klaster"]
    J --> K["Faktor yang berkaitan<br/>dengan perubahan stunting"]
    G --> L["Kartu kelompok di dasbor"]
    K --> L
    L --> M["Kaitan kelompok ke skenario A-D<br/>dibahas bersama Menu 4"]
```

| Unsur | Logika |
|---|---|
| Input klasterisasi | Lintasan hasil model (bukan angka survei mentah), agar tahun tanpa survei sudah terisi dan galat sampling diredam. |
| Jumlah kelompok | Dipilih dari kriteria statistik dan kemudahan dibaca pengguna; jumlah final [KEPUTUSAN TIM]. |
| Wild cluster bootstrap | Galat baku biasa terlalu optimistis bila klaster hanya 22; bootstrap memberi interval yang lebih jujur. |
| Batas tafsir | Hubungan faktor dengan stunting bersifat asosiasi, bukan sebab-akibat. |

---

## Menu 4 Skenario 2045

Menampilkan matriks empat skenario, posisi NTT (atau satu kabupaten) dari tahun ke tahun, penanda arah, dan rentang proyeksi stunting per skenario.

### 4a. Penyusunan skenario (sekali, awal 2027)

```mermaid
flowchart TD
    A["Daftar pendorong perubahan<br/>kerangka STEEP"] --> B["Sumber: kajian pustaka,<br/>hasil Menu 2-3, narasi warga PK3"]
    B --> C["Penilaian dampak dan ketidakpastian<br/>oleh pakar dan warga"]
    C --> D["Pilih dua pendorong<br/>dampak tinggi, ketidakpastian tinggi"]
    D --> E["Dua dimensi skenario<br/>usulan: pola pangan keluarga,<br/>kekuatan layanan gizi"]
    E --> F["Matriks 2x2: skenario A, B, C, D"]
    F --> G["Narasi tiap skenario<br/>FGD bersama warga dan pemangku kepentingan"]
    G --> H["Pemetaan ke SSP<br/>usulan: A=SSP1, B=SSP5, C=SSP3, D=SSP4"]
    G --> I["Backcasting<br/>langkah kebijakan menuju skenario A"]
```

### 4b. Posisi tahunan dan proyeksi

```mermaid
flowchart TD
    A["Indikator dimensi pangan<br/>pangan olahan, minimarket, protein"] --> C["Standardisasi<br/>KEPUTUSAN TIM: skor z atau min-maks<br/>terhadap tahun dasar"]
    B["Indikator dimensi layanan<br/>posyandu, cakupan TTD"] --> C
    C --> D["Indeks pangan dan indeks layanan<br/>rata-rata berbobot, KEPUTUSAN TIM"]
    D --> E["Titik posisi per tahun<br/>sumbu X pangan, sumbu Y layanan"]
    E --> F["Kotak skenario saat ini<br/>dan jejak 2019 sampai tahun t"]
    D --> G["Penanda arah<br/>perubahan tiap indikator dari t-1"]
    H["Model BYM2 Menu 2"] --> I["Jalur kovariat per skenario<br/>sampai 2030 dan 2045"]
    I --> J["Proyeksi stunting per skenario<br/>rentang interval 95%"]
```

| Unsur | Logika |
|---|---|
| Arah penanda | Pangan olahan dan minimarket naik berarti bergeser ke pangan olahan; protein turun juga dibaca bergeser ke pangan olahan. Posyandu dan TTD naik berarti layanan menguat. Bulan kering dibaca sebagai tekanan iklim, tidak masuk dua dimensi. |
| Rumus indeks | Rumus di mockup hanya ilustrasi. Indikator, bobot, dan cara standardisasi ditetapkan tim [KEPUTUSAN TIM]. |
| Proyeksi | Ditampilkan sebagai rentang, bukan satu angka. Skenario bukan ramalan; gunanya menguji kebijakan mana yang tetap berhasil di keempat kemungkinan. |

---

## Menu 5 Suara warga

Menampilkan hasil komponen warga (PK3) dari 6 desa di 3 lokasi gradien urbanisasi: TTS, Kabupaten Kupang, Kota Kupang.

```mermaid
flowchart TD
    A["Pemilihan desa<br/>Podes 2025 tingkat desa:<br/>1 terpapar kota + 1 agraris per lokasi"] --> B["Pra-rekrutmen oleh supervisor lokal<br/>±30 informan per lokasi"]
    B --> C["Persetujuan<br/>e-consent atau lisan terekam"]
    C --> D["Kunjungan I<br/>narasi teks, foto, audio<br/>aplikasi ponsel luring atau kertas"]
    C --> E["Pemetaan partisipatif<br/>sumber pangan, layanan, sinyal"]
    D --> F["Transkripsi"]
    F --> G["Anonimisasi<br/>hapus nama, alamat, ciri pengenal"]
    G --> H["Pengodean di ATLAS.ti<br/>Framework Method, dua pengode"]
    H --> I{"Alfa Krippendorff<br/>memadai?"}
    I -- "belum" --> J["Diskusi pengode,<br/>perbaiki buku kode"]
    J --> H
    I -- "ya" --> K["Matriks tema per desa"]
    E --> L["Digitasi peta<br/>tingkat desa, tanpa titik rumah"]
    K --> M["Kunjungan II: tafsir warga<br/>pola data dikembalikan ke warga"]
    L --> M
    M --> N["Posisi desa di matriks skenario<br/>kini dan 5 tahun lalu"]
    M --> O["Indikator dari warga<br/>menu anak, kios kemasan,<br/>kecukupan pangan musim kering"]
    K --> P["Joint display<br/>temuan kuantitatif dan kualitatif berdampingan"]
    N --> Q["Menu 5 dasbor"]
    O --> Q
    P --> Q
```

| Unsur | Logika |
|---|---|
| Kesepakatan pengode | Target alfa Krippendorff sekurang-kurangnya 0,80; nilai 0,667 sampai 0,80 hanya untuk kesimpulan sementara (Krippendorff, 2004). |
| Kutipan | Hanya kutipan yang sudah dianonimkan dan disetujui tampil; label informan hanya peran (misalnya ibu balita, kader). |
| Tingkat lokasi | Paling rinci desa. Tidak ada koordinat rumah atau nama informan (UU 27/2022 dan protokol etik). |
| Indikator warga | Dikumpulkan kader per semester atau tahunan; instrumen disusun dari hasil PK3 dan diuji pada Kunjungan II. |

---

## Menu 6 Metadata

Menampilkan definisi, sumber, tingkat, frekuensi, dimensi skenario, dan batas peringatan tiap indikator, serta menyediakan data agregat.

```mermaid
flowchart TD
    A["Kamus variabel<br/>definisi, satuan, sumber, frekuensi"] --> B["Tabel metadata indikator"]
    C["Hasil FGD II<br/>daftar final, batas peringatan"] --> B
    D["data/output agregat"] --> E{"Ada sel dengan<br/>jumlah responden kecil?"}
    E -- "ya" --> F["Sembunyikan sel<br/>batas jumlah: KEPUTUSAN TIM"]
    E -- "tidak" --> G["Berkas CSV agregat"]
    F --> G
    B --> H["Menu 6 dasbor"]
    G --> H
```

---

## 7. Pemutakhiran tahunan

Alur yang dijalankan Bapperida atau tim setiap kali data baru terbit.

```mermaid
flowchart TD
    A["Data baru terbit<br/>SSGI/SKI, Susenas, Podes, CHIRPS"] --> B["Simpan di data/raw<br/>dengan nama sesuai kamus"]
    B --> C["Jalankan targets::tar_make()"]
    C --> D["Laporan pemeriksaan mutu otomatis"]
    D --> E{"Ada peringatan mutu?"}
    E -- "ya" --> F["Periksa dan perbaiki data,<br/>catat di log perubahan"]
    F --> C
    E -- "tidak" --> G["quarto render dasbor"]
    G --> H["git commit dan push ke GitHub"]
    H --> I["GitHub Pages memperbarui situs"]
```

| Frekuensi | Sumber |
|---|---|
| Bulanan | CHIRPS, VIIRS |
| Tahunan | SSGI/SKI, Susenas, data rutin TTD, indikator warga tahunan |
| Sekitar 3 tahunan | Podes |
| Sekitar 5 tahunan | Anemia (SKI), hanya konteks |

---

## 8. Rujukan metode

- Cameron, A. C., Gelbach, J. B., & Miller, D. L. (2008). Bootstrap-based improvements for inference with clustered errors. *The Review of Economics and Statistics, 90*(3), 414–427.
- de Onis, M., dkk. (2019). Prevalence thresholds for wasting, overweight and stunting in children under 5 years. *Public Health Nutrition, 22*(1), 175–179.
- Fay, R. E., & Herriot, R. A. (1979). Estimates of income for small places: An application of James-Stein procedures to census data. *Journal of the American Statistical Association, 74*(366), 269–277.
- Gale, N. K., Heath, G., Cameron, E., Rashid, S., & Redwood, S. (2013). Using the framework method for the analysis of qualitative data in multi-disciplinary health research. *BMC Medical Research Methodology, 13*, 117.
- Genolini, C., & Falissard, B. (2010). KmL: k-means for longitudinal data. *Computational Statistics, 25*(2), 317–328.
- Krippendorff, K. (2004). *Content analysis: An introduction to its methodology* (2nd ed.). Sage.
- Riebler, A., Sørbye, S. H., Simpson, D., & Rue, H. (2016). An intuitive Bayesian spatial model for disease mapping that accounts for scaling. *Statistical Methods in Medical Research, 25*(4), 1145–1165.

Rujukan lain (SSP, CIOMS, BPS, WorldPop, CHIRPS, VIIRS) ada di daftar pustaka kerja proyek.
