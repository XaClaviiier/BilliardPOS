# CerdikPOS: Spesifikasi sistem

**Arsip spesifikasi awal, bukan acuan implementasi saat ini.** Isi dan diagram di bawah merekam rancangan lama. Gunakan [kebutuhan terkini](plan/requirement.md), [stack dan kontrak](plan/stack.md), [desain](plan/design.md), [model data](plan/database.md) dan [keamanan](plan/security.md). Referensi penyelarasan adalah folder `cerdikpos-offline-first_UPDATE/`; arsip ZIP dan sketsa koneksi lama bukan sumber aturan baru. Tidak ada klaim implementasi atau hasil uji dalam dokumen ini.

## 1\. Cakupan

CerdikPOS adalah aplikasi kasir berbasis web (PWA) untuk **satu kafe billiard**. Kasir mencatat penjualan kafe dan sewa meja. Direktur memantau omzet, mengelola inventaris, dan memeriksa catatan yang perlu diverifikasi. Pencatatan rutin kasir tidak memerlukan persetujuan direktur satu per satu. Setelah pemasangan dan serah terima, kafe mengoperasikan sistem.

- **Satu perangkat kasir aktif.** Tidak terbatas pada tablet. Perangkat, OS, dan browser yang didukung perlu dibuktikan lewat uji lokasi.  
- Satu PC server di kafe, jaringan lokal (LAN/Wi-Fi), dan akses privat direktur melalui NetBird. Tidak ada VPS maupun basis data publik.  
- Kafe menyiapkan kebutuhan di lokasinya dan data awal (daftar barang, stok, harga, dan tarif meja). Penyedia memeriksa kesiapan dan kesesuaiannya saat pemasangan dan uji lokasi.  
- Di luar cakupan: multi-cabang/SaaS, beberapa kasir offline, membership/loyalty, promo, reservasi, pemesanan mandiri, resep bahan baku/BOM, split bill atau pembayaran campuran, payment gateway dan pengembalian dana otomatis, serta jaminan integrasi printer thermal. Cetak dari browser belum menjadi fitur yang dijanjikan.

## 2\. Hak akses dan layar

| Area | Kasir | Direktur |
| :---- | :---- | :---- |
| Penjualan dan meja | Kelola katalog, kategori, metadata produk dan meja; jalankan pesanan, pembayaran, sewa, perpanjangan, dan pelepasan meja. | Lihat laporan; tidak menjalankan transaksi kasir. |
| Inventaris barang | Lihat stok, catat barang masuk dan penyesuaian stok dengan alasan. Buat kode barang (SKU) dan harga awal produk baru. Tidak dapat menonaktifkan atau mengaktifkan kembali produk. | Lihat saldo dan riwayat per kode barang, bandingkan stok fisik dengan catatan, catat barang masuk dan penyesuaian stok dengan alasan, serta tambah, ubah, nonaktifkan, atau aktifkan kembali produk. Pencatatan rutin kasir tidak memerlukan persetujuan direktur. |
| Pengaturan | Tidak dapat mengubah harga produk yang sudah ada, tarif sewa, durasi pembersihan, atau zona waktu. | Kelola akun, harga produk yang sudah ada, tarif sewa, durasi pembersihan, dan zona waktu saat terhubung ke server. |
| QRIS dan catatan | Periksa bukti bayar, simpan foto secara manual di luar CerdikPOS, konfirmasi pembayaran di aplikasi saat server terjangkau, dan catat penanganan jika pesanan tidak bisa dipenuhi. | Verifikasi tanggal pembukuan jika bukti hanya memuat tanggal. Tinjau selisih stok dan pengecualian waktu atau otorisasi sesuai jenisnya. |
| Perangkat | Operasikan satu perangkat aktif di lokasi kafe. Kasir baru memakai akun yang ditentukan direktur. Catat kondisi saat pergantian perangkat dan cocokkan transaksi lama di lokasi. | Direktur atau petugas yang ditunjuk mendaftarkan perangkat kasir pengganti melalui NetBird di perangkat itu, lalu menyiapkan akses akun kasir di CerdikPOS. Hanya direktur yang bisa mengakses aplikasi dari luar kafe, dengan sesi terpisah di masing-masing perangkat terdaftar. |
| Laporan | Tidak punya akses ke laporan omzet lengkap atau ekspor. | Lihat omzet, detail transaksi, audit, dan ekspor CSV. |

