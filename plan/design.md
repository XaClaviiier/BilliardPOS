# CerdikPOS — Rancangan interaksi

Status: rancangan interaksi hasil validasi 1.3, menunggu persetujuan Phase 1; bukan prototype. Kasir harus dapat membedakan uang yang baru akan diterima, pembayaran yang sudah diterima di lokasi, dan hasil simpan yang belum diketahui. Lihat [aturan bisnis dan AC](requirement.md) dan [kontrak API](stack.md).

## Struktur layar

| Area | Navigasi dan tugas utama |
|---|---|
| Kasir | **Jual** (cari produk, kategori, keranjang, pembayaran), **Meja** (status, mulai, perpanjang, selesai, lepas), **Nota susulan** (cari/cocokkan, input, hasil), **Stok & master** (stok per SKU, barang masuk, hitung fisik, katalog dan meja). Kasus operasional dapat dibuka dari transaksi/nota. Tidak ada menu omzet atau ekspor. |
| Direktur | **Ringkasan** (periode, penjualan/refund/bersih), **Transaksi** (detail dan CSV), **Perlu diperiksa** (tanggal QRIS, stok, uang, waktu/meja), **Inventaris**, **Pengaturan** (tarif, pembersihan, zona, akun dan perangkat), **Audit**. Satu detail memperlihatkan bukti tekstual, pelaku, dan alasan; tidak menampilkan foto dari perangkat kasir. |
| Header bersama | Nama kafe, peran/pengguna, status hubungan dengan server, logout. Dashboard mencantumkan waktu data terakhir diterima. Jangan menyimpulkan server putus hanya dari status internet browser. |

## Alur operasional

**Jual langsung.** Pada layar lebar, katalog di kiri, keranjang dan total di kanan. Pilih meja terkait bila relevan, bukan gabung tagihan. Tampilkan harga × jumlah. Pilih satu metode bayar. Tunai: uang diterima dan kembalian yang terlihat sebelum konfirmasi. QRIS: petugas menyatakan bukti sudah diperiksa; tanggal bukti wajib, jam hanya jika ada, rujukan jika tersedia. Ingatkan SOP foto di luar aplikasi. QRIS bukan tombol cek dana. Ketika menyimpan, nonaktifkan kirim ganda. Sukses menunjukkan nominal, nomor transaksi, status pemenuhan, dan bila relevan status pembukuan; bukan sekadar toast yang menghilang.

**Gangguan.** Jika PC tidak terjangkau tetapi PWA siap, tampilkan “Mode offline — tunai, sewa dan QRIS tersimpan di perangkat ini”. Kasir memeriksa bukti QRIS seperti biasa; jangan tampilkan “dana terverifikasi”. Setelah commit lokal tampilkan **Tersimpan di perangkat — belum masuk server**, ID transaksi dan jumlah antrean. Jangan tawarkan pembayaran kedua. Tombol transaksi yang memerlukan server tidak tersedia. Jika perangkat tidak siap, izin offline habis, atau commit pasti gagal, arahkan ke nota cadangan. Jika respons simpan online hilang, tampilkan **Hasil belum diketahui** dan ID asal; jangan otomatis membuat transaksi lokal baru. Amankan item/meja yang terdampak sampai hasil dapat dipastikan.

**Ganti kasir saat offline.** Header menampilkan nama kasir aktif dan tanggal/jam **Izin perangkat berakhir**, maksimal 7 hari sejak diterbitkan online. Saat persiapan, direktur memilih petugas; masing-masing masuk online pada perangkat ini dan menetapkan PIN pribadi. **Ganti kasir** mengunci layar dan tab lain. Petugas berikutnya memilih namanya, memasukkan PIN, lalu memeriksa nama di header sebelum transaksi berikutnya. PIN salah: layar tetap terkunci. Belum disiapkan, izin habis, waktu tak dapat dipercaya atau perangkat mati: arahkan ke nota; jangan sarankan memakai akun sebelumnya. Tidak ada pembuatan akun atau reset PIN offline.

Contoh: Ani menjual tunai, lalu Budi membuka akses lokal dengan PIN sendiri dan mencatat sewa/QRIS selama pemadaman 18 jam. Transaksi tetap memiliki nama pencatat masing-masing. Pergantian tidak mengubah stok, sesi meja berjalan, transaksi yang menunggu kirim, atau tenggat. Event audit pergantian mengikuti antrean yang sama, tetapi bukan penjualan. Saat koneksi kembali, gunakan sesi online petugas yang sah untuk mengirim antrean tanpa mengganti nama kasir asal.

