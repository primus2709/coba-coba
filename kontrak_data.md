# Kontrak Data Jaga Hak Jakarta

Versi 1, 14 Oktober 2026. Skema SQL-nya ada di `data/skema_database.sql` dan sudah diuji di PostgreSQL 16.

Dokumen ini adalah kesepakatan bentuk data antara halaman Warga, halaman Kelurahan, workflow n8n, dan database. Anggap saja seperti formulir standar kantor: semua bagian wajib memakai nama kolom dan isian yang sama persis. Kalau ada yang perlu diubah, bicarakan dulu dengan koordinator, jangan mengubah sendiri.

Semua data di sistem ini fiktif. Tidak ada kolom untuk NIK, KK, nomor HP, atau alamat lengkap, dan memang tidak boleh ditambahkan.

## 1. Siapa menulis dan membaca apa

| Tabel atau view | Diisi oleh | Dibaca oleh |
|---|---|---|
| `aturan_program` | Jalur A (PJ A) | Halaman Warga (pemeriksaan program), view antrean |
| `rumah_tangga` | Jalur B (data simulasi) | Halaman Warga (pilihan persona), W4 |
| `pengajuan` | W1 (baru), W0 (chat_id), W2 (status), W3 (penanda pengingat) | Halaman Kelurahan, W4 |
| `log_notifikasi` | Semua workflow yang mengirim pesan | Halaman Kelurahan, W4 |
| `kontak_internal` | Koordinator atau PJ B, sekali saat persiapan | W1, W3, W4, W5 |
| `antrean_kelurahan` (view) | Dihitung otomatis dari `pengajuan` | Halaman Kelurahan, W1 |

Nama workflow:

| Kode | Nama | Pemicu |
|---|---|---|
| W0 | Kotak masuk Telegram | Telegram Trigger (satu-satunya untuk bot ini) |
| W1 | Penerimaan pengajuan | Webhook dari tombol "Ajukan" |
| W2 | Perubahan status | Webhook dari halaman Kelurahan |
| W3 | Pengingat dan eskalasi | Jadwal, setiap 1 menit saat demo |
| W4 | Laporan harian | Jadwal |
| W5 | Kanal komunitas (bonus) | Jadwal; balasan kata kunci ditangani lewat W0 |

W0 ada karena satu bot Telegram hanya bisa dipakai oleh satu Telegram Trigger dalam satu waktu. Karena itu semua pesan yang masuk ke bot lewat satu pintu, lalu dipilah di dalam W0.

## 2. Tabel `aturan_program`

Satu baris untuk satu program. Halaman Warga memakai tabel ini untuk menentukan program yang "dapat diajukan". Tidak ada AI yang terlibat di sini.

| Kolom | Isi | Wajib | Keterangan |
|---|---|---|---|
| kode_program | teks | ya | Huruf besar dan garis bawah. Dipakai: `KJP_PLUS`, `KJMU`, `KLJ`, `KPDJ`, `KAJ`, `USUL_SANGGAH` |
| nama_program | teks | ya | Nama yang tampil ke warga |
| kelompok_sasaran | teks | ya | Salah satu dari: siswa, mahasiswa, lansia, disabilitas, anak_usia_dini, keluarga |
| jenjang_berlaku | teks | tidak | Dipisah koma tanpa spasi, misalnya `SD,SMP,SMA,SMK` atau `PT`. Kosong berarti tidak dibatasi jenjang |
| usia_min, usia_max | angka | tidak | Usia orang yang dibantu |
| min_lama_domisili_tahun | angka | tidak | |
| desil_maks | angka 1-10 | tidak | Dibandingkan dengan desil yang dilihat warga sendiri di Cek Bansos. Sistem tidak menghitung desil |
| tanggal_buka, tanggal_tutup | tanggal (YYYY-MM-DD) | tidak | Kosongkan kalau jadwal tidak berupa tanggal pasti |
| keterangan_jadwal | teks | tidak | Dipakai kalau tanggal dikosongkan |
| langkah_pengajuan | teks | tidak | |
| berkas_wajib | teks | tidak | Dipisah titik koma, misalnya `Kartu Keluarga; Kartu pelajar` |
| sumber_judul | teks | ya | Judul dokumen resmi sumber aturan |
| sumber_tautan | teks | tidak | |
| sumber_tanggal_akses | tanggal | tidak | |
| adalah_contoh | ya/tidak | ya | `true` untuk data contoh, `false` untuk data asli dari PJ A |
| aktif | ya/tidak | ya | Program nonaktif tidak ditampilkan |