**Layar kasir:** katalog/keranjang, pembayaran, grid meja dan timer, stok dan data master, status offline/antrean sinkronisasi, ekspor/pemulihan antrean, serta pencatatan pengecualian QRIS.

**Layar direktur:** dashboard dan filter tanggal, laporan/CSV, pengelolaan inventaris, pengaturan dan akun, penggantian perangkat, catatan yang perlu diverifikasi, dan audit. Tampilan responsif untuk perangkat yang didukung.

## 3\. Penjualan, pembayaran, dan inventaris

### 3.1 Tunai

1. Saat pelanggan memesan, kasir mencari produk di katalog jual. Jika produk tidak tampil atau stok tidak cukup, jangan masukkan produk itu ke pesanan atau terima pembayaran untuknya. Pakai katalog dan stok server saat terhubung, atau salinan terakhir yang tersedia di perangkat dalam mode offline yang didukung.  
2. Pesanan kafe dibayar terpisah dan boleh dikaitkan dengan sesi billiard. Untuk pembayaran tunai langsung di kasir, masukkan uang yang diterima. Sistem menolak uang kurang dan menghitung kembalian (uang diterima dikurangi tagihan). Kasir menyerahkan kembalian saat itu juga. Perhitungan ini juga berlaku untuk pembayaran sewa tunai, bukan QRIS atau pengembalian dana setelah transaksi.  
3. Simpan penjualan, pembayaran, perubahan stok atau sewa, dan catatan audit secara atomik pada penyimpanan yang sedang dipakai. Tampilkan sukses hanya setelah penyimpanan tersebut mengonfirmasi transaksi tersimpan.  
4. Kasir menyerahkan barang atau menjalankan sewa setelah konfirmasi penyimpanan. Transaksi lunas tidak diedit/dihapus; catatan administratif tidak mengubah omzet.

Contoh: tagihan Rp85.000, uang diterima Rp100.000, kembalian Rp15.000. Omzetnya Rp85.000.

### 3.2 QRIS manual (Q1)

1. Sebelum meminta pelanggan membayar QRIS, kasir memastikan produk bisa dipilih di katalog dan stoknya cukup, atau meja tersedia. Pesanan belum dianggap lunas sebelum kasir mengonfirmasi. Jangan mengurangi stok atau memulai timer sewa dari pesanan yang belum dikonfirmasi.  
2. Pelanggan memperlihatkan **bukti pembayaran**. Kasir memeriksanya secara manual, menyimpan foto bukti di luar CerdikPOS sesuai prosedur kafe, lalu menekan **konfirmasi** di aplikasi jika bukti diterima dan server terjangkau. CerdikPOS tidak mengunggah atau menyimpan foto bukti QRIS, mengecek dana otomatis, terhubung ke payment gateway, atau mewajibkan pemeriksaan aplikasi merchant.  
3. Selesaikan pesanan jika stok atau meja tersedia. Jika pembayaran sudah dikonfirmasi tetapi pesanan tidak bisa dipenuhi, kasir mencatat penggantian atau pengembalian dana manual dengan alasan dan rujukan transaksi asal. Pengembalian uang dilakukan di luar aplikasi. Simpan catatan pengembalian terpisah dari penjualan asli; bukti foto tetap di luar aplikasi.  
4. Gunakan tanggal pembayaran yang terlihat pada bukti tepercaya, bukan tanggal kasir menekan konfirmasi jika keduanya berbeda. Jika bukti hanya memuat tanggal, direktur memeriksa tanggal sebelum pembukuan difinalkan. Jangan mengarang jam transaksi. Layanan boleh berjalan jika stok atau meja tersedia. Pemeriksaan tanggal bukan persetujuan pembayaran per transaksi.

**Uji QRIS manual dilakukan di lokasi**, bukan syarat demo final. Jika PC server tidak terjangkau, aplikasi tidak menerima konfirmasi QRIS dan tidak mengantrekannya sebagai transaksi offline. Mode offline tetap untuk penjualan tunai dan sewa yang didukung. Konfirmasi kasir bukan konfirmasi dana dari bank atau merchant.

### 3.3 Inventaris

