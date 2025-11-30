# 📘 Penerapan Genetic Algorithm (GA) untuk Optimasi Pemilihan Induk Tanaman Padi

Proyek ini berisi implementasi **Genetic Algorithm (GA)** untuk melakukan *optimasi pemilihan induk tanaman padi* berbasis data genotipe. GA digunakan untuk mencari kombinasi pasangan induk yang secara genetik **paling berjauhan** sehingga diharapkan dapat meminimalkan inbreeding dan meningkatkan peluang munculnya kombinasi sifat unggul.

Seluruh eksperimen diimplementasikan dalam bentuk Jupyter/Colab Notebook, mulai dari pemrosesan data genomik hingga evaluasi kualitas solusi GA.

---

## 🎯 Tujuan Proyek

1. Menerapkan PCA dan Genomic Relationship Matrix (GRM) untuk merepresentasikan kedekatan genetik antar individu.
2. Menggunakan Genetic Algorithm (GA) untuk mencari pasangan induk dengan **jarak genetik maksimum**.
3. Mengevaluasi kinerja GA dibandingkan:
   - pemilihan pasangan secara acak, dan
   - solusi optimal global (brute force) berdasarkan matriks jarak.

---

## 🧩 Alur Utama Notebook

### 1. Import Package

Notebook memuat berbagai pustaka Python yang dibutuhkan, di antaranya:

- `numpy`, `pandas`
- `dask`, `dask.array`
- `pandas_plink` (membaca file PLINK: `.bed/.bim/.fam`)
- `sklearn.metrics` (khususnya `pairwise_distances`)
- `matplotlib` (visualisasi)
- modul standar seperti `random` dan `time`

### 2. Pemrosesan Data Genotipe & PCA

1. Membaca data PLINK:

   - `bim` → informasi SNP (kromosom, posisi, alel)
   - `fam` → informasi individu (ID sampel)
   - `bed` → matriks genotipe (variants × samples)

2. Mengganti nilai **missing** dengan rata-rata per SNP dan melakukan standardisasi.

3. Menghitung **Genomic Relationship Matrix (GRM)** sebagai ukuran kedekatan genetik antar individu.

4. Melakukan **eigendecomposition** pada GRM untuk mendapatkan:

   - eigenvalues → total variasi yang dijelaskan tiap komponen
   - eigenvectors → principal components (PC)

5. Menentukan **jumlah PCA terbaik** menggunakan:

   - **Elbow plot** kumulatif variance explained
   - Pemilihan jumlah PC (misalnya sekitar ratusan PC pertama yang menjelaskan sebagian besar variasi)

6. Menggunakan sejumlah PC terpilih (`PC_use`) sebagai representasi koordinat individu, lalu menghitung:

   - **Matriks jarak genetik**: `dist_matrix = pairwise_distances(PC_use)`

---

## 🧮 Definisi Fungsi Fitness

Fungsi fitness dalam proyek ini didefinisikan sebagai:

> **Jarak genetik antara dua individu** berdasarkan koordinat PCA

```python
def fitness(pair):
    i, j = pair
    return float(dist_matrix[i, j])  # semakin besar, semakin baik
```

- Semakin besar nilai fitness → semakin jauh jarak genetik antar calon induk.
- Tujuan GA: **memaksimalkan** nilai fitness ini.

> Catatan: Di konteks pemuliaan, jarak genetik yang besar diharapkan dapat mengurangi risiko inbreeding dan meningkatkan keragaman genetik keturunan.

---

## 🌱 Inisialisasi Populasi

- Populasi awal berupa **sekumpulan pasangan individu** yang dibentuk secara acak.
- Setiap individu dalam populasi GA direpresentasikan sebagai *pair* `(i, j)` yang berisi indeks dua calon induk dalam `fam`.

```python
def random_pair():
    a = random.randrange(n_samples)
    b = random.randrange(n_samples)
    while b == a:
        b = random.randrange(n_samples)
    return (a, b)
```

Populasi awal berisi `POP` pasangan acak, misalnya 200 pasangan.

---

## ⚙️ Operator GA

### 1. Seleksi (Tournament Selection)

Dipakai untuk memilih pasangan yang akan menjadi “parent” dalam GA. Individu dengan fitness lebih tinggi punya peluang lebih besar untuk terpilih.

```python
def tournament_select(pop, fitness_scores, k=3):
    idxs = random.sample(range(len(pop)), k)
    best = max(idxs, key=lambda i: fitness_scores[i])
    return pop[best]
```

