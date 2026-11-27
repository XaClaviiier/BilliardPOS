# CerdikPOS — Stack dan kontrak internal

Status: rancangan hasil validasi 1.3, revisi offline-first, backend TypeScript, autentikasi dan VPN, menunggu persetujuan Phase 1. Aturan bisnis dan AC: [requirement.md](requirement.md). Model simpan: [database.md](database.md); kontrol akses: [security.md](security.md).

> Revisi frontend: Tailwind CSS, shadcn/ui, Zustand, TanStack Query dan library pendukung menggantikan batasan CSS biasa pada rancangan awal. Kontrak API dan mekanisme nota kertas dipertahankan.

> Perubahan jaringan: VPN-only untuk semua peran; seluruh dokumen diselaraskan. Backend produksi menggunakan TypeScript + Fastify dengan Better Auth terintegrasi dalam satu proses Node.js.

## Keputusan teknis yang diusulkan

| Bagian | Pilihan dan alasan |
|---|---|
| Prototype | `prototype/`: HTML, CSS, JS vanilla dan data simulasi, terpisah dari produksi. Tidak memerlukan database atau framework. |
| Frontend produksi | `web/`: TypeScript + Vite untuk build, React untuk reuse formulir kasir, nota dan tampilan meja. Tailwind CSS + shadcn/ui untuk antarmuka, Zustand untuk state UI lintas komponen dan TanStack Query untuk server state. Tanpa framework SSR. Library belum diinstal. |
| PWA | Offline-first kasir: Workbox untuk shell, IndexedDB melalui Dexie untuk snapshot, transaksi dan outbox. Nota fisik cadangan terakhir. |
| Server | `server/`: TypeScript + Node.js LTS + Fastify untuk REST API, Better Auth dan penyajian build frontend melalui `@fastify/static`. Pisahkan handler dari fungsi transaksi domain; satu aplikasi modular, tanpa service autentikasi atau sinkronisasi terpisah. |
| Data | PostgreSQL dalam container pada PC lokal; driver `pg` (node-postgres), query SQL berparameter dan migrasi SQL berurutan. Lock dan constraint menjaga stok/meja/nota pada satu commit. Tidak memerlukan broker, ORM atau cache terdistribusi. |
| Jaringan | Domain stabil dan HTTPS tepercaya melalui NetBird untuk seluruh pengguna (kasir, direktur, dan perangkat lain). Akses VPN-only tanpa bypass LAN atau internet publik; autentikasi aplikasi terpisah. Docker Desktop + Docker Compose wajib untuk deployment lokal; Caddy wajib sebagai reverse proxy dan terminasi TLS. Tidak menggunakan VPS atau Ubuntu Server sebagai OS host. DB tidak diekspos ke kasir/VPN. |

Development: pisahkan database pengembangan, uji, dan operasional; konfigurasi lokal di luar repo, tanpa data pelanggan. Kunci versi toolchain/dependency saat pemilihan disetujui, tidak menginstal pada fase ini. Usulan direktori lain: `server/migrations/`, `web/src/`, `shared/`, `tests/e2e/`; buat hanya pada fasenya. Penamaan API dan SQL `snake_case`; status stabil berbahasa Inggris, label UI bahasa Indonesia. Nilai rupiah integer (JSON dalam rentang integer aman; `bigint` PostgreSQL dibaca sebagai string atau BigInt di server, konversi ke Number hanya setelah pemeriksaan batas aman; jangan serialisasi BigInt langsung), jumlah barang/blok positif, ID UUID sebagai string, tanggal `YYYY-MM-DD`, timestamp RFC 3339 dengan offset. Tidak menyamakan tanggal tanpa jam dengan tengah malam UTC.

## Rincian stack dan library

Daftar berikut memperjelas pilihan teknis untuk Phase 1. Library tambahan bersifat usulan, dipilih sesuai kebutuhan nyata dan tidak otomatis diinstal. Arsitektur tetap satu PC lokal, satu PostgreSQL dan satu origin HTTPS; Fastify menyajikan frontend statis, REST API dan autentikasi Better Auth dalam satu proses.

### Frontend (`web/`)