Saat ini tabel berisi 6 baris contoh bertanda `adalah_contoh = true` supaya workflow bisa diuji. Angka syarat dan tanggalnya bukan aturan resmi. Templat CSV dengan kolom yang sama ada di `data/aturan_program/templat_aturan_program.csv`. PJ A mengisi spreadsheet dengan judul kolom persis seperti templat, lalu diimpor ke Supabase.

## 3. Tabel `rumah_tangga`

Data simulasi 300 rumah tangga, termasuk persona demonstrasi seperti Bu Lia.

| Kolom | Isi | Keterangan |
|---|---|---|
| id | angka, otomatis | Jangan diisi manual |
| nama_perwakilan | teks, wajib | Nama fiktif anggota keluarga yang mengurus, misalnya "Bu Lia" |
| kelurahan, kecamatan | teks, wajib | Wilayah Jakarta Barat |
| rw, rt | teks | |
| jumlah_anggota | angka | |
| jenjang_anak, usia_anak | teks, angka | Anak utama saja, boleh kosong |
| ada_lansia, ada_disabilitas | ya/tidak | |
| lama_domisili_tahun | angka | |
| desil_menurut_warga | angka 1-10 | Desil yang dilihat warga, bukan hasil hitungan sistem |
| desil_sebelumnya_menurut_warga | angka 1-10 | |
| adalah_persona | ya/tidak | `true` untuk 3-5 tokoh demonstrasi |
| catatan | teks | |

## 4. Tabel `pengajuan`

Jantung sistem. Satu baris untuk satu pengajuan warga.

| Kolom | Diisi oleh | Keterangan |
|---|---|---|
| kode_pengajuan | otomatis | Format `JHJ0001`, `JHJ0002`, dan seterusnya. Dibuat database, jangan dibuat di Streamlit atau n8n |
| sumber_data | W1 atau data simulasi | `stan` (pengajuan dari pengunjung), `simulasi` (data fiktif Jalur B), `uji` (percobaan selama build) |
| id_rumah_tangga | W1 | Diisi kalau pengunjung memakai persona, selain itu kosong |
| nama_pemohon | W1, wajib | Nama fiktif atau nama samaran |
| kelurahan | W1, wajib | |
| kecamatan | W1 | |
| kode_program | W1, wajib | Harus ada di `aturan_program`. Diambil dari hasil pemeriksaan program di halaman Warga |
| jenjang_anak, usia_sasaran | W1 | |
| ada_lansia, ada_disabilitas | W1 | |
| desil_menurut_warga, desil_sebelumnya_menurut_warga | W1 | Angka 1-10 |
| tenggat_program | W1 | Disalin dari `aturan_program.tanggal_tutup` saat pengajuan dibuat |
| kronologi | W1, wajib | Cerita warga dengan kata-katanya sendiri |
| ringkasan_ai | W1 (Gemini) | Ringkasan kronologi 1-2 kalimat untuk petugas. Tidak dipakai untuk keputusan apa pun |
| status | otomatis, lalu W2 | Lihat bagian 5 |
| catatan_petugas, nama_petugas | W2 | |
| telegram_chat_id | W0 | Kosong sampai warga memindai QR dan menekan Start |
| telegram_terhubung_pada | W0 | |
| waktu_pengingat | otomatis | Saat petugas diingatkan. Mode demo: 3 menit setelah pengajuan dibuat |
| batas_tindak_lanjut | otomatis | Saat pengajuan dieskalasi ke atasan. Mode demo: 5 menit |
| pengingat_terkirim_pada, eskalasi_terkirim_pada | W3 | Penanda supaya pesan tidak terkirim dua kali |
| dibuat_pada, diperbarui_pada | otomatis | `diperbarui_pada` berubah sendiri setiap kali baris diubah |