**Status antrean.** Tampilkan jumlah dan rincian terbatas bagi kasir asal: menunggu kirim, hasil belum diketahui, tersimpan di server, atau perlu diperiksa. Saat server kembali, kirim antrean dengan ID asli; sediakan **Periksa hasil / Kirim ulang** dan **Ekspor darurat terenkripsi** jika belum terkirim. Jangan meminta kasir mengetik ulang. Pengiriman berhasil berbeda dari pembukuan final: “Tersimpan di server — tanggal QRIS perlu diperiksa” tetap mungkin. Direktur hanya melihat data yang telah sampai; tampilkan waktu pembaruan.

**Input susulan untuk nota cadangan saja.** Cocokkan terlebih dahulu terhadap antrean perangkat/ekspor dan transaksi server. Antrean dengan ID asli dipulihkan lewat sinkronisasi, bukan form ini.

1. Cari nomor nota dan, bila ada, nomor percobaan asli. Pilih **Cocokkan ke transaksi yang sudah ada** atau **Catat dari nota**. Jika belum dapat dipastikan, pilih **Catat selisih** tanpa membuat penjualan.
2. Tampilkan judul “Pembayaran sudah diterima — jangan tagih lagi”. Isi petugas asal, nomor nota, tanggal usaha asli dan zona. Untuk kafe: barang, jumlah, harga nota. Untuk sewa: meja, tarif, blok, jam/batas aktual, perpanjangan, apakah sesi selesai atau masih berjalan. Jangan mengisi jam historis dengan waktu input. Akun penginput serta waktu simpan ditetapkan server, bukan ditulis ulang oleh kasir.
   Petugas asal dipilih dari pencarian kasir, termasuk petugas yang sudah nonaktif, tanpa membuka pengaturan akun. Untuk **Cocokkan**, tampilkan tanggal, metode, nominal dan barang/meja transaksi asli; kasir mengisi alasan, bukan mengisi pembayaran baru. Jika belum ada bukti, **Catat selisih** menyimpan apa yang diketahui dan menampilkan nomor kasus, bukan struk lunas. Jika bukti ditemukan kemudian, buka nota/kasus yang sama.
3. Tunai: nominal asli, uang diterima dan kembalian sesuai bukti. QRIS: pemeriksaan bukti saat pelanggan membayar, tanggal bukti, jam/rujukan jika ada; jangan minta scan atau bayar ulang. Tunjukkan selisih harga/stok/ketersediaan terhadap catatan sekarang secara terpisah.
4. Ringkas nota sebelum **Simpan nota susulan**. Tampilkan “Tercatat”, “Sudah tercatat — lihat transaksi”, atau “Pembayaran tercatat, stok/tanggal perlu diperiksa”. Jika nomor sama dengan isi berbeda, tampilkan kedua rujukan sesuai hak dan arahkan ke pencocokan, bukan penggantian nomor untuk melewati validasi. Setelah restore, tampilkan instruksi mencocokkan meja fisik dan nota sebelum menjual meja itu kembali.

**Kembali dari nota ke aplikasi.** Sebelum menjual meja setelah gangguan, buka pencocokan fisik. Tandai meja yang belum diperiksa; cocokkan meja kosong, sesi berjalan atau pembersihan dengan sesi/nota. Tombol mulai tetap nonaktif pada meja yang belum cocok. Pemulihan satu meja tidak perlu menunggu semua nota penjualan kafe selesai diketik.

**Meja.** Kartu berisi nama meja, status teks (tersedia, dipakai, pembersihan, siap dilepas, perlu diperiksa), batas waktu dan aksi yang sah saja. Perpanjangan menampilkan tarif sesi dan batas akhir baru. Penutupan dini memperingatkan tidak ada refund; akhiri dan mulai pembersihan sesuai waktu kejadian. Jangan memulai ulang timer untuk nota sewa selesai. Jam/tabel historis meragukan: tampilkan waktu terakhir yang diketahui dan arah pemeriksaan. Hitungan mundur hanya representasi; reload, timer tab tidur, dan jam perangkat tidak dapat melepas meja.

