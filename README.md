# UTS Graf Sains Data --- Neo4j dan Sistem Rekomendasi

Proyek ini mengolah **RetailRocket E-commerce Dataset** untuk membangun
graf interaksi pengunjung dan item menggunakan **Neo4j**, mengeksplorasi
struktur graf menggunakan **Cypher** dan **NetworkX**, menghitung
kemiripan antar-Visitor dengan **Jaccard similarity**, lalu membuat
rekomendasi item menggunakan **K-Nearest Neighbors (KNN)** dan
mengevaluasinya dengan **Hit@5** serta **NDCG@5**.

> README ini mengikuti alur notebook
> `UTS_Graf_Sains_Data_Neo4j_Rekomendasi_updated.ipynb`. Angka hasil
> yang tercantum adalah hasil dari eksekusi yang tersedia pada notebook;
> hasil lokal dapat berbeda jika data, rentang tanggal, isi database,
> atau versi kode berbeda.

## Daftar isi

1.  [Tujuan proyek](#tujuan-proyek)
2.  [Teknologi](#teknologi)
3.  [Struktur folder](#struktur-folder)
4.  [Prasyarat](#prasyarat)
5.  [Menyiapkan environment Python](#menyiapkan-environment-python)
6.  [Menyiapkan Neo4j](#menyiapkan-neo4j)
7.  [Menyiapkan dataset](#menyiapkan-dataset)
8.  [Menjalankan notebook dari awal sampai
    akhir](#menjalankan-notebook-dari-awal-sampai-akhir)
9.  [Alur analisis dan metode](#alur-analisis-dan-metode)
10. [Query Cypher untuk pemeriksaan dan
    visualisasi](#query-cypher-untuk-pemeriksaan-dan-visualisasi)
11. [Hasil yang tercatat pada
    notebook](#hasil-yang-tercatat-pada-notebook)
12. [Troubleshooting](#troubleshooting)
13. [Checklist sebelum dikumpulkan](#checklist-sebelum-dikumpulkan)
14. [Keamanan dan batasan](#keamanan-dan-batasan)

## Tujuan proyek

-   Memahami koneksi Python ke Neo4j dan penggunaan bahasa query Cypher.
-   Membagi event menjadi periode data training (**P1**) dan testing
    (**P2**).
-   Menyimpan event sebagai node dan relasi graf.
-   Menghitung komposisi node dan relasi untuk eksplorasi data.
-   Membuat Vector Space Model (VSM) untuk Visitor dan Item.
-   Membandingkan proyeksi graf User dan Item dengan NetworkX.
-   Menghitung kemiripan antar-Visitor dan menyimpan relasi
    `SIMILAR_TO`.
-   Menghasilkan rekomendasi item berbasis tetangga terdekat.
-   Mengukur kualitas rekomendasi menggunakan Hit@5 dan NDCG@5.

## Teknologi

-   Python 3.10
-   Jupyter Notebook
-   Neo4j Python Driver (`neo4j`)

## Struktur folder

Siapkan folder proyek dengan susunan seperti berikut:

``` text
proyek-uts-graf/
├── myvenv/
├── Source_Code.ipynb
├── data/
│   ├── category_tree.csv
│   ├── item_properties_part1.csv
│    ├── item_properties_part2.csv
│   ├── events.csv
│   ├── events_p1.csv       # dibuat oleh notebook
│   └── events_p2.csv       # dibuat oleh notebook
│
│
├── README.md
└── requirements.txt        
```

`events.csv` adalah file sumber. File `events_p1.csv` dan
`events_p2.csv` dihasilkan pada tahap wrangling. Pastikan file sumber
berada tepat di folder `data/`.

## Prasyarat

1.  Python terpasang dan dapat dipanggil dari terminal.
2.  Jupyter Notebook atau JupyterLab tersedia.
3.  Neo4j berjalan dan database yang akan digunakan aktif.
4.  Dataset RetailRocket tersedia secara lokal.
5.  Ruang penyimpanan cukup untuk file CSV dan proses impor graf.

Notebook menggunakan database bernama `neo4j` dan koneksi lokal Bolt.
Sesuaikan konfigurasi jika instalasi Neo4j milikmu berbeda.

## Menyiapkan environment Python

Buka terminal di folder proyek. Pada Windows PowerShell, jalankan:

``` powershell
py -m venv myvenv
.\myvenv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -m requirements.txt
```


Lalu jalankan Jupyter:

``` powershell
jupyter notebook
```

Buka `UTS_Graf_Sains_Data_Neo4j_Rekomendasi_updated.ipynb` dari halaman
Jupyter. Pastikan kernel notebook menggunakan environment `.venv` yang
baru dibuat. Jika paket tidak terdeteksi, pasang `ipykernel` di
environment dan daftarkan kernel:

``` powershell
pip install ipykernel
python -m ipykernel install --user --name uts-graf --display-name "Python (uts-graf)"
```

Pilih kernel **Python (uts-graf)** di notebook.

## Menyiapkan Neo4j

1.  Jalankan Neo4j Desktop atau layanan Neo4j yang digunakan.
2.  Pastikan database aktif.
3.  Buka Neo4j Browser dan uji login dengan kredensial milikmu.
4.  Periksa alamat Bolt. Untuk server lokal standar, alamat biasanya
    `bolt://127.0.0.1:7687`.
5.  Sesuaikan variabel konfigurasi di bagian awal notebook:

``` python
URI = "bolt://127.0.0.1:7687"
AUTH = ("neo4j", "PASSWORD")
DATABASE = "neo4j"
```

**Penting:** gunakan URI Bolt yang valid dan password lokal milikmu.
Jangan unggah password asli ke repositori atau laporan publik. Jika
konfigurasi di notebook berbeda dengan contoh ini, samakan dengan alamat
yang benar-benar digunakan oleh instalasi Neo4j.

Notebook memanggil `driver.verify_connectivity()` untuk memeriksa
koneksi. Jika berhasil, notebook akan menampilkan pesan koneksi
berhasil.

### Catatan database

Bagian impor menghapus relasi interaksi lama bertipe `VIEWED`,
`ADDED_TO_CART`, dan `BOUGHT` sebelum mengimpor P1 kembali. Ini membantu
mencegah penumpukan relasi interaksi dari eksekusi berulang, tetapi juga
berarti proses tersebut **mengubah data pada database**. Gunakan
database khusus proyek dan jangan jalankan terhadap database penting
tanpa cadangan.

Notebook juga memiliki contoh awal graf `Person` dengan relasi `KNOWS`
untuk mempraktikkan `MERGE` dan `MATCH`.

## Menyiapkan dataset

Notebook mengharapkan file sumber bernama:

``` text
data/events.csv
```

Kolom yang digunakan:

  -----------------------------------------------------------------------
  Kolom                               Keterangan
  ----------------------------------- -----------------------------------
  `timestamp`                         Waktu event dalam Unix epoch
                                      milidetik

  `visitorid`                         ID Visitor

  `event`                             Jenis event, misalnya `view`,
                                      `addtocart`, atau `transaction`

  `itemid`                            ID Item

  `transactionid`                     ID transaksi; dapat kosong untuk
                                      event non-transaksi
  -----------------------------------------------------------------------

Pastikan nama kolom sesuai dengan yang diharapkan kode. Jangan mengubah
nama kolom tanpa menyesuaikan notebook.

### Pembagian P1 dan P2

Notebook memfilter data berdasarkan `timestamp` dalam UTC:

-   **P1 / training:** 31 Mei 2015 sampai 31 Juli 2015, termasuk seluruh
    hari 31 Juli.
-   **P2 / testing:** 9 Agustus 2015 sampai 24 Agustus 2015, termasuk
    seluruh hari 24 Agustus.

Batas akhir dibuat eksklusif, sehingga kode menggunakan awal hari
berikutnya sebagai batas akhir. Rentang ini harus dicocokkan dengan
instruksi resmi dosen sebelum hasil dikumpulkan.

Tahap wrangling membaca CSV dalam potongan (chunks) agar tidak perlu
memuat seluruh dataset sekaligus, kemudian membuat:

-   `data/events_p1.csv`
-   `data/events_p2.csv`

P2 digunakan untuk evaluasi rekomendasi, sedangkan graf interaksi dan
pembentukan rekomendasi menggunakan data P1.

## Menjalankan notebook dari awal sampai akhir

Jalankan cell **secara berurutan dari atas ke bawah**. Jangan
menjalankan cell di tengah sebelum variabel dan proses yang
dibutuhkannya selesai.

### Tahap 1 --- Import library dan uji koneksi

Notebook mengimpor pustaka analisis, visualisasi, graf, machine
learning, dan Neo4j Driver. Jalankan cell konfigurasi untuk memastikan
Python dapat terhubung ke database.

**Hasil yang diharapkan:** pesan koneksi berhasil.

### Tahap 2 --- Tugas pendahuluan graf

Notebook membuat node `Person` dan relasi `KNOWS` sebagai contoh
penggunaan `MERGE` dan `MATCH`. Contoh ini merupakan latihan Cypher
terpisah dari graf e-commerce.

### Tahap 3 --- Wrangling P1 dan P2

Notebook membaca `events.csv`, memvalidasi timestamp, lalu memisahkan
event sesuai rentang tanggal. Periksa jumlah baris P1/P2 dan pastikan
kedua file keluaran tersedia di folder `data/`.

Pada eksekusi yang tersimpan di notebook, hasilnya:

-   P1: **1.324.774 baris**
-   P2: **279.544 baris**

Angka lokal bisa berbeda apabila file sumber atau rentang tanggal
berbeda.

### Tahap 4 --- Impor P1 ke Neo4j

Notebook membuat constraint unik untuk ID Visitor dan Item, lalu
mengimpor data P1 dalam batch berukuran 5.000 baris. Node dan relasi
dibuat dengan pola:

``` text
(:Visitor {visitorId: ...})
(:Item {itemId: ...})
(:Visitor)-[:VIEWED {timestamp: ...}]->(:Item)
(:Visitor)-[:ADDED_TO_CART {timestamp: ...}]->(:Item)
(:Visitor)-[:BOUGHT {timestamp: ..., transactionId: ...}]->(:Item)
```

Pemetaan jenis event:

-   `view` → `VIEWED`
-   `addtocart` → `ADDED_TO_CART`
-   `transaction` → `BOUGHT`

Setelah impor selesai, periksa jumlah node dan relasi dengan query pada
bagian [Query Cypher](#query-cypher-untuk-pemeriksaan-dan-visualisasi).

### Tahap 5 --- Eksplorasi data (EDA)

Notebook menghitung jumlah node `Visitor`, node `Item`, serta jumlah
relasi untuk masing-masing jenis interaksi. Hasil ini digunakan untuk
memahami skala graf setelah impor.

### Tahap 6 --- Membentuk VSM Visitor dan Item

Notebook mengambil interaksi dari Visitor teratas yang dipilih
berdasarkan jumlah interaksi. Matriks interaksi dibentuk dengan:

-   **VSM Visitor:** baris = Visitor, kolom = Item.
-   **VSM Item:** baris = Item, kolom = Visitor.
-   VSM Item adalah transpose dari VSM Visitor.

Pada output notebook yang tersimpan:

-   VSM Visitor: **(200, 22.479)**
-   VSM Item: **(22.479, 200)**
-   Jumlah nilai nonzero: **43.319**

Matriks menggunakan jumlah interaksi per pasangan Visitor--Item. Karena
itu, nilainya tidak harus biner: pasangan dengan beberapa interaksi
dapat memiliki nilai lebih dari satu.

### Tahap 7 --- Graf bipartit dan proyeksi dengan NetworkX

Graf bipartit menghubungkan Visitor dengan Item yang mereka
interaksikan. Proyeksi graf mengubah hubungan dua tipe node menjadi
hubungan satu tipe:

-   **Proyeksi User/Visitor:** dua Visitor terhubung apabila memiliki
    Item yang sama.
-   **Proyeksi Item:** dua Item terhubung apabila memiliki Visitor yang
    sama.

Notebook menggunakan NetworkX untuk membentuk dan memvisualisasikan
graf. Karena graf penuh bisa besar dan padat, tahap
visualisasi/perbandingan menggunakan subset yang dibatasi. Pastikan
membandingkan subset dan kondisi pengukuran yang sebanding.

Untuk perbandingan yang tercatat pada salah satu output notebook, 100
node pada masing-masing proyeksi menghasilkan:

  Karakteristik         Proyeksi User   Proyeksi Item
  ------------------- --------------- ---------------
  Jumlah simpul                   100             100
  Jumlah edge                   2.741           3.474
  Kepadatan graf              0,55374         0,70182
  Rata-rata derajat             54,82           69,48
  Waktu pembentukan     0,04773 detik   0,06366 detik

Waktu dapat berubah antar-eksekusi. Gunakan angka dari satu eksekusi
yang sama pada tabel laporan. Jumlah edge/kepadatan yang lebih besar
menunjukkan graf lebih rapat, bukan otomatis lebih baik atau lebih
efisien.

### Tahap 8 --- Similarity antar-Visitor

Notebook menghitung **Jaccard similarity** berdasarkan himpunan Item
yang berinteraksi dengan setiap Visitor. Secara umum:

\[ J(A,B)=`\frac{|A\cap B|}{|A\cup B|}`{=tex} \]

dengan (A) dan (B) sebagai himpunan Item dari dua Visitor. Nilai
mendekati 1 menunjukkan lebih banyak kesamaan relatif, sedangkan 0
menunjukkan tidak ada Item bersama.

Untuk membatasi komputasi, kode memilih sampai **300 Visitor** dengan
jumlah interaksi tinggi, lalu menyimpan hingga **1.000 pasangan
positif** sebagai relasi `SIMILAR_TO` dengan properti `similarity`. Ini
adalah subset graf, bukan perhitungan seluruh pasangan Visitor dalam
dataset.

### Tahap 9 --- Rekomendasi menggunakan KNN

Notebook membangun matriks Visitor--Item dari P1 dan menggunakan
`NearestNeighbors` dari scikit-learn dengan:

-   `K = 5`
-   metrik jarak `cosine`
-   algoritma pencarian `brute`
-   `random_state`/seed yang ditetapkan agar pemilihan sampel dapat
    direproduksi.

Kode meminta hingga `K + 1` tetangga karena hasil pencarian biasanya
menyertakan Visitor target sendiri. Visitor target kemudian dikeluarkan
dari daftar tetangga. Skor rekomendasi dihitung dari interaksi tetangga
berbobot kemiripan, item yang sudah pernah diinteraksikan target pada P1
dikecualikan, dan fallback popularitas P1 dapat digunakan jika tidak ada
skor positif.

P2 dipakai untuk menentukan item relevan pada tahap evaluasi. Pada
output notebook, Visitor pertama yang ditampilkan adalah **428868**,
dengan tiga item rekomendasi teratas:

1.  `248450`
2.  `344723`
3.  `259571`

Hasil rekomendasi dan tetangga harus diambil dari output terbaru setelah
notebook dijalankan.

### Tahap 10 --- Evaluasi Hit@5 dan NDCG@5

Evaluasi membandingkan rekomendasi dengan item relevan yang muncul di
P2.

-   **Hit@5:** bernilai 1 jika setidaknya satu item relevan muncul pada
    lima rekomendasi teratas; jika tidak, 0.
-   **NDCG@5:** mengukur kualitas urutan rekomendasi, dengan bobot lebih
    besar untuk item relevan yang berada di peringkat lebih tinggi.

Pada output evaluasi yang tersimpan:

  Metrik                         Nilai
  --------------------------- --------
  Rata-rata Hit@5               0,0000
  Rata-rata NDCG@5              0,0000
  Jumlah Visitor dievaluasi         26

Nilai tersebut hanya menggambarkan 26 Visitor yang lolos kriteria
evaluasi pada eksekusi tersebut, bukan seluruh dataset. Nilai nol
menunjukkan tidak ada item relevan pada Top-5 untuk user yang dievaluasi
dalam output itu. Periksa kembali data P2, definisi relevansi, filter
kandidat, dan rekomendasi jika hasil tidak sesuai ekspektasi.

## Query Cypher untuk pemeriksaan dan visualisasi

Jalankan query ini di **Neo4j Browser**. Untuk melihat graf, pilih tab
**Graph** setelah query menghasilkan node dan relasi.

### 1. Menghitung jumlah Visitor dan Item

``` cypher
MATCH (v:Visitor)
RETURN count(v) AS jumlah_visitor;
```

``` cypher
MATCH (i:Item)
RETURN count(i) AS jumlah_item;
```

### 2. Menghitung masing-masing jenis relasi

``` cypher
MATCH ()-[r:VIEWED]->()
RETURN count(r) AS jumlah_viewed;
```

``` cypher
MATCH ()-[r:ADDED_TO_CART]->()
RETURN count(r) AS jumlah_added_to_cart;
```

``` cypher
MATCH ()-[r:BOUGHT]->()
RETURN count(r) AS jumlah_bought;
```

### 3. Melihat contoh graf interaksi sebelum similarity

``` cypher
MATCH (v:Visitor)-[r:VIEWED|ADDED_TO_CART|BOUGHT]->(i:Item)
RETURN v, r, i
LIMIT 100;
```

Query dibatasi hingga 100 relasi agar tampilan tidak terlalu besar. Ini
hanya contoh tampilan, bukan seluruh graf.

### 4. Melihat lima pasangan similarity tertinggi

``` cypher
MATCH (a:Visitor)-[r:SIMILAR_TO]-(b:Visitor)
WHERE a.visitorId < b.visitorId
RETURN
    a.visitorId AS visitor_a,
    b.visitorId AS visitor_b,
    r.similarity AS similarity
ORDER BY similarity DESC
LIMIT 5;
```

Kondisi `a.visitorId < b.visitorId` mencegah pasangan yang sama tampil
dua kali dalam urutan terbalik, dengan asumsi ID Visitor bertipe
numerik.

### 5. Visualisasi relasi similarity sesudah penambahan

``` cypher
MATCH (a:Visitor)-[r:SIMILAR_TO]-(b:Visitor)
RETURN a, r, b
ORDER BY r.similarity DESC
LIMIT 100;
```

Query ini menampilkan hingga 100 relasi similarity dengan nilai
tertinggi. Untuk laporan, ambil screenshot graf sebelum dan sesudah
similarity. Karena query dibatasi, jangan menyebutnya sebagai
visualisasi seluruh graf.

### 6. Menampilkan interaksi dan similarity dalam satu tampilan

``` cypher
MATCH (v:Visitor)-[r:VIEWED|ADDED_TO_CART|BOUGHT]->(i:Item)
WITH v, r, i
LIMIT 100
OPTIONAL MATCH (v)-[s:SIMILAR_TO]-(other:Visitor)
RETURN v, r, i, s, other
LIMIT 100;
```

Query gabungan ini hanya menampilkan sebagian interaksi beserta relasi
similarity yang ditemukan pada Visitor terkait. Jika tampilan menjadi
terlalu padat, gunakan query sebelum dan sesudah secara terpisah.

## Hasil yang tercatat pada notebook

Berikut ringkasan angka dari output notebook yang tersedia. **Jalankan
ulang notebook dan gunakan output aktual** jika hasil lokal berbeda.

  Bagian                                 Hasil pada output tersimpan
  -------------------------------------- -----------------------------
  Baris P1                               1.324.774
  Baris P2                               279.544
  Node Visitor                           684.227
  Node Item                              171.126
  Relasi `VIEWED`                        1.281.023
  Relasi `ADDED_TO_CART`                 32.807
  Relasi `BOUGHT`                        10.944
  VSM Visitor                            (200, 22.479)
  VSM Item                               (22.479, 200)
  Nilai nonzero pada matriks interaksi   43.319
  Hit@5 rata-rata                        0,0000
  NDCG@5 rata-rata                       0,0000
  User evaluasi                          26

**Catatan:** jumlah node dan relasi bergantung pada database yang
digunakan. Jika database telah berisi data lama, menjalankan notebook
pada database yang salah, atau rentang P1/P2 berbeda, hasilnya bisa
berubah.

## Troubleshooting

### Python tidak menemukan paket

Pastikan environment aktif dan pasang paket pada environment yang sama
dengan kernel notebook:

``` powershell
python -m pip install pandas numpy matplotlib networkx scikit-learn tqdm neo4j jupyter
```

### Koneksi Neo4j gagal

-   Pastikan database sedang berjalan.
-   Pastikan URI Bolt dan port benar.
-   Periksa username, password, dan nama database.
-   Uji login di Neo4j Browser.
-   Pastikan notebook menggunakan konfigurasi yang sama dengan instalasi
    Neo4j.

### `events.csv` tidak ditemukan

Periksa bahwa file berada di `data/events.csv`, relatif terhadap folder
kerja notebook. Jalankan notebook dari folder proyek yang benar.

### P1/P2 kosong atau jumlah baris berbeda

-   Periksa nama kolom dan format `timestamp`.
-   Pastikan timestamp menggunakan Unix epoch milidetik sesuai kode.
-   Pastikan rentang tanggal sesuai instruksi dosen.
-   Pastikan file sumber adalah dataset yang benar.

### Impor Neo4j berjalan lama

Dataset berukuran besar dan impor dilakukan bertahap. Tunggu hingga
progress selesai. Jangan memulai beberapa proses impor sekaligus ke
database yang sama.

### VSM atau rekomendasi kosong

Periksa apakah P1 sudah diimpor, apakah query menemukan relasi
interaksi, dan apakah matriks memiliki Visitor/Item. Jalankan cell
secara berurutan dari awal.

### Hasil similarity atau evaluasi berbeda

Perbedaan dapat terjadi karena versi data, isi database, subset yang
dipilih, konfigurasi kode, atau output yang belum diperbarui. Gunakan
output dari satu eksekusi final yang konsisten dan jangan mencampur
angka dari beberapa eksekusi.