Catatan: kalau penyimpanan gagal (misalnya kode program salah), nomor urut tetap terpakai, jadi kode bisa melompat (JHJ0007 lalu JHJ0010). Ini normal.

## 5. Status pengajuan

Hanya empat nilai ini yang diterima database. Penulisan harus persis, termasuk huruf besar di awal.

| Status | Arti |
|---|---|
| `Diterima` | Pengajuan baru masuk, belum disentuh petugas. Status awal otomatis |
| `Sedang diverifikasi` | Petugas sedang memeriksa berkas |
| `Perlu berkas tambahan` | Warga perlu melengkapi berkas |
| `Diteruskan ke dinas` | Kelurahan selesai memproses dan meneruskan ke dinas terkait |

Sengaja tidak ada status "Ditolak" atau "Disetujui". Keputusan penerima bantuan ada di dinas terkait, bukan di kelurahan dan bukan di sistem ini.

W3 hanya memproses pengajuan berstatus `Diterima`. Begitu petugas mengubah status, pengingat untuk pengajuan itu berhenti.

## 6. Kode pengajuan dan penghubung Telegram

Alurnya:

1. Warga menekan "Ajukan". W1 menyimpan pengajuan dan membalas kode, misalnya `JHJ0051`.
2. Halaman Warga menampilkan kode dan QR berisi `https://t.me/<TELEGRAM_BOT_USERNAME>?start=JHJ0051`. Nama bot diambil dari `.env`.
3. Warga memindai QR lalu menekan Start. Bot menerima pesan teks `/start JHJ0051`.
4. W0 mengambil kata kedua (`JHJ0051`), mengisi `telegram_chat_id` di baris pengajuan itu, lalu membalas nomor tiket.

Format kode aman untuk Telegram: parameter `start` hanya menerima huruf, angka, garis bawah, dan tanda hubung, maksimal 64 karakter.

Pesan lain yang masuk ke W0:
- `/start` tanpa kode: balas sapaan, sekaligus tampilkan chat_id pengirim. Ini dipakai anggota tim untuk mendaftarkan diri di `kontak_internal`.
- `/start` dengan kode yang tidak dikenal: balas bahwa kode tidak ditemukan.
- Kata kunci seperti "KJP" atau "KJMU": balasan otomatis W5 (bonus, dikerjakan belakangan).

Kalau QR yang sama dipindai dua orang, chat_id yang terakhir yang dipakai. Untuk demo ini justru menguntungkan: pengunjung yang memindai terakhir yang menerima kabar.

Nomor tiket tidak bisa dikirim lewat Telegram di langkah 1, karena saat itu chat_id warga belum diketahui. Pesan tiket pertama dikirim W0 di langkah 4.

## 7. Skor prioritas

Dihitung oleh fungsi database `hitung_skor_prioritas`, bukan oleh AI. Input yang sama selalu menghasilkan skor yang sama, dan petugas bisa melihat alasannya.

| Kondisi | Poin |
|---|---|
| Tenggat program 0 sampai 3 hari lagi | 50 |
| Tenggat program 4 sampai 7 hari lagi | 30 |
| Tenggat program 8 sampai 14 hari lagi | 15 |
| Tenggat lebih dari 14 hari, tidak ada tenggat, atau sudah lewat | 0 |
| Ada lansia di rumah | 15 |
| Ada penyandang disabilitas di rumah | 15 |

Kategori: 50 ke atas **Tinggi**, 20 sampai 49 **Sedang**, di bawah 20 **Rendah**. W1 mengirim peringatan ke petugas kalau kategorinya Tinggi.