| Teknologi / library | Fungsi | Status |
|---|---|---|
| TypeScript + React | UI kasir dan direktur, komponen reusable | Utama |
| Vite | Development server dan build statis | Utama |
| Node.js LTS + pnpm | Toolchain development/build; Node.js LTS juga runtime backend produksi | Utama / pnpm usulan |
| Tailwind CSS + `@tailwindcss/vite` | Styling responsif dan integrasi Vite; design tokens melalui CSS custom properties | Utama |
| shadcn/ui | Komponen UI yang dimiliki proyek, disesuaikan dengan design system | Utama |
| Radix UI | Primitive aksesibel untuk komponen shadcn/ui yang menggunakan Radix | Utama sesuai komponen |
| `clsx`, `tailwind-merge`, `class-variance-authority` | Komposisi class dan varian komponen | Pendukung |
| Zustand (`zustand`) | State UI lintas komponen, keranjang/draft dalam memori; bukan database transaksi | Utama |
| TanStack Query (`@tanstack/react-query`) | Server state, query invalidation, polling terkontrol dan status request | Utama |
| React Router (`react-router`) | Navigasi SPA; server menyajikan fallback SPA di luar `/api` | Utama |
| Fetch API | HTTP client same-origin, cookie sesi, CSRF dan timeout eksplisit | Utama |
| React Hook Form (`react-hook-form`) | Formulir transaksi dan administrasi | Utama |
| Zod + `@hookform/resolvers` | Schema input bersama; server memvalidasi ulang setiap request | Utama |
| Lucide React (`lucide-react`) | Ikon lokal | Usulan |
| TanStack Table (`@tanstack/react-table`) | Tabel laporan dan inventaris | Usulan |
| Recharts | Grafik dashboard direktur | Usulan |
| Sonner | Notifikasi; status transaksi tidak hanya ditampilkan melalui toast | Usulan |
| React state dan context | State lokal komponen; identitas sesi berasal dari backend, bukan sumber otorisasi | Utama |

Gunakan TanStack Query untuk data server dan Zustand untuk state UI yang benar-benar lintas komponen. Hindari menduplikasi cache query ke Zustand. Jangan persist sesi, laporan atau transaksi ke localStorage. Dexie menjadi satu-satunya antrean persisten; jangan memakai antrean mutasi TanStack Query atau Workbox kedua. Status lokal dan commit server dibedakan. Komponen shadcn/ui dipilih sesuai kebutuhan, bukan memasang seluruh katalog. Tailwind digunakan pada frontend produksi; prototype vanilla tetap terpisah.

### PWA dan penanganan gangguan

| Teknologi / library | Fungsi |
|---|---|
| Web App Manifest | Instalasi aplikasi pada perangkat |
| Service Worker API | Cache aset statis dan halaman petunjuk gangguan |
| `vite-plugin-pwa` + Workbox | Integrasi manifest dan service worker dengan Vite; gunakan konfigurasi cache eksplisit |
| Cache Storage API | Aset aplikasi saja, bukan transaksi, laporan, respons sesi atau data pelanggan |
| `AbortController` | Membatasi waktu tunggu request; timeout tidak membuktikan transaksi gagal |
| `crypto.randomUUID()` | Identitas request yang dipakai kembali untuk replay identik sesuai kontrak |

| IndexedDB + `dexie` | Database lokal kasir, transaksi atomik snapshot/proyeksi/outbox |
| `dexie-react-hooks` | Query reaktif lokal melalui `useLiveQuery` |
| StorageManager (`navigator.storage.persist`, `estimate`) | Minta persistensi dan pantau quota; bukan jaminan data tidak hilang |

Sync engine TypeScript sederhana berjalan saat aplikasi dibuka, kembali foreground, event online dan polling/backoff terbatas saat aktif. Periksa API nyata, bukan hanya `navigator.onLine`. Tidak mengandalkan Background Sync ketika PWA ditutup. Gunakan Web Locks jika tersedia ditambah transaksi IndexedDB dan lease lokal untuk satu pengirim; server tetap idempoten. Retry hanya payload immutable dengan ID sama pada gangguan transport/5xx; 401 menunggu login, 403/409/422 masuk pemeriksaan. Setelah ack atomik, refresh snapshot tanpa menimpa proyeksi pending. Workbox hanya cache aset publik, API network-only. Uji dukungan browser, quota, update SW dan migrasi Dexie tanpa menghapus antrean.

