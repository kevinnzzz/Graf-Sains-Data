# Analisis Graf Perilaku Pengunjung E-Commerce dengan Neo4j

Proyek UTS mata kuliah Graf Sains Data. Perilaku pengunjung toko online (melihat produk, memasukkan ke keranjang, membeli) dimodelkan sebagai graf di Neo4j, lalu dipakai untuk menghitung kemiripan antar produk dan membuat rekomendasi produk dengan KNN.

Nama  : Kevin Davindio Atha
NIM   : 23031554173
Kelas : Graf Sains Data 2023B

## Dataset

Data yang dipakai hanya `events.csv` dari dataset RetailRocket (E-commerce dataset). File ini berisi 2.756.101 baris log perilaku pengunjung dengan rentang waktu 3 Mei 2015 sampai 18 September 2015.

| Kolom | Keterangan |
|---|---|
| `timestamp` | Waktu kejadian dalam milidetik sejak 1 Januari 1970 |
| `visitorid` | ID pengunjung |
| `event` | Jenis aksi: `view`, `addtocart`, atau `transaction` |
| `itemid` | ID produk |
| `transactionid` | ID transaksi (hanya terisi untuk event `transaction`) |

Dataset tidak disertakan di repositori. Unduh sendiri, lalu letakkan `events.csv` di folder `data/` (atau sebagai `ecommerce-dataset.zip` di root proyek, nanti diekstrak otomatis oleh notebook).

## Isi Repositori

| File | Keterangan |
|---|---|
| `Pendahuluan-UTS-graf.ipynb` | Latihan pendahuluan: koneksi driver Neo4j, membuat node dan relasi (graf pertemanan), menjawab pertanyaan pendahuluan, dan impor awal `events.csv` ke Neo4j |
| `Soal-UTS-Graf.ipynb` | Pengerjaan soal UTS: wrangling dan EDA, impor ke Neo4j, VSM, similarity item, dan rekomendasi KNN beserta evaluasinya |
| `requirements.txt` | Daftar dependensi (lihat catatan di bagian Instalasi) |

## Model Graf

Simpul:

- `Visitor` dengan properti `visitorId`
- `Item` dengan properti `itemId` (dan `degree`, jumlah visitor unik, yang dihitung kemudian)

Relasi, semuanya berarah dari `Visitor` ke `Item`:

| Relasi | Asal event | Bobot |
|---|---|---|
| `VIEWED` | `view` | 1 |
| `ADDED_TO_CART` | `addtocart` | 3 |
| `BOUGHT` | `transaction` | 5 |

Bobot 1, 3, 5 mengikuti kekuatan niat pengunjung: melihat adalah sinyal paling lemah, keranjang sedang, dan pembelian paling kuat. Relasi `BOUGHT` menyimpan `transactionId`. Setiap relasi menyimpan `timestamp`. Relasi `SIMILAR_TO` (dengan properti `score`) ditambahkan pada tahap similarity.

## Alur Pengerjaan

1. **Pembagian periode.** Data difilter menjadi dua periode lalu disimpan ke CSV.
   - P1 (training): 2015-05-28 sampai 2015-07-28, 1.319.284 baris
   - P2 (testing): 2015-08-06 sampai 2015-08-21, 285.075 baris
2. **Impor ke Neo4j.** Hanya P1 yang diimpor, per batch 10.000 baris dengan `UNWIND` dan `MERGE`. Hasilnya 680.693 simpul `Visitor`, 170.794 simpul `Item`, dan 1.319.284 relasi.
3. **VSM (Vector Space Model).** Interaksi diambil dari Neo4j, diberi bobot, lalu dibentuk menjadi matriks sparse.
   - VSM User `A` berukuran Visitor x Item (680.693 x 170.794)
   - VSM Item `Aᵀ` berukuran Item x Visitor
   - Kepadatan hanya 0,00088 persen, dan 80,26 persen visitor hanya punya satu interaksi, sehingga data sangat sparse dan berekor panjang