Hari dihitung menurut tanggal Jakarta. Skor tidak disimpan di tabel, melainkan dihitung ulang setiap kali view `antrean_kelurahan` dibaca, sehingga pengajuan otomatis naik ke atas antrean saat tenggatnya makin dekat.

Contoh hasil uji:

| Pengajuan | Kondisi | Skor | Kategori |
|---|---|---|---|
| Bu Lia | Tenggat 2 hari lagi, tanpa kelompok rentan | 50 | Tinggi |
| Bu B | Tenggat 5 hari lagi, ada lansia dan disabilitas | 60 | Tinggi |
| Pak A | Tanpa tenggat, ada lansia | 15 | Rendah |
| Bu D | Tenggat sudah lewat | 0 | Rendah, bertanda "Lewat tenggat" |

## 8. View `antrean_kelurahan`

Dibaca halaman Kelurahan. Isinya kolom pengajuan ditambah kolom hitungan:

| Kolom tambahan | Keterangan |
|---|---|
| nama_program | Dari `aturan_program` |
| sisa_hari_tenggat | Bisa negatif kalau sudah lewat |
| tanda_tenggat | `Tenggat dekat` (0-7 hari), `Lewat tenggat`, atau kosong |
| kelompok_rentan | Benar kalau ada lansia atau disabilitas |
| skor_prioritas, kategori_prioritas | Lihat bagian 7 |
| telegram_terhubung | Benar kalau warga sudah menekan Start |
| lewat_batas_tindak_lanjut | Benar kalau status masih `Diterima` dan batas waktu sudah lewat |

Urutan tampilan di halaman Kelurahan:

```sql
select * from antrean_kelurahan
order by skor_prioritas desc, tenggat_program asc nulls last, dibuat_pada asc;
```

## 9. Payload webhook

Semua isi dikirim sebagai JSON. Nama kunci harus persis sama dengan nama kolom.

### W1: halaman Warga ke webhook penerimaan pengajuan

Contoh lengkap ada di `docs/contoh_payload/w1_permintaan.json`.

| Kunci | Wajib | Catatan |
|---|---|---|
| sumber_data | ya | `stan` untuk demo, `uji` selama build |
| id_rumah_tangga | tidak | `null` kalau tidak memakai persona |
| nama_pemohon | ya | |
| kelurahan | ya | |
| kecamatan | tidak | |
| kode_program | ya | Hasil pemeriksaan program di halaman Warga |
| jenjang_anak, usia_sasaran | tidak | |
| ada_lansia, ada_disabilitas | ya | `true` atau `false`, bukan teks "ya" |
| desil_menurut_warga, desil_sebelumnya_menurut_warga | tidak | Angka 1-10 atau `null` |
| tenggat_program | tidak | `YYYY-MM-DD` atau `null` |
| kronologi | ya | |

Balasan W1 bila berhasil:

```json
{ "ok": true, "kode_pengajuan": "JHJ0051", "kategori_prioritas": "Tinggi" }
```

Balasan W1 bila data tidak lengkap:

```json
{ "ok": false, "pesan": "Kolom kronologi belum diisi." }
```

W1 membalas segera setelah data tersimpan, sebelum Gemini membuat ringkasan, supaya warga tidak menunggu lama. Ringkasan dan peringatan ke petugas berjalan setelah balasan terkirim.

Halaman Warga tetap harus siap menghadapi kegagalan di luar format di atas (n8n mati, internet putus, balasan bukan JSON). Pasang batas waktu tunggu sekitar 15 detik dan tampilkan pesan yang sopan.

Halaman Warga sebaiknya tidak menampilkan tombol "Ajukan" untuk program yang tenggatnya sudah lewat.

### W2: halaman Kelurahan ke webhook perubahan status

Contoh lengkap ada di `docs/contoh_payload/w2_permintaan.json`.

| Kunci | Wajib | Catatan |
|---|---|---|
| kode_pengajuan | ya | |
| status_baru | ya | Salah satu dari empat status di bagian 5 |
| catatan_petugas | tidak | Ikut dikirim ke warga |
| nama_petugas | tidak | |