### Backend (`server/`)

| Teknologi / library | Fungsi | Status |
|---|---|---|
| TypeScript + Node.js LTS | Bahasa dan runtime backend; build dengan `tsc` | Utama |
| Fastify | REST API, hooks, request ID, error handler dan batas ukuran body | Utama |
| `@fastify/static` | Sajikan build Vite; fallback SPA hanya di luar `/api` | Utama |
| Better Auth + plugin username (`better-auth/plugins`) | Login username/password, scrypt dan sesi PostgreSQL dalam proses yang sama | Utama |
| Zod | Schema request ketat pada server dan schema bersama di `shared/`; aturan domain tetap di service | Utama |
| `pg` (node-postgres) | Pool PostgreSQL untuk SQL bisnis dan adapter Better Auth | Utama |
| SQL berparameter | Query dan transaksi atomik tanpa ORM | Utama |
| `node-pg-migrate` | Runner tunggal untuk migrasi SQL bernomor, termasuk SQL autentikasi yang ditinjau | Usulan |
| Pino (logger Fastify) | Structured logging, redaksi rahasia | Utama |
| `@fastify/rate-limit` | Pembatasan endpoint aplikasi sensitif; Better Auth membatasi login | Utama |
| `@fastify/helmet` | Security headers dan CSP yang kompatibel dengan PWA | Utama |
| `node:crypto` | UUID, kredensial perangkat acak, SHA-256 digest, CSRF dan perbandingan aman | Standard library |
| `csv-stringify` | Ekspor CSV dengan quoting; lindungi sel teks dari formula sebelum serialisasi | Usulan |

Better Auth dibuat sebagai instance server-only di `server/`, menggunakan adapter PostgreSQL, scrypt bawaan, plugin username, dan `session.cookieCache` nonaktif. Pertahankan kontrak `/api/v1/session`: handler Fastify memanggil API server Better Auth (`signInUsername`, `getSession`, `signOut`) secara langsung dalam proses yang sama, meneruskan header yang diperlukan melalui `fromNodeHeaders` dan mengembalikan seluruh `Set-Cookie` yang relevan. Jangan mengekspos route Better Auth mentah ke browser. Terapkan rate limiting pada wrapper login sekalipun pemanggilan API server tidak melewati limiter internal library. Validasi origin login, status akun dan kredensial perangkat sebelum cookie sesi diterbitkan; binding `device_id` wajib ditetapkan server saat sesi dibuat, bukan oleh klien. Pada setiap request terlindungi, verifikasi sesi lewat Better Auth, cocokkan perangkat dan status akun terbaru, lalu periksa izin operasi. Jika binding/pemeriksaan gagal, jangan menerbitkan cookie atau izinkan operasi; cabut sesi yang terlanjur dibuat. Gunakan CSRF token terikat sesi untuk mutasi aplikasi. Provisioning/reset hanya melalui direktur atau bootstrap lokal memakai API resmi library; uji jalur username dan reset tanpa email, termasuk pencabutan sesi, sebelum operasional. Tidak ada signup publik, MFA, SSO atau framework RBAC tambahan.

Gunakan Zod strict untuk payload, query dan parameter; schema bersama tidak menggantikan validasi domain dan constraint DB. Fastify hooks untuk autentikasi/izin, request ID, rate limit dan error handling; Pino meredaksi cookie, authorization, password dan token. Konfigurasi proxy hanya untuk Caddy tepercaya; same-origin tanpa CORS lintas origin. Caddy mengelola TLS, Helmet mengelola header aplikasi; hindari kebijakan CSP ganda yang bertentangan.

Referensi: https://better-auth.com/docs/integrations/fastify, https://better-auth.com/docs/adapters/postgresql, https://better-auth.com/docs/plugins/username, https://zod.dev/.

### Database, deployment dan backup