### 2. Crossover

Menggabungkan dua pasangan untuk menghasilkan pasangan baru.

```python
def crossover(p1, p2):
    return (p1[0], p2[1])
```

### 3. Mutasi

Mengubah salah satu atau kedua anggota pasangan secara acak dengan probabilitas tertentu (`mut_rate`).

```python
def mutate(pair, mut_rate=0.1):
    a, b = pair
    if random.random() < mut_rate:
        a = random.randrange(n_samples)
    if random.random() < mut_rate:
            b = random.randrange(n_samples)
    while b == a:
        b = random.randrange(n_samples)
    return (a, b)
```

### 4. Elitism

Sebagian kecil individu terbaik (misalnya top 10%) disalin langsung ke generasi berikutnya tanpa diubah, untuk menjaga kualitas solusi.

---

## 🔄 Proses Evolusi GA

1. **Hitung fitness** seluruh populasi.
2. **Urutkan** populasi berdasarkan fitness (terbesar → terkecil).
3. **Simpan elit** (misalnya 10% teratas).
4. Lakukan **seleksi, crossover, mutasi** untuk mengisi sisa populasi.
5. Ulangi selama sejumlah generasi (`GEN`), misalnya 200 generasi.
6. Simpan riwayat:
   - `best_fitness` per generasi
   - `mean_fitness` per generasi

Notebook juga menampilkan **kurva konvergensi GA**, yang menunjukkan bagaimana nilai best dan mean fitness berkembang dari generasi ke generasi.

---

## 🏁 Hasil Akhir

Setelah proses evolusi selesai, diambil beberapa pasangan terbaik, misalnya **top 10–20 pasangan** dengan jarak genetik terbesar.

Hasil disajikan dalam bentuk `DataFrame` yang berisi:

- indeks individu (`idx1`, `idx2`)
- ID sampel (`ID1`, `ID2`) dari `fam`
- nilai `distance` (jarak genetik)

Contoh kolom hasil:

| idx1 | idx2 | ID1            | ID2            | distance |
|------|-----:|----------------|----------------|----------|
|  123 |  456 | IRIS_313-XXXXX | IRIS_313-YYYYY | 1.0960   |
|  ... |  ... | ...            | ...            | ...      |

---

## 📊 Evaluasi GA

Beberapa bentuk evaluasi yang dilakukan dalam proyek ini:

### 1. GA vs Brute Force (Solusi Optimal Global)

- Mencari jarak **maksimum global** langsung dari `dist_matrix` (brute force).
- Membandingkan pasangan terbaik GA dengan pasangan jarak maksimum global.
- Dalam eksperimen, GA mampu menemukan solusi yang **identik** dengan solusi global optimum (selisih 0%).

### 2. GA vs Baseline (Semua Pasangan)

- Menghitung rata-rata jarak genetik **seluruh pasangan unik** dalam populasi.
- Membandingkan rata-rata jarak top-10/15/20/30 pasangan hasil GA dengan baseline tersebut.
- GA menunjukkan peningkatan yang signifikan (puluhan persen lebih besar dibanding rata-rata semua pasangan).

### 3. Konvergensi GA

- Menampilkan grafik **best fitness** dan **mean fitness** per generasi.
- Menunjukkan bahwa GA mengalami peningkatan pesat di generasi awal lalu mencapai kondisi konvergen.

### 4. Visualisasi di Ruang PCA

- Menampilkan scatter plot PC1 vs PC2 untuk semua individu.
- Menambahkan garis yang menghubungkan pasangan hasil GA.
- Terlihat bahwa pasangan GA umumnya menghubungkan individu yang **berjauhan** di ruang PCA, yang mendukung interpretasi bahwa jarak genetik mereka memang tinggi.

---

## 🚀 Cara Menjalankan Notebook

1. Buka Google Colab atau Jupyter Notebook.
2. Pastikan file data PLINK (`.bed`, `.bim`, `.fam`) telah tersedia di path yang benar.
3. Jalankan sel instalasi dependensi (misalnya `!pip install pandas-plink dask-ml`).
4. Jalankan sel-sel berikutnya secara berurutan:
   - pembacaan data genotipe
   - pembuatan GRM dan PCA
   - pembuatan matriks jarak (`dist_matrix`)
   - inisialisasi dan eksekusi GA
   - evaluasi dan visualisasi hasil