Balasan W2:

```json
{ "ok": true, "notifikasi_terkirim": true }
```

Kalau warga belum menghubungkan Telegram, status tetap tersimpan:

```json
{ "ok": true, "notifikasi_terkirim": false, "pesan": "Warga belum menghubungkan Telegram." }
```

## 10. Perintah SQL inti per workflow

Semua perintah di bawah sudah diuji. `$1`, `$2`, dan seterusnya adalah parameter yang diisi dari data workflow, jangan disambung langsung ke teks SQL.

**W0, menghubungkan Telegram.** `$1` = chat_id, `$2` = kode dari pesan `/start`. Kode huruf kecil tetap dikenali. Kalau hasilnya 0 baris, artinya kode tidak ditemukan.

```sql
update pengajuan
set telegram_chat_id = $1, telegram_terhubung_pada = now()
where kode_pengajuan = upper(trim($2))
returning kode_pengajuan, nama_pemohon, kode_program, status;
```

**W1, menyimpan pengajuan.** Kolom `kode_pengajuan`, `status`, dan semua kolom waktu tidak perlu diisi.

```sql
insert into pengajuan (sumber_data, id_rumah_tangga, nama_pemohon, kelurahan, kecamatan,
    kode_program, jenjang_anak, usia_sasaran, ada_lansia, ada_disabilitas,
    desil_menurut_warga, desil_sebelumnya_menurut_warga, tenggat_program, kronologi)
values ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14)
returning kode_pengajuan;
```

Lalu membaca kategori prioritas:

```sql
select kode_pengajuan, skor_prioritas, kategori_prioritas
from antrean_kelurahan where kode_pengajuan = $1;
```

Lalu menyimpan ringkasan Gemini:

```sql
update pengajuan set ringkasan_ai = $2 where kode_pengajuan = $1;
```

**W2, mengubah status.** Hasilnya membawa `telegram_chat_id` untuk mengirim kabar ke warga.

```sql
update pengajuan
set status = $2, catatan_petugas = $3, nama_petugas = $4
where kode_pengajuan = $1
returning kode_pengajuan, nama_pemohon, kode_program, status, catatan_petugas, telegram_chat_id;
```

**W3, pengingat.** Menandai dan mengambil sekaligus, jadi satu pengajuan tidak pernah diingatkan dua kali. Data simulasi diabaikan supaya tidak terjadi banjir pesan.

```sql
update pengajuan
set pengingat_terkirim_pada = now()
where status = 'Diterima'
  and sumber_data <> 'simulasi'
  and pengingat_terkirim_pada is null
  and now() >= waktu_pengingat
returning kode_pengajuan, nama_pemohon, kode_program, batas_tindak_lanjut;
```

**W3, eskalasi.** Sama, dengan kolom penanda dan batas yang berbeda.

```sql
update pengajuan
set eskalasi_terkirim_pada = now()
where status = 'Diterima'
  and sumber_data <> 'simulasi'
  and eskalasi_terkirim_pada is null
  and now() >= batas_tindak_lanjut
returning kode_pengajuan, nama_pemohon, kode_program;
```

**Penerima pesan di sisi pemerintah.** `$1` = `petugas`, `atasan`, `pimpinan`, atau `rt_rw`.

```sql
select nama, telegram_chat_id from kontak_internal where peran = $1 and aktif = true;
```

**Mencatat setiap pesan.** Dipakai semua workflow setelah node Telegram.

```sql
insert into log_notifikasi (kode_pengajuan, workflow, jenis, penerima, telegram_chat_id, isi_pesan, berhasil, pesan_error)
values ($1, $2, $3, $4, $5, $6, $7, $8);
```

Nilai `jenis`: tiket, alert_prioritas, status, pengingat, eskalasi, laporan_harian, komunitas, balasan_otomatis. Nilai `penerima`: warga, petugas, atasan, pimpinan, rt_rw.