| Teknologi | Fungsi |
|---|---|
| PostgreSQL | Container database; sumber kebenaran transaksi pada PC lokal |
| Constraint, unique index, row lock dan SQL transaction | Integritas stok, meja, nota, idempotency dan audit |
| `pg_dump` / `pg_restore` | Backup dan pemulihan yang diuji; simpan salinan terpisah dari PC operasional |
| NetBird | Jalur privat wajib seluruh pengguna dan host server; bukan pengganti autentikasi aplikasi |
| HTTPS | Transport HTTPS tepercaya melalui jaringan privat NetBird |
| Caddy | Reverse proxy wajib, terminasi TLS dan akses same-origin |
| Docker Desktop | Runtime container pada PC operasional lokal |
| Docker Compose v2 | Menjalankan dan mengelola seluruh layanan melalui `docker compose up -d` |
| Multi-stage Dockerfile | Build Vite dan TypeScript; satu image runtime Node.js LTS dengan dependency produksi dan aset statis |
| Docker named volumes | Persistensi PostgreSQL dan data/konfigurasi runtime Caddy |
| Docker bridge networks | Isolasi jaringan antarservice; database hanya di jaringan internal |

Deployment menggunakan Docker Desktop pada PC operasional lokal (tanpa VPS dan tanpa Ubuntu Server sebagai OS host). OS host spesifik belum ditetapkan; Linux container dapat berjalan melalui backend virtualisasi Docker Desktop. Caddy wajib sebagai satu-satunya entry point aplikasi.

### Topologi Docker Compose

```text
Kasir / Direktur / perangkat lain (semua peer NetBird)
              |
       Domain privat -> IP NetBird host
              |
          HTTPS :443
              |
       caddy (container)
              |
       app (Fastify + Better Auth + build React/Vite)
              |
       db (PostgreSQL)
              |
       named volume PostgreSQL
```

| Service | Tanggung jawab |
|---|---|
| `caddy` | Reverse proxy ke `app`, TLS, konfigurasi Caddyfile; hanya TCP 443 dipublikasikan dengan pembatasan akses NetBird pada host. |
| `app` | Node.js + Fastify menyajikan REST API, Better Auth dan build statis frontend; tidak mempublikasikan port langsung ke host. |
| `db` | PostgreSQL dengan named volume dan healthcheck; tanpa publikasi port 5432. |
| `migrate` (opsional) | Job sekali jalan dari image yang sesuai untuk migrasi SQL sebelum aplikasi menerima trafik. |

NetBird wajib pada host Docker Desktop dan semua perangkat pengguna, bukan pada tiap container internal. Usulan: buat grup server, kasir dan direktur; hapus kebijakan allow-all dan izinkan grup pengguna menuju server hanya TCP 443. Batasi ingress Caddy pada IP/interface NetBird jika didukung Docker Desktop dan gunakan firewall host untuk menolak akses dari LAN biasa dan jaringan publik. Uji dari peer berizin, peer tanpa izin, LAN tanpa VPN, dan internet publik. Jangan mengandalkan IP internal Docker atau header yang dapat dipalsukan untuk otorisasi.

Gunakan domain milik pengguna (contoh `pos.example.id`) dengan NetBird Custom DNS Zone yang memetakan hostname ke IP NetBird host; distribusikan zona hanya ke grup terkait. Domain dan provider DNS aktual belum ditentukan. Jangan membuat port forwarding router atau public ingress. Referensi: https://docs.netbird.io/manage/dns/custom-zones dan https://caddyserver.com/docs/automatic-https.

### Operasional deployment

- Simpan `compose.yaml`, `Dockerfile`, `Caddyfile` dan `.env.example` di repo; rahasia aktual di luar Git. Gunakan image dengan versi terkunci setelah pengujian.
- Gunakan `docker compose up -d --build` untuk build dan menjalankan stack; `docker compose ps` dan `docker compose logs` untuk verifikasi. Healthcheck dan kesiapan migrasi harus mengendalikan startup, bukan sekadar urutan container.
- Gunakan restart policy `unless-stopped` untuk layanan jangka panjang. Docker Desktop harus aktif; konfigurasi startup host dan uji reboot nyata. Restart policy tidak menjamin layanan tersedia sebelum Docker Desktop berjalan.
- Pertahankan volume PostgreSQL serta volume `/data` dan `/config` Caddy; jangan menjalankan `docker compose down -v` pada lingkungan operasional. Backup PostgreSQL dengan `pg_dump` dan uji `pg_restore`; simpan salinan di luar PC dan di luar volume database.
- Rekomendasi HTTPS: sertifikat publik melalui ACME DNS-01 pada Caddy dengan modul provider DNS yang sesuai; tidak memerlukan port publik 80/443. Simpan token DNS dengan izin minimum di luar Git. Alternatif: Caddy internal CA dengan root CA terpasang pada semua perangkat. Metode final belum disetujui. Uji resolusi domain, trust browser/PWA, renewal sertifikat, dan kehilangan ISP (termasuk restart peer); jangan menjanjikan ketersediaan VPN saat control plane atau relay tidak terjangkau.
- Database tidak diekspos ke LAN/VPN. Gunakan jaringan Compose terpisah untuk ingress dan database; hanya `app` yang memerlukan akses database saat runtime; pisahkan role runtime dan migrasi/backup. Uji akses privat, pemulihan, restart dan kehilangan ISP sebelum operasional.