- SKU adalah kode unik untuk satu jenis atau ukuran barang, misalnya `AIR-600` untuk air mineral 600 ml. Tolak kode yang sudah dipakai, termasuk kode produk nonaktif; beri peringatan jika nama mirip. Kasir boleh memasukkan harga awal produk baru, tetapi tidak boleh mengubah harga produk yang sudah ada.  
- **Barang masuk** menambah jumlah sesuai barang yang diterima. **Penyesuaian stok** mencocokkan catatan dengan hasil hitung fisik: jika aplikasi mencatat 20 botol tetapi hanya ada 18, masukkan 18 dan alasan; sistem mencatat pergerakan −2. Jika penyesuaian salah, buat catatan penyesuaian baru. Jangan timpa riwayat, ubah penjualan lama, atau hitung ulang omzet. Penjualan lunas mengurangi stok sekali dan saldo tidak boleh negatif.  
- Jika saldo berubah sejak layar penyesuaian dibuka, server menolak penyimpanan berdasarkan saldo lama. Tampilkan saldo terbaru dan minta pengguna memeriksa jumlah fisik terkini sebelum mengirim ulang penyesuaian. Pengecekan dan penyimpanan dilakukan dalam satu transaksi database.  
- Saat terhubung ke server, direktur dapat menambah dan mengubah produk, serta menonaktifkan atau mengaktifkannya kembali dengan alasan. Katalog jual tidak menampilkan produk nonaktif; server juga menolak produk itu pada transaksi baru meski masih ada di keranjang lama. Setelah katalog diperbarui, produk yang diaktifkan kembali dapat dipilih lagi. Kode barang, saldo, dan riwayat tetap sama.  
- Saat kasir offline, katalog berasal dari salinan terakhir. Produk yang sudah tidak tampil di salinan itu tidak boleh dijual. Perubahan yang dibuat di server baru diketahui perangkat saat tersambung kembali. Jika uang tunai telanjur diterima dan transaksi sudah dicatat offline sebelum perubahan diketahui, simpan bukti transaksi, tandai untuk ditinjau direktur, dan perbarui katalog saat koneksi pulih. Jangan menggandakan pembayaran atau memaksakan saldo negatif. Pengaktifan kembali produk tidak otomatis menutup kasus yang sudah ditandai.  
- Jika stok server tidak cukup atau kode barang bentrok saat transaksi tunai offline dikirim, simpan pembayaran asli sekali dan tandai kasus untuk diperiksa, tanpa memaksakan stok negatif. Kasir atau direktur memeriksa barang fisik dan catatan transaksi, lalu membuat penyesuaian dengan alasan bila perlu. Sebelum menutup kasus, pastikan saldo sesuai hasil hitung fisik dan pengeluaran barang tidak dihitung dua kali. Jangan mencatat ulang pembayaran.

## 4\. Sewa meja billiard

| Langkah | Perilaku |
| :---- | :---- |
| Pilih | Pelanggan datang langsung. Kasir memilih meja yang tersedia dan blok waktu prabayar. Tidak ada reservasi, jeda, atau pemisahan tagihan. |
| Mulai | Konfirmasi pembayaran dan alokasi meja dalam satu transaksi. Timer mulai otomatis setelah keduanya berhasil. |
| Perpanjang | Perpanjangan hanya sebelum waktu yang dibayar habis. Bayar di muka; gunakan tarif yang direkam saat sesi mulai. Tambahkan durasi ke batas waktu sesi, bukan mulai ulang timer. |
| Selesai | Saat waktu habis, meja masuk masa pembersihan. Jika selesai lebih awal, pembersihan dimulai saat kasir menutup sesi; tidak ada pengembalian dana untuk pemakaian yang lebih singkat. |
| Buka kembali | Durasi pembersihan standar 10 menit dan dapat diubah direktur. Meja tersedia lagi setelah durasi berlalu **dan** kasir melepas meja secara manual. Penutupan terlambat tidak memulai ulang masa pembersihan. |

Simpan tarif, batas waktu, dan durasi pembersihan saat sesi dimulai. Perpanjangan dan pembersihan sesi berjalan memakai aturan sesi itu, termasuk saat offline. Durasi baru berlaku untuk sesi berikutnya setelah perangkat menerima pembaruan. Saat terhubung, gunakan waktu server sebagai acuan. Saat offline, bandingkan perubahan jam perangkat dengan waktu berjalan sejak acuan terakhir jika pengukurannya masih tersedia. Jika jam melompat atau acuan tidak dapat dipercaya, tandai waktu untuk diperiksa, pertahankan batas waktu yang sudah tercatat, dan jangan mengubah status meja otomatis berdasarkan jam tersebut.