4. **Pemilihan proyeksi.** Proyeksi Item-Item dipilih karena simpul dan sisinya lebih sedikit (170.794 simpul dan sekitar 13,0 juta sisi, dibanding 680.693 simpul dan sekitar 23,1 juta sisi pada User-User).
5. **Similarity.** Cosine biner pada proyeksi Item-Item, dihitung langsung dengan Cypher untuk item acuan peringkat ke-20 berdasarkan jumlah visitor (itemId 112782). Lima item teratas disimpan ke graf sebagai relasi `SIMILAR_TO`, lalu graf sebelum dan sesudah similarity divisualisasikan dengan NetworkX.
6. **Rekomendasi KNN (k = 5).** Kemiripan antar user dihitung dengan cosine pada VSM User. Skor item diperoleh dari bobot tetangga terdekat, item yang sudah pernah disentuh dibuang, dan item terpopuler dipakai sebagai cadangan bila rekomendasi kurang dari lima.
7. **Evaluasi.** Rekomendasi dari P1 dibandingkan dengan item yang benar-benar disentuh user di P2 memakai Hit@5 dan NDCG@5.

## Hasil Ringkas

Top 5 item mirip dengan item 112782:

| Item mirip | Visitor sama | Cosine |
|---|---|---|
| 441668 | 55 | 0,1157 |
| 103900 | 21 | 0,1037 |
| 422671 | 22 | 0,1020 |
| 372394 | 4 | 0,0790 |
| 408043 | 3 | 0,0726 |

Evaluasi rekomendasi KNN (k = 5) pada 4.114 user kandidat (punya minimal 2 item di P1 dan muncul di P2):

| Cakupan | Hit@5 | NDCG@5 |
|---|---|---|
| 1 user acak (visitor 130944, SEED = 42) | 0 | 0,0 |
| Rata-rata 200 user acak | 0,0050 | 0,0007 |

Nilai yang rendah wajar untuk data ini. Matriks sangat sparse, sebagian besar user hanya punya satu interaksi, dan item yang disentuh di P2 sering belum ada di P1.

## Instalasi dan Cara Menjalankan

Prasyarat:

- Python 3.11 (dipakai pada pengembangan)
- Neo4j yang berjalan di `bolt://127.0.0.1:7687` (disarankan versi 5.x)
- Jupyter Notebook atau JupyterLab

Instal dependensi:

```bash
pip install neo4j pandas numpy scipy networkx matplotlib tqdm jupyter
```

Catatan: `requirements.txt` saat ini mengunci `neo4j` pada versi 4.x, sedangkan notebook memakai `driver.execute_query` dan `RoutingControl` yang baru tersedia di driver 5.x ke atas (pengembangan memakai versi 6.4.1). Paket seperti `pandas`, `numpy`, `scipy`, `networkx`, `matplotlib`, dan `tqdm` juga belum tercantum. Sebaiknya `requirements.txt` diperbarui sebelum dipakai.

Langkah menjalankan:

1. Jalankan Neo4j dan pastikan koneksi sesuai. Ubah `URI`, `USER`, dan `PASSWORD` di notebook bila berbeda.
2. Letakkan `events.csv` di folder `data/`.
3. Jalankan `Pendahuluan-UTS-graf.ipynb` bila ingin mengikuti latihan pendahuluan.
4. Jalankan `Soal-UTS-Graf.ipynb` dari atas ke bawah. Notebook ini mengosongkan database `neo4j` sebelum impor, jadi jangan dijalankan pada database yang berisi data penting.

Perkiraan waktu: impor P1 sekitar 4 menit pada mesin pengembangan.

## Catatan

- Password Neo4j (`admin123`) tertulis langsung di notebook untuk keperluan lokal. Jangan dipakai di lingkungan nyata, dan sebaiknya ganti dengan `getpass` atau variabel lingkungan sebelum repositori dipublikasikan.
- Nilai acak dikunci dengan `SEED = 42` agar hasil sampel dan evaluasi dapat diulang.