### Testing dan tooling

| Teknologi / library | Fungsi |
|---|---|
| ESLint + Prettier + `tsc --noEmit` | Lint, formatting dan pemeriksaan tipe frontend dan backend |
| Vitest | Unit test frontend/backend, domain dan mock adapter |
| React Testing Library | Pengujian komponen/formulir |
| MSW (opsional) | Mock HTTP mengikuti kontrak API, termasuk respons hilang setelah commit |
| Playwright | E2E, kontrol akses dan simulasi gangguan jaringan |
| Fastify `inject()` + Vitest | Integration test handler dan kontrol akses tanpa membuka port |
| `pnpm audit` + test integrasi auth | Pemeriksaan dependency TypeScript dan alur sesi/provisioning/reset. |
| PostgreSQL uji nyata | Integrasi SQL, concurrency, constraint, rollback dan restore |
| Testcontainers for Node.js (opsional) | PostgreSQL sementara untuk integration test menggunakan Docker Desktop pada lingkungan pengujian terpisah |
| GitHub Actions (opsional) | CI untuk lint, typecheck, test dan build jika disetujui |

### Batas arsitektur dan dependency

- Tidak menggunakan Next.js, ORM, atau Serwist. Backend Fastify dan Better Auth berjalan dalam satu proses; frontend Vite + React.
- Tailwind CSS, shadcn/ui, Zustand dan TanStack Query merupakan pilihan frontend produksi yang diperbarui. Gunakan library pendukung sesuai kebutuhan dan hindari state ganda.
- Tidak menggunakan Redis, Kafka, RabbitMQ, Kubernetes, microservices atau payment gateway. QRIS tetap berupa pencatatan bukti sesuai kontrak, bukan integrasi gateway.
- Pertahankan versi dependency yang kompatibel dan lockfile (`pnpm-lock.yaml`) setelah disetujui. Versi spesifik belum dipilih atau diuji.
- Tidak menggunakan VPS atau Ubuntu Server sebagai OS host deployment. PostgreSQL tetap lokal dalam Docker Desktop. IndexedDB hanya di perangkat kasir; PostgreSQL tetap sumber kebenaran pusat. Tidak ada broker atau database terdistribusi.

## Kontrak API untuk mock dan backend

Base `/api/v1`. JSON kecuali CSV. Sesi dalam cookie; GET tanpa efek. Respons sukses resource `{data: ...}`; daftar `{data: [...], next_cursor: string|null}`. Mutasi baru `201`, perubahan atau replay identik `200`. Error `{error:{code, message, fields?}}`. `401` sesi habis; `403` peran/perangkat/jalur ditolak; `404` tidak ditemukan; `409` konflik bisnis/versi/duplikasi isi; `422` alasan/input tidak valid. Gagal jaringan atau timeout berarti **hasil belum diketahui**, bukan `409` dan bukan bukti tidak tersimpan. Mock frontend harus bisa mensimulasikan respons hilang setelah commit.