## 5\. Laporan, audit, dan status transaksi

- Dashboard menampilkan omzet harian atau rentang tanggal, rincian kafe vs. billiard, tunai vs. QRIS, stok, detail transaksi, serta ekspor CSV. Omzet tidak memakai nominal uang yang diserahkan atau kembalian sebagai nilai penjualan.  
- Laporan dan ekspor CSV membedakan **penjualan**, **pengembalian dana**, dan **penjualan setelah pengembalian** (penjualan dikurangi pengembalian dalam rentang laporan yang sama). Catat penjualan pada tanggal penjualan dan pengembalian pada tanggal uang dikembalikan, dengan rujukan ke penjualan asli. Contoh: penjualan Rp100.000 pada Senin dan pengembalian Rp20.000 pada Selasa ditampilkan pada tanggal masing-masing; laporan Senin–Selasa menunjukkan Rp80.000 setelah pengembalian.  
- Gunakan tanggal transaksi asli jika waktunya bisa dipercaya. Sinkronisasi terlambat dapat memperbarui laporan historis; tampilkan kapan data terakhir diperbarui. Jika waktu tidak dapat dipercaya, tandai untuk verifikasi.  
- Zona usaha dapat dipilih dari WIB/WITA/WIT. Simpan waktu aktual dalam UTC jika diketahui serta tanggal usaha dan zona yang berlaku. Perubahan zona tidak memindahkan transaksi yang sudah dibukukan. Bukti QRIS yang hanya berisi tanggal tidak diberi jam buatan.  
- Bedakan **status pembayaran, pemenuhan pesanan, sinkronisasi, verifikasi catatan, pembukuan omzet, dan status meja**. Misalnya, tunai sudah tersimpan di perangkat tetapi belum masuk server; QRIS sudah dikonfirmasi kasir tetapi tanggal pembukuannya masih menunggu verifikasi.  
- Dashboard membedakan data terkini, data lama/tidak tersedia, dan catatan yang menunggu rekonsiliasi. Audit mencatat perubahan penting, pelaku, dan waktu, bukan setiap kali layar dibuka. Koreksi tidak menimpa catatan audit sebelumnya.  
- Direktur dapat melihat riwayat per kode barang. Untuk pembuatan dan perubahan produk, harga, stok, serta status aktif, catat pengguna, waktu, kode barang, jenis tindakan, dan nilai sebelum/sesudah yang relevan. Pada penyesuaian stok, catat juga selisih dan alasan; pada barang masuk dan perubahan status produk, catat alasannya. Simpan perubahan beserta jejak audit dalam satu transaksi PostgreSQL tanpa menimpa catatan sebelumnya.

## 6\. Arsitektur dan cara sistem menangani gangguan