Perintah untuk W4 (laporan harian) ditulis bersama penjelasan W4.

## 11. Koneksi ke database

| Pemakai | Cara terhubung | Rahasia disimpan di |
|---|---|---|
| Streamlit (PC stan dan laptop) | Connection string jenis Session pooler, port 5432 | `.env`, kunci `DATABASE_URL` |
| n8n | Node Postgres dengan data dari connection string yang sama | Credential n8n |

Connection string Session pooler diambil dari tombol **Connect** di dashboard Supabase. Bentuknya:

```
postgresql://postgres.<project_ref>:<PASSWORD>@aws-0-<region>.pooler.supabase.com:5432/postgres
```

Perhatikan nama penggunanya `postgres.<project_ref>`, bukan sekadar `postgres`. Jenis ini dipakai karena bisa diakses lewat jaringan IPv4 biasa, sedangkan koneksi langsung ke Supabase hanya mendukung IPv6.

Tips: buat password database dari huruf dan angka saja, tanpa simbol. Simbol seperti `@` atau `#` di dalam connection string harus diubah ke kode khusus dan sering menjadi sumber error koneksi.

Keamanan: semua tabel sudah diaktifkan Row Level Security tanpa aturan tambahan. Artinya kunci publik Supabase (publishable atau anon) tidak bisa membaca apa pun, termasuk lewat view. Ini sudah diuji. Streamlit dan n8n tetap bisa membaca dan menulis karena memakai connection string database. Connection string dan password hanya dibagikan ke PJ B dan PJ C lewat pesan pribadi.

## 12. Aturan untuk data simulasi (Jalur B)

- Isi `sumber_data` dengan `simulasi` untuk semua 50 pengajuan fiktif.
- Jangan mengisi `kode_pengajuan`. Biarkan database yang membuat.
- Isi `dibuat_pada` dengan tanggal acak dalam 14 hari terakhir supaya antrean terlihat wajar. Isi juga `waktu_pengingat` dan `batas_tindak_lanjut` secara wajar (misalnya 1 dan 2 hari setelah `dibuat_pada`).
- Sebarkan status: sebagian `Diterima`, sebagian status lain.
- `tenggat_program` mengikuti tanggal tutup program di `aturan_program`, supaya sebagian pengajuan bertanda "Tenggat dekat".
- `telegram_chat_id` dibiarkan kosong. Data simulasi tidak boleh mengirim pesan ke siapa pun.
- Isi rumah tangga persona dengan `adalah_persona = true`.

## 13. Perintah bantu

Menghapus semua data uji selama build (log notifikasinya ikut terhapus):

```sql
delete from pengajuan where sumber_data = 'uji';
```

Mengganti durasi pengingat dan eskalasi (contoh: kembali ke ukuran kerja nyata):

```sql
alter table pengajuan alter column waktu_pengingat set default now() + interval '1 day';
alter table pengajuan alter column batas_tindak_lanjut set default now() + interval '2 days';
```

Menghapus data pengunjung setelah acara selesai:

```sql
delete from pengajuan where sumber_data = 'stan';
```

## 14. Yang masih terbuka

- Tabel aturan asli dari PJ A belum masuk. Baris contoh dihapus setelah tabel asli siap.
- Cara mengirim parameter dari n8n: teks yang mengandung koma (misalnya kronologi) berpotensi terpotong kalau dikirim lewat kolom parameter berbentuk daftar dipisah koma. Cara pengisian yang aman di node Postgres n8n akan dicek dan ditulis di penjelasan W1 dan W2.
- Pengaturan SSL di credential Postgres n8n dicek saat uji koneksi pertama.
- Proyek Supabase gratis dijeda kalau aktivitas databasenya rendah selama sekitar 7 hari. Selama 14 sampai 23 Oktober database dipakai terus, tetapi tim inti tetap perlu membuka aplikasi atau menjalankan workflow setiap hari pada 19 sampai 21 Oktober.