| Operasi | Permintaan inti | Hasil dan batas |
|---|---|---|
| `GET /health`, `GET /session`, `POST /session`, `DELETE /session` | Health tanpa data privat; login `{username,password}` lewat NetBird dan perangkat yang telah didaftarkan; Fastify memanggil Better Auth langsung | Keterjangkauan layanan, identitas/role/perangkat aktif atau error; logout hanya sesi saat ini. Health tidak membuktikan transaksi tersimpan. |
| `GET/POST /products`, `PATCH /products/{id}`, `GET/POST /categories`, `PATCH /categories/{id}` | Daftar dengan `q`, `active`, `cursor`; buat `{sku,name,category_id,price}`; edit field yang diizinkan, alasan untuk perubahan status | Produk/kategori; SKU bentrok `409`; saran nama mirip bukan penolakan. Perubahan harga lama atau status hanya direktur; tidak ada DELETE produk. |
| `GET /products/{id}/movements`, `POST /stock-movements` | `{request_id,product_id,kind,quantity,reason,expected_version?}`; `receipt`: jumlah masuk; `count`: jumlah fisik dan versi wajib | Saldo, delta, versi. `409 stale_stock` membawa saldo/versi terkini; jangan kirim ulang hasil hitung lama otomatis. |
| `GET/POST /tables`, `PATCH /tables/{id}`, `GET /rentals/{id}` | Metadata meja, status, sesi; query sesi menurut ID | Termasuk waktu server, tarif sesi, batas akhir, waktu siap dilepas. Tidak memberikan hak ubah tarif kepada kasir. |
| `POST /sales` | `{request_id,kind,details,payment}`. `cafe`: `details={items:[{product_id,qty,expected_price}],rental_id?}`; `rental_start`: `{table_id,blocks,expected_rate}`; `rental_extend`: `{rental_id,blocks}`. `payment={method:'cash',cash_received}` atau `{method:'qris',amount,proof_checked:true,proof_date,proof_at?,proof_ref?}`. | `{sale_id,request_id,total,change_due,fulfillment,booking_state,rental_id?,case_ids}`. Tolak tunai kurang atau stok tidak tersedia sebelum transaksi; jika QRIS sudah dibayar tetapi pesanan tidak dapat dipenuhi, simpan pembayaran asli sekali dan buka kasus. Jangan mengganti harga yang dikonfirmasi dengan harga baru. |
| `POST /rentals/{id}/close`, `POST /rentals/{id}/release` | `{request_id}` | Status sesi dan `clean_until`; tolak lepas terlalu awal atau dari sesi yang salah. |
| `GET /transactions/by-request/{request_id}`, `GET /paper-notes/{note_no}` | ID percobaan atau nomor nota yang di-URL-encode | Hasil commit/tautan yang sesuai, atau `404`. Jika percobaan pertama masih berlangsung atau DB baru direstore, `404` sendiri tidak mengizinkan penjualan kedua. |
| `POST /paper-notes` | Semua aksi: `{request_id,note_no,action,original_request_id?}`. `link`: tambah `{existing_sale_id,reason}` setelah nilai, tanggal, metode dan layanan cocok; ambil tanggal, zona dan petugas asal dari penjualan. `record`: tambah `{occurred_date,occurred_at?,business_zone,original_cashier_id,kind,details,payment}` memakai harga, tarif, jumlah, dan keadaan nyata dari nota, bukan katalog hari input. `discrepancy`: tambah `{occurred_date,business_zone,reason,details}` untuk bukti yang ada meski petugas/penjualan belum diketahui. | `{note_no,action,sale_id,total,change_due,fulfillment,booking_state,case_ids}`. Field penjualan bernilai null untuk selisih tanpa penjualan. Nota identik `200`; isi berbeda pada nota tercatat `409 note_conflict`. `link` tidak memicu efek jual. `record`/`link` untuk menyelesaikan nota selisih memerlukan `case_id` dan bukti yang dicocokkan, bukan nomor nota baru. |
| `GET /cases`, `POST /cases`, `POST /cases/{id}/actions`, `POST /cases/{id}/resolve`, `POST /refunds` | Kasus biasa `{request_id,type,reason,sale_id?,original_request_id?}`; selisih dengan nota melalui `POST /paper-notes`. Tindakan kasir `{request_id,kind,reason,details}`. Penyelesaian direktur `{request_id,decision,reason,verified_date?,movement_id?}`. Refund kasir `{request_id,sale_id,amount,returned_date,reason}`. | Kasir mencatat penanganan/pengganti beserta item dan jumlah aktual tanpa pembayaran baru; hanya direktur memverifikasi tanggal atau menutup kasus yang memerlukan pemeriksaan. Daftar operasional kasir bukan laporan. Setiap efek keuangan/stok dan audit dicatat dalam transaksi yang sama. |
| `GET /reports/sales`, `GET /reports/sales.csv`, `GET /reports/sales/{id}`, `GET /audit` | Direktur; `from`, `to` inklusif menurut tanggal usaha, `kind?`, `method?`, `cursor?` untuk detail | Total penjualan final, refund periode, hasil bersih, pending terpisah, waktu pembaruan, baris detail. CSV memakai filter dan rumus sama; refund tidak dipindahkan ke tanggal penjualan. |
| `GET/PATCH /settings`, `GET/POST /rates`, `GET/POST /users`, `PATCH /users/{id}`, `POST /users/{id}/reset-password`, `GET /devices`, `POST /devices/register`, `POST /devices/{id}/activate-cashier`, `POST /devices/{id}/revoke` | Direktur; pengaturan zona/durasi, versi tarif/blok, akun/peran/status, registrasi/penggantian perangkat beralasan | Penggantian **perangkat** kasir mencabut perangkat lama secara atomik. Ganti petugas pada perangkat yang sama tidak mencabut perangkat. Reset direktur terakhir melalui prosedur lokal, bukan endpoint anonim. |
| `GET /paper-notes/cashiers/lookup` | Kasir aktif melalui NetBird mencari nama/kode petugas asal dengan `q`; termasuk akun kasir nonaktif. | Hanya `id,name`, tanpa hak baca/manajemen akun. Asal tidak terbukti: simpan bukti sebagai kasus. |
| `POST /devices/enroll` | Perangkat tujuan menukar undangan sekali pakai dari `POST /devices/register`, melalui NetBird dan jalur pendaftaran tepercaya. | Cookie perangkat untuk browser tujuan, bukan sesi pengguna. Belum boleh menjual sampai login melalui NetBird dan perangkat diaktifkan. |