**Inventaris.** Pisahkan “Barang masuk” (jumlah tambahan) dan “Hitung fisik” (jumlah akhir, bukan selisih). Tampilkan saldo lama, hasil baru, delta dan alasan. Jika saldo berubah selama pengisian, tampilkan saldo terbaru dan minta pemeriksaan ulang. Produk nonaktif tidak muncul di katalog jual tetapi tetap terlihat di riwayat/nota. Bedakan harga awal untuk SKU baru dari edit harga direktur.

**Direktur.** Filter tanggal yang sama untuk layar dan CSV. Tampilkan penjualan, pengembalian, hasil bersih dan pembayaran menunggu verifikasi secara terpisah. Pada kasus QRIS bertanggal saja, tindakan adalah **Verifikasi tanggal**, bukan **Setujui pembayaran**. Pada selisih stok, tunjukkan baris yang sudah/belum dibukukan, tindakan sebelumnya, dan alasan penutupan. Logout pada satu perangkat tidak menampilkan data privat sebelumnya pada layar kembali.

## Sistem visual dan komponen

Usulan: font sistem lokal, dasar 16 px, line-height 1,5; judul 22–28 px, angka uang/timer memakai digit tabular. Jarak antarelemen kelipatan 4 px, area sentuh utama minimal 44 × 44 px. Latar terang `#FFFFFF`, teks `#172B42`, aksi utama `#075E59`, perhatian `#774800`, gagal `#9D2121`; uji kontras setiap pasangan termasuk hover/fokus/disabled pada prototype. Warna selalu didampingi teks/ikon berlabel. Format IDR tanpa desimal; tanggal usaha ditulis jelas, zona terlihat pada bukti/laporan lintas hari.

Komponen bersama: bar status server, pencarian berlabel, baris item dengan jumlah/harga, kartu meja, ringkasan uang, badge status per dimensi, pesan kesalahan dekat field, panel pencocokan nota dan konfirmasi tindakan. Tidak memakai satu badge “sukses” untuk pembayaran, pemenuhan, serta pembukuan sekaligus.

## Responsif, aksesibilitas, dan status

Pada layar sempit, susun katalog dan ringkasan berurutan, sediakan tombol menuju keranjang dengan total yang terlihat, ubah grid meja menjadi kartu dan tabel transaksi menjadi daftar detail. Jangan menutup kontrol dengan keyboard virtual. Pada desktop, pertahankan urutan fokus sama dengan urutan baca. Jangan mengandalkan hover, warna, atau drag. Label permanen, heading berurutan, fokus terlihat, semua aksi lewat keyboard; dialog mengembalikan fokus, error terhubung ke field. Target WCAG 2.2 AA untuk alur penting; uji zoom 200%, keyboard dan kontras, bukan klaim kepatuhan saat ini. Jangan umumkan hitung mundur setiap detik ke pembaca layar.

| Keadaan | Tampilan dan aksi |
|---|---|
| Memuat | Beri label apa yang dimuat; jangan tampilkan saldo/omzet nol sebagai pengganti data yang belum ada. |
| Kosong | Bedakan belum ada transaksi, filter tanpa hasil, serta data tidak tersedia. Tawarkan reset filter bila relevan. |
| Gagal sebelum commit diketahui pasti | Sebut penyebab dan langkah aman; pertahankan isian yang masih terlihat. Jangan menandai lunas. |
| Hasil tidak diketahui | Tampilkan nomor percobaan dan tindakan periksa; tidak ada retry dengan nomor baru. |
| Tersimpan dengan pengecualian | Tampilkan apa yang sudah permanen dan bagian yang masih menunggu. Jangan tawarkan bayar lagi. |
| Sesi berakhir | Arahkan login ulang tanpa membocorkan detail lama. Hubungkan kembali dengan nomor nota/percobaan, bukan keranjang dari cache. |

Validasi prototype: demonstrasikan penjualan tunai, sewa dan QRIS dalam PWA saat PC mati, daftar antrean, respons hilang, sinkron ulang, tanggal bukti perlu diperiksa, kegagalan lokal yang memerlukan nota, dan konflik stok/meja. Uji layar sempit dan lebar. Simulasi bukan bukti persistensi IndexedDB nyata, keamanan, atau pemulihan operasional.