[Diagram arsitektur utama](http://CerdikPOS_Diagram_Koneksi/05_arsitektur_koneksi.png) · [Internet putus](http://CerdikPOS_Diagram_Koneksi/01_internet_mati.png) · [PC tidak terjangkau](http://CerdikPOS_Diagram_Koneksi/02_server_mati.png) · [Akses direktur terputus](http://CerdikPOS_Diagram_Koneksi/03_akses_direktur_terputus.png) · [Beberapa perangkat direktur](http://CerdikPOS_Diagram_Koneksi/04_direktur_multi_device.png)

| Komponen | Rancangan |
| :---- | :---- |
| PWA | TypeScript, Service Worker, dan Web App Manifest. Simpan tampilan dasar dan data minimum untuk penggunaan offline. React/Vite dan Tailwind/shadcn masih pilihan teknis. Node.js untuk build, bukan server produksi kedua. |
| Server dan data | Go \+ chi dan PostgreSQL di PC lokal. PostgreSQL adalah sumber utama data transaksi, stok, sewa, pembayaran, audit, dan laporan. API dan proses sinkronisasi memakai aturan bisnis yang sama. |
| Perangkat kasir | IndexedDB untuk salinan data kerja dan antrean perubahan yang belum dikirim (*outbox*). Bukan pengganti database server. |
| Jaringan | LAN/Wi-Fi tetap berjalan saat ISP putus. Hostname dan HTTPS tepercaya harus berfungsi di perangkat kasir tanpa internet publik. Jangan mengandalkan akses HTTP polos ke alamat IP untuk fungsi offline PWA. |
| Akses kasir dan direktur | Akses online kasir hanya dari perangkat aktif di LAN kafe; server menolak akses kasir dari jalur remote. Hanya direktur yang dapat mengakses dari luar lewat NetBird, lalu masuk ke CerdikPOS dengan sesi terpisah pada setiap perangkat yang didaftarkan. Pendaftaran perangkat kasir di NetBird tidak memberi hak akses remote. Jangan membuka port database atau API langsung ke internet publik. |

&nbsp;

| Kondisi | Kasir | Direktur dari luar kafe |
| :---- | :---- | :---- |
| Normal | PWA mengirim ke API lewat LAN; sukses setelah PostgreSQL menyimpan transaksi. | Data terbaru jika jalur NetBird, internet, dan PC sehat. |
| Internet putus; LAN dan PC hidup | Tunai dan rental tetap dicatat ke PostgreSQL lewat LAN. Jangan masuk mode antrean hanya karena internet putus. | Tidak melihat data terkini; tampilkan status koneksi atau label data lama. |
| PC/jaringan lokal tidak terjangkau | Perangkat yang sudah disiapkan bisa menyimpan tunai/rental yang didukung ke IndexedDB, selama penyimpanan bisa ditulis dan sesi offline masih berlaku (maksimal 7 hari). Tandai transaksi belum sinkron. | Jika PC atau jalur remote tidak terjangkau, tidak ada data live. |
| Koneksi ke PC pulih | Kirim ulang antrean dan cocokkan hasil. Status terkirim hanya setelah server benar-benar menyimpan. Pisahkan konflik untuk verifikasi. | Tampilkan data terbaru setelah kembali terhubung; laporan dapat berubah setelah rekonsiliasi. |

### 6.1 Penyimpanan, konflik, dan sesi

- Simpan transaksi tunai/rental, perubahan stok atau proyeksi meja, dan antrean kirim dalam satu transaksi IndexedDB saat fallback. Jika salah satu gagal, jangan tandai transaksi berhasil.  
- Gunakan ID stabil untuk transaksi dan event. Kirim ulang event dengan ID yang sama agar server tidak menggandakan pembayaran, pengurangan stok, atau sesi sewa. Periksa ID jika hasil simpan belum diketahui.  
- Jika perangkat belum siap offline atau masa sesi habis, jangan menerima transaksi baru lewat fallback. Data antrean tetap aman; ekspor darurat terenkripsi tetap tersedia. Perangkat perlu terhubung dan login ulang untuk melanjutkan kerja.  
- Setelah pergantian, server menolak transaksi baru dari perangkat yang tidak lagi aktif atau akun yang telah dicabut. Pemulihan hanya menerima catatan lama melalui prosedur di §7, tanpa mengubah pengguna, perangkat asal, atau ID transaksi. Perangkat lama tidak dipakai lagi untuk berjualan.  
- Saat petugas kasir berganti, direktur menentukan akun untuk petugas baru dan mencabut akses petugas lama untuk transaksi baru. Jangan mengalihkan kepemilikan atau membuang antrean lama; jika dipulihkan, catatan tetap atas nama petugas asli.  
- Jika direktur lupa kata sandi untuk akses NetBird, gunakan reset kata sandi pada layanan login NetBird yang dipakai kafe. Reset ini memulihkan akses jaringan, bukan kata sandi akun CerdikPOS yang terpisah. Prosedur lupa kata sandi CerdikPOS tetap perlu ditetapkan.  
- Mengakhiri sesi direktur pada satu perangkat tidak mengakhiri sesi perangkat lain. Hapus cache privat sesi tersebut tanpa menghapus tampilan dasar PWA atau antrean transaksi kasir. Verifikasi izin kasir/direktur di server, bukan hanya dengan menyembunyikan tombol.

### 6.2 Tunai sudah diterima tetapi belum pasti tersimpan (O-04)

| Situasi | Tindakan |
| :---- | :---- |
| Pasti gagal; layanan belum diberikan | Jangan tandai lunas. Kembalikan uang bila memungkinkan sebelum barang diserahkan atau permainan dimulai. |
| Hasil simpan belum diketahui | Kasir mencatat kejadian secara manual dan mencocokkannya di lokasi. Periksa status di aplikasi dengan ID transaksi yang sama sebelum mencatat ulang atau mengembalikan uang. Catatan kejadian bukan transaksi baru. |
| Penyimpanan pasti gagal; layanan sudah diberikan atau uang tidak dapat dikembalikan saat itu | Kasir mencatat kejadian secara manual dan mencocokkannya di lokasi dengan uang yang diterima dan catatan aplikasi. Jangan buat catatan lunas fiktif atau hitung ulang omzet dan stok. |

Kasir bertanggung jawab atas pencatatan dan pencocokan di lokasi; kafe menentukan media catatan manual sebelum operasional.

## 7\. Backup dan pemulihan

| Yang dilindungi | Cara pemulihan |
| :---- | :---- |
| PostgreSQL | Cadangkan database ke media terpisah dari PC. Uji pemulihan dengan data uji. Foto bukti QRIS disimpan di luar CerdikPOS dan tidak termasuk backup aplikasi. |
| Antrean yang belum sinkron | Ekspor/impor terenkripsi lewat PWA. Jika koneksi belum pulih, kasir mengekspor antrean ke media terpisah dari perangkat kasir saat pergantian shift atau tutup kas. Simpan frasa sandi di luar aplikasi. Backup server tidak mencakup transaksi yang belum sampai ke server. |
| Perangkat dan akses | Dokumentasikan instalasi ulang, jaringan lokal, HTTPS, pendaftaran perangkat kasir pengganti dan perangkat direktur ke NetBird, serta penyiapan akun kasir. Latih pemulihan tanpa mengandalkan akun pribadi pembuat sistem. |

### 7.1 Perangkat kasir rusak dan transaksi lama belum terkirim

1. Direktur atau petugas yang ditunjuk memasang NetBird di perangkat pengganti dan mendaftarkannya. Setelah itu, siapkan login kasir di CerdikPOS sebagai langkah terpisah. Akun kasir yang sama boleh dipakai jika orangnya tetap sama. Alihkan perangkat kasir aktif ke perangkat pengganti setelah akses ke server teruji. NetBird untuk pendaftaran tidak menggantikan akses LAN sehari-hari.  
2. Saat beralih, kasir mencatat jam pergantian, uang fisik di laci, dan meja yang masih dipakai. Lanjutkan transaksi baru di perangkat pengganti tanpa menunggu perangkat lama pulih. Pisahkan pencatatan transaksi baru dari pencocokan transaksi lama; jangan menjual ulang sesi meja yang masih berjalan.  
3. Kasir memeriksa catatan lama di server. Jika transaksi sudah tercatat, cocokkan saja tanpa input ulang. Jika belum tercatat dan data asli masih bisa dibaca atau tersedia dalam ekspor, pulihkan melalui perangkat pengganti yang sudah masuk ke CerdikPOS dan terhubung ke server. Pertahankan ID, pengguna, dan perangkat asal agar pengiriman atau impor ulang tidak menggandakan pembayaran, stok, atau sesi sewa. Catat siapa yang menjalankan pemulihan dan kapan.  
4. Jika data asli dan ekspor tidak tersedia, kasir mencocokkan catatan manual dengan uang fisik, catatan pesanan, dan sesi meja. Jika rincian tidak bisa dibuktikan, catat sebagai selisih yang belum terselesaikan, bukan sebagai penjualan baru. Jangan mengklaim data sudah pulih.

Direktur bertanggung jawab atas backup rutin, pemulihan akses, dan penggantian perangkat. Penunjukan petugas adalah keputusan internal kafe di luar aplikasi. Buktikan pemulihan saat pemasangan atau serah terima. Jadwal backup, retensi, batas kehilangan data (RPO), target waktu pulih (RTO), dan akses frasa sandi darurat masih perlu ditetapkan.

## 8\. Arah implementasi

Ini rancangan teknis, bukan kontrak endpoint atau skema database final.

| Area | Data dan batas yang perlu dijaga |
| :---- | :---- |
| Pengguna dan perangkat | Akun, peran, sesi, perangkat kasir aktif, masa sesi offline, persetujuan pergantian, jejak perubahan akses. |
| Katalog, stok, dan meja | Produk/SKU dan status aktifnya, harga, saldo dan pergerakan stok, master meja, serta perubahan tarif dan pengaturan. Status nonaktif tidak memutus referensi historis. |
| Penjualan dan pembayaran | Pesanan, item, pembayaran, catatan pemeriksaan, dan penyelesaian pengecualian. Pengembalian dana dicatat terpisah dengan rujukan ke transaksi asli. Foto bukti QRIS tidak disimpan di aplikasi. Nominal dalam rupiah bilangan bulat; harga item dan tarif disimpan saat transaksi atau sesi dibuat. |
| Sewa dan waktu | Sesi, alokasi meja, riwayat perpanjangan, batas waktu, status pembersihan, tanggal usaha dan zona. |
| Sinkronisasi dan audit | Event, hasil kirim, perubahan yang diunduh, catatan verifikasi, dan audit. ID event unik; penerapan ulang tidak mengubah hasil finansial atau stok. |

Kelompok API yang direncanakan: autentikasi dan pengguna; produk, harga, meja, dan stok; penjualan tunai dan konfirmasi QRIS manual; sewa; laporan, ekspor, dan verifikasi; sinkronisasi dan pergantian perangkat. Rancangan REST `/api/v1` adalah acuan awal. Jangan mengartikan nama endpoint QRIS dari v2.8 sebagai pengecekan dana otomatis.

API memeriksa izin operasi inventaris pada setiap permintaan. Hanya direktur yang dapat menonaktifkan atau mengaktifkan kembali produk. Operasi inventaris tidak menyediakan penghapusan permanen produk atau pergerakan stok.

## 9\. Pembuktian sebelum digunakan

| Skenario | Bukti minimum |
| :---- | :---- |
| Penjualan dan rental | Uji kembalian pada pembayaran tunai langsung untuk penjualan kafe dan sewa; QRIS tidak memiliki kembalian. Pastikan stok berkurang sekali dan timer mulai setelah pembayaran dan alokasi meja. Uji perpanjangan, selesai awal, pelepasan meja, serta sesi offline yang mempertahankan aturan lama saat durasi pembersihan berubah. Jika jam perangkat meragukan, pastikan batas waktu tidak berubah dan meja tidak dilepas otomatis. |
| Internet benar-benar diputus | PC/LAN tetap aktif. PWA dan HTTPS lokal berfungsi; transaksi masuk langsung ke PostgreSQL. Direktur dari luar tidak melihat data lama sebagai data live. |
| PC tidak dapat diakses | Buka ulang PWA yang sudah siap offline; simpan tunai/rental; pulihkan PC dan kirim ulang transaksi. Tidak ada uang, stok, atau sesi sewa yang hilang atau tercatat ganda. |
| Akses dan hak | Uji kasir lokal pada perangkat aktif dan penolakan login serta operasi kasir dari luar LAN, termasuk lewat NetBird. Uji direktur dari beberapa perangkat terdaftar, logout per perangkat, dan pemulihan login NetBird tanpa perubahan akun aplikasi. Saat petugas berganti, akses transaksi baru milik akun lama ditolak; catatan lama tetap memiliki pencatat asli. Kasir tidak dapat mengubah harga yang sudah ada atau mengakses laporan lengkap. |
| Penyesuaian stok dan audit | Sebelum transaksi nyata, pakai data uji: stok 20 dihitung 18\. Pastikan saldo menjadi 18, perubahan −2 tercatat dengan pengguna, waktu, dan alasan, serta penjualan dan omzet lama tetap sama. Ulangi dengan penjualan saat layar penyesuaian terbuka: aplikasi tidak menyimpan angka lama dan meminta pemeriksaan ulang. |
| Status produk dan transaksi offline | Uji produk yang tidak tampil tidak dapat dipesan atau dijual. Uji server menolak produk nonaktif yang masih ada di keranjang lama. Setelah diaktifkan kembali oleh direktur dan katalog diperbarui, produk dapat dipilih dengan kode, saldo, dan riwayat yang sama. Kasir ditolak untuk kedua perubahan status. Uji transaksi tunai yang telanjur tercatat dari salinan offline lama: bukti tetap ada, kasus ditandai, dan pengiriman ulang tidak menggandakan pembayaran. |
| QRIS di lokasi | Kasir memeriksa bukti dan menyimpan foto di luar aplikasi, lalu mengonfirmasi saat server terjangkau. Uji bukti hanya bertanggal, penolakan konfirmasi saat server tidak terjangkau, serta penjualan dan pengembalian manual pada dua tanggal berbeda. Laporan harian mengikuti tanggal masing-masing; laporan gabungan mengurangi pengembalian tanpa mengubah penjualan asli. |
| Pemulihan dan kasus gagal | Uji backup server dan ekspor/impor antrean secara terpisah. Uji kasir mencatat dan mencocokkan sendiri saat uang sudah diterima tetapi hasil simpan belum pasti, tanpa membuat transaksi ganda. Uji juga benturan stok atau kode barang dan jam perangkat yang tidak tepercaya. |
| Perangkat kasir pengganti | Daftarkan perangkat lewat NetBird, masuk ke CerdikPOS, dan alihkan perangkat aktif tanpa menunggu perangkat lama pulih. Uji server menolak transaksi baru dari perangkat lama. Uji tiga kondisi: transaksi sudah ada di server, impor ekspor dengan ID asli tanpa duplikasi, serta tidak ada data/ekspor sehingga kasir hanya mencatat selisih, bukan transaksi baru. |
| Kesiapan lokasi dan data awal | Penyedia memeriksa perangkat, jaringan, akses, dan data awal yang disiapkan kafe; jalankan uji menggunakan perangkat dan jaringan lokasi. |

**Demo** memakai data uji: penjualan tunai, stok, rental, dashboard dasar, dua kondisi gangguan, dan pemulihan tanpa duplikasi. **Uji lokasi** mengulangnya pada perangkat/jaringan kafe serta memeriksa QRIS dan pemulihan dari backup. Catat kondisi uji, langkah, hasil, bukti, dan status **lulus / gagal / belum diuji**. Matriks v2.8 berisi 143 skenario pada saat ditulis, bukan 143 hasil lulus. Perbarui skenario yang masih mengasumsikan tablet atau pengecekan merchant untuk mengikuti dokumen ini.

Tunda penggunaan untuk transaksi pelanggan jika uang, stok, atau sesi rental bisa hilang/tercatat ganda, QRIS bisa dianggap lunas tanpa konfirmasi kasir, atau pemulihan kritis belum terbukti.

## 10\. Keputusan teknis dan operasional yang masih terbuka

| Area | Yang perlu ditentukan |
| :---- | :---- |
| Lokasi dan perangkat | Kafe menyiapkan perangkat, jaringan, dan data awal. Konfirmasi spesifikasi PC, perangkat dan browser, konfigurasi LAN/HTTPS/NetBird, serta daya cadangan untuk uji lokasi. |
| Akun CerdikPOS | Tetapkan prosedur pemulihan jika kata sandi CerdikPOS lupa. Ini terpisah dari reset akses NetBird. |
| Offline dan pencocokan catatan | Tetapkan ambang deteksi perubahan jam dan cara meninjau kasus waktu yang ditandai. Kafe memilih media catatan manual; kasir yang mencatat dan mencocokkan di lokasi. |
| Penyimpanan foto | Kafe menentukan tempat, akses, dan retensi foto bukti bayar di luar aplikasi. |
| Pemulihan | Tetapkan jadwal dan retensi backup, RPO/RTO, akses frasa sandi darurat, serta buktikan pemulihan pada perangkat lokasi. |

## Rujukan

- [Catatan internal v9](http://CerdikPOS-Catatan-Bisnis-Internal-v9.docx): kesepakatan terbaru, peran, alur kasir, serah terima teknis, dan diagram.  
- [Proposal CerdikPOS Visual v1.14](http://CerdikPOS-Proposal-Visual-v1.14.docx): pembanding cakupan yang ditawarkan kepada klien; bukan bukti persetujuan atau hasil uji.  
- [Baseline teknis v2.8](../../PROJEKT/notes/CerdikPOS_Laporan_Internal_Tim_v2.8.docx.md): stack, aturan domain, pemulihan, dan skenario uji. Untuk topik yang diperbarui, ikuti keputusan terbaru di atas, terutama **pemeriksaan QRIS manual oleh kasir**, **PWA tidak terbatas pada tablet**, **NetBird**, dan **prosedur O-04**.