### Kontrak offline-first (tambahan)

Semua endpoint di bawah berada pada `/api/v1`, melalui NetBird dan sesi Better Auth sah. Mutasi memakai CSRF token. Jalur penjualan/rental UI selalu masuk Dexie lebih dahulu; endpoint bisnis lama tetap untuk kompatibilitas mock/internal dan tidak boleh menjadi jalur yang melewati outbox/izin.

| Endpoint | Aksi dan syarat | Hasil dan batas |
|---|---|---|
| `PUT /devices/{id}/offline-cashiers`, `POST /offline/prepare-cashier` | Online saja. Direktur menentukan daftar petugas untuk perangkat. Masing-masing petugas login sendiri pada perangkat tersebut; persiapan menggunakan aktor dan perangkat dari sesi, bukan ID pilihan klien. | Server mencatat persiapan individual dan mengembalikan ID persiapan untuk mengikat verifier PIN lokal. PIN maupun verifier tidak dikirim ke API. Perangkat siap untuk petugas itu hanya setelah verifier tersimpan. Perubahan daftar tidak memperluas izin yang sudah terbit. |
| `POST /offline/open` | Kasir berwenang membuka satu periode per perangkat dengan snapshot konsisten. | `{grant_id,device_id,issued_at,expires_at,snapshot_id,server_time,cashiers,...}`. `cashiers` hanya petugas yang diizinkan dan telah disiapkan (`user_id,preparation_id,name`). `expires_at` paling lambat 7 hari sejak penerbitan server. Pergantian petugas lokal bukan request API dan tidak menerbitkan grant baru. |
| `GET /offline/status` | Baca status grant, epoch server, recovery gate dan receipt yang berhak dilihat. | Tidak memperpanjang izin. |
| `POST /sync/operations` | FIFO perangkat: `{request_id,grant_id,snapshot_id,sequence,original_cashier_id,kind,occurred_at,business_date,payload}`. Jenis penjualan/rental tetap; `cashier_switch` merekam pergantian tanpa pembayaran atau efek stok/meja. Pengirim online memakai sesi sahnya sendiri. | Server mencocokkan pencatat dengan daftar petugas grant, perangkat, waktu dan urutan pergantian; jangan memakai akun pengirim sebagai pencatat asal. Asal/pencabutan meragukan masuk pemeriksaan. Receipt mengakui commit dengan ID asli. |
| `POST /offline/close` | Kasir online menutup grant setelah semua operasi dicocokkan dan sequence akhir dikonfirmasi perangkat. | Membuka perubahan master terkait; pergantian shift dan kedaluwarsa tidak menutup periode secara otomatis. |

Sync mengembalikan receipt `{request_id,status,result_ref,server_time,rental_id?,case_ids?}` hanya setelah commit. Replay identik menghasilkan receipt sama; ID/sequence berbeda isi `409`. Konflik tercatat sebagai kasus melalui transaksi server bila pembayaran sudah terjadi, status `needs_review`; tidak menganggap kasus sebagai pembayaran final. Grant kedaluwarsa tidak otomatis membatalkan transaksi yang sudah dicatat dalam periode: login online ulang, verifikasi bukti/waktu dan status grant, lalu kirim; bila meragukan tahan untuk review. Restore wajib recovery gate sebelum replay; 404 bukan bukti belum commit. Snapshot baru hanya setelah antrean aman; tidak menimpa transaksi lokal. Laporan server menunjukkan periode offline masih terbuka dan waktu data terakhir.


Konvensi yang digunakan mock dan server:

- `record` kafe: `details.items[]` memuat `sku_snapshot,name_snapshot,qty,unit_price,product_id?`. `record` sewa memuat `table_id,blocks,rate_snapshot,block_minutes,opened_at?,paid_until?,closed_at?,released_at?,rental_id?,physical_state`; jam hanya diisi dari bukti. Tanpa waktu tepercaya, jangan mengalokasikan meja; catat kasus. Penjualan tunai langsung maupun susulan memakai `payment={method:'cash',cash_received}`; tunai kurang ditolak. QRIS mencatat `payment.amount` yang telah diperiksa bersama field bukti. Untuk bukti bertanggal saja, `proof_at` kosong; tanggal pembukuan sementara mengikuti `proof_date`, bukan hari input. Jangan membuat penjualan final untuk nominal yang belum cocok.
- Pada `PATCH /tables/{id}`, kasir hanya mengubah metadata, bukan asosiasi tarif atau status `needs_check` untuk melewati pemeriksaan. Setelah gangguan, kasir menyatakan kondisi meja fisik dari layar pencocokan; `POST /tables/{id}/reconcile` menerima `{request_id,physical_state,rental_id?,reason}`. Alokasi ditolak selama meja belum cocok. Kondisi terpakai yang tidak cocok tetap diblokir sampai sesi asli/nota atau kasus diselesaikan.
- Pada `POST /session`, validasi origin login; setelah login, `GET /session` menyertakan `csrf_token` untuk header `X-CSRF-Token` pada mutasi bersesi. Token bukan kredensial perangkat. Hak login tidak mencakup penerbitan undangan perangkat.

### Contoh yang mengikat kedua sisi

Nota `A-0007`, kopi 2 × Rp10.000, QRIS bukti bertanggal 18 Mei tanpa jam; diinput 19 Mei. Mock mengembalikan `total=20000`, `change_due=null`, `booking_state=pending_date`, `fulfillment=fulfilled`, tanpa `proof_at`. Laporan 18 Mei: pending Rp20.000, final Rp0. Setelah verifikasi tanggal 18 Mei: pending Rp0, final Rp20.000. Mengirim payload identik kembali mengembalikan transaksi yang sama, tidak mengurangi stok ulang. Bila SKU tidak cocok, stok masuk pemeriksaan per baris; status pembayaran tidak kembali menjadi belum dibayar.

## Cara pembuktian per fase

Fase 2: prototype tanpa backend, termasuk jalur nota kertas dengan instruksi dan input susulan simulasi. Fase 3: uji Dexie dan mock offline (restart, quota, dua tab); mock adapter mengikuti endpoint/payload/status di atas; uji formulir, akses dan status UI dengan data uji. Fase 4: uji integrasi Better Auth (provisioning, sesi, perangkat, revocation, kegagalan auth) dan Vitest dan Fastify inject untuk aturan domain dan handler serta PostgreSQL uji nyata untuk constraint, rollback, dua request bersamaan, pembukuan, dan restore; tanpa frontend. Fase 5: integrasi sync FIFO, receipt dan recovery gate; ganti adapter mock dengan API privat, uji kontrak dan perjalanan kritis. Fase 6: runner browser (usulan Playwright) dan uji lokasi terpisah mengikuti AC di requirement. Pemilihan tool bukan izin instalasi, operasi Git atau deployment.
