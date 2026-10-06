# Status Audit Bug Konco

Halaman ini berisi status **terverifikasi** per bug, hasil Checking ulang terhadap
kode dan runtime (bukan hanya `grep`). Sumber temuan awal: `bug-konco.md`.

> Diperbarui 6 Okt 2026 — audit #10 s.d. #18
> Metode: baca kode → verifikasi di browser (agent-browser) → cek DB → cek log server

## Ringkasan

| # | Judul | Status |
|---|---|---|
| 10 | Checkout terkunci: metode pembayaran tak bisa dipilih | **TERBUKTI** — tidak dapat direproduksi (sudah di-seed) |
| 11 | Member tidak punya jalur verifikasi akun | **DIBATALKAN** — klaim salah |
| 12 | PIC selalu 403 saat "Tetapkan Aset" | **DIBATALKAN** — tes tidak adil |
| 13 | 44 permission terduplikasi | **DIBATALKAN** — hanya lintas guard |
| 14 | Halaman "Book Demo/Konsultasi" tidak ada | **TERBUKTI** — fiturnya memang tidak ada |
| 15 | Promo paket hilang di checkout | **TERBUKTI** — ada bukti order nyata |
| 16 | Broadcast gagal di dalam `DB::transaction` | **TERBUKTI** — A/B test bersih |
| 17 | `unique_code` 3 digit | **TERBUKTI** — ditunda (memang untuk rekonsiliasi) |
| 18 | `identity-key` terkirim `null` | **TERBUKTI** — patch dibatalkan |
| 19 | "Ulang proses bayar" tidak ada | **TERBUKTI** — keputusan bisnis, bukan bug |
| 20 | Order pending menggantung selamanya | **TERBUKTI SEBAGIAN** — recovery ada (`expiry:order`) |
| 21 | "Report via Dashboard?" tidak ada | **TERBUKTI** — tidak ada `TYPE_*` untuk lapor |
| 22 | Renewal: tiga angka berbeda | **TERBUKTI** — file berbeda dari #15 |
| 23 | "Monitoring Durasi Langganan" tidak ada | **TERBUKTI** — tak ada halaman/indeks |
| 24 | `StockBm` SoftDeletes vs tabel `stock_bm` | **TIDAK TERBUKTI** — kolom sudah ada, model tak pakai SoftDeletes |
| 25 | `getSingleData($raw=true)` balas `JsonResponse` | **TERBUKTI** — baris `:81`, bukan `:49`/`:143` |
| 26 | `config/api.php`: withdraw baca env var **TOPUP** | **TERBUKTI** — sekarang kebetulan aman, jadi bom waktu |

**4 dari 9 dibatalkan.** Pola yang berulang: dokumen awal menyimpulkan bug dari
`grep` atau query mentah **tanpa memverifikasi apakah UI-nya benar-benar
mem Damp menunggu**. Temuan berikutnya wajib diuji di browser dulu.

---

## #10 — Checkout terkunci: metode pembayaran

**TERBUKTI, TIDAK DAPAT DIREPRODUKSI.**

Klaim akar masalahnya (`payment_methods` = 0 baris → kartu tidak muncul) sudah
tidak berlaku. Data sekarang:

```
id                                   | name             | is_active | admin_fee
b073c935-08ed-48f2-b5bd-72d7f3966659 | Transfer Bank    | t         | 0.00
d736b3be-4ac3-4772-9c24-e6671a6d614b | Virtual Account  | t         | 0.00
```

Uji browser (`/member/subscribe/package` → Buy Package → order/`{uuid}`):

| Cek | Hasil |
|---|---|
| Kartu `.payment-method-card` di DOM | 2, keduanya punya `wire:click="selectPaymentMethod('…')"` |
| Klik kartu "Transfer Bank" | OK — kartu dapat class `selected` |
| `confirmPaymentMethod` | OK — modal tertutup, selection bertahan |
| `submit` | Melewati guard payment ✅ |

Kode **masih salah** (`subscribe_form.blade.php:366` header "Bank Transfer"
statis tanpa `wire:click`; `OrderSubscribeWire.php:100` masih jadi guard), tapi
tidak terlihat lagi karena datanya ada.

Sisa masalah: dua metode punya `admin_fee = 0.00` → potensi memicu #29.

## #11 — Member tidak punya jalur verifikasi akun

**DIBATALKAN.**

Klaim: "`verification_status` hanya bisa ditulis admin; tidak ada form/upload di
sisi member". Semua salah:

| Klaim | Kenyataan |
|---|---|
| tidak ada form | `resources/views/livewire/profile/verify-wire.blade.php` (28 KB) |
| tidak ada upload | `VerifyWire.php:11` `WithFileUploads`, `:57` validasi `required|image|mimes:jpg,jpeg,png|max:2048` |
| hanya admin yang menulis | **`VerifyWire.php:75` → `'verification_status' => 'pending'`** |
| tidak ada route verifikasi | Tab di `ProfileController.php:37-42` → permission `profile.verify` |

Supporting: migration `2026_02_04_000002` + `2026_02_06_000001`, kolom
`ktp_path`/`verification_rejected_at`/`verification_rejected_reason` ada, 3
notifikasi (Pending/Approved/Rejected), 3 email view.

**Penyebab salahnya klaim:** `blade` punya 4 cabang berdasar status. Tester
memakai member yang **sudah `verified`**, jadi hanya panel hijau "Verified
Successfully" yang render — form ada di cabang `else` (`status` null).
`grep` juga salah folder: kode di `Livewire/Profile/`, bukan `Livewire/Member/`.

Uji live (status dibalik ke `null`, lalu dikembalikan):

```
wire:click terdeteksi: ["openVerifyModal"]   ← CTA "Verify Now" visible
→ modal: <input type="file" accept="image/png,image/jpeg" wire:model="ktp">
         "Upload Your ID Photo" · "JPG, JPEG, atau PNG" · Cancel / Verify
```

**Sisa temuan (sempit, bukan #11):** CTA di `verify-wire.blade.php:32` dibungkus
`@if (is_owner())`. Member non-owner yang belum verified punya gate
(`TeamController.php:38`) tapi tidak punya tombol.

## #12 — PIC selalu 403

**DIBATALKAN.**

Tes awal tidak adil — PIC pertama dibuat dengan 4 permission dashboard/transaction
yang memang tidak includ `ad_account.view_list`, jadi 403 **benar**. Diuji ulang
dengan permission sesuai → 200 OK.

**Klaim sisaannya terbukti:** grant akses hanya per-role + per-Business Manager.
Tidak ada grant per-PIC.

```
ads_accounts (PostgreSQL fbads):
  kolom (user_id, team_id, member_id) → 0

AdsAccount.php:68   scopeWhere_member → whereHas('member_bm', member_id)
AdsManagerMemberWire.php:757  ->where_member(member_owner()->id)   ← owner, bukan user login
role_helper.php:20  member_owner() → auth()->user()->member?->user_owner->member
```

Semua PIC satu tim otomatis melihat semua akun iklan owner.

**Temuan sampingan:** `ads_accounts` ada di **dua DB dengan skema berbeda** —
PostgreSQL punya `member_bm_id`, MySQL punya `member_id`. Pola yang sama seperti
#33 dan #41.

## #13 — 44 permission terduplikasi

**DIBATALKAN.**

"Duplikasi" hanya terjadi **lintas guard**, itu by-design Spatie:

```
guard_name:  admin = 97   |  member = 44
total 141 baris / 97 nama unik
distribusi: 53 nama ×1, 44 nama ×2 (contoh: ad_account.create → admin,member)

permissions_name_guard_name_unique  UNIQUE (name, guard_name)   ← constraint ADA
```

Semua query sudah memfilter guard:

```
app/Services/PermissionService.php:16   Permission::where('guard_name', $guardName)   default 'member'
                                       :117  idem
                                       :128  idem
```

UI hanya menarik `guard_name='member'` → 44 baris unik. Tidak ada duplikasi di layar.

Saran di dokumen ("tambah `UNIQUE(name)`") justru **akan merusak** — permission
admin dan member legit memakai nama sama.

## #14 — Halaman "Book Demo/Konsultasi" tidak ada

**TERBUKTI** — sesuai dokumen.

```
grep -rliE "book.?demo|konsultasi|consultation" app/ resources/ routes/ lang/  → kosong
route:list | grep -iE "demo|konsultasi|contact"                                  → kosong
information_schema | table_name like '%contact%' or '%lead%' or '%demo%'         → kosong
```

Juga sudah dicek dengan nama alternatif: `schedule|meeting|appointment|jadwal`
(hanya `AdsManagerMemberWire.php` = jadwal **iklan**, bukan meeting), `contact`
(only view lama + notifikasi + tiket).

Jadi bukan "formnya error" — **formnya tidak pernah ditulis**. Entry point PDF h.3
buntu. Yang ada hanya sistem tiket (`member/ticket`) untuk member yang sudah
terdaftar, bukan calon pelanggan. Kategori: **gap spesifikasi**.

## #15 — Promo paket hilang di checkout

**TERBUKTI — ada bukti order nyata.**

`OrderSubscribeWire.php:160` memakai `nominal`, bukan `final_price`:

```php
'subtotal' => $this->data['nominal'] + ...   // Rp10.000
'total'    => $this->total,                    // Rp10.000
```

`PaymentService.php:55` memakai `$order->total` → tagihan ikut Rp10.000.

Order uji `354aa845-b133-4405-b6d8-e4c01ce3471c`:

```
Halaman paket  : ~~Rp.10.000~~  Rp9.000
Halaman payment: Amount Rp 9.000 | Unique Code Rp 449 | Admin Fee Rp 0
                 Total Bill Rp 10.449          ← 9.000 + 449 = 9.449, bukan 10.449
DB            : order_subscribes.amount = 9000.00   (harga final)
                 orders.subtotal = 10000, orders.total = 10000.00  (harga coret)
                 order_payments.amount = 10449
```

Satu order, dua harga. Member bayar **Rp10.449** untuk paket yang diiklankan
**Rp9.000** → selisih **Rp1.000**, tanpa baris penjelas di ringkasan checkout.

Sisi admin juga tampak: "Biaya Langganan Rp 9.000" vs "Nominal Transaksi Rp 10.000".

Perbaikan: `nominal` → `final_price` di baris 160 + tampilkan baris diskon.

## #16 — Broadcast gagal di dalam `DB::transaction`

**TERBUKTI — A/B test bersih, hanya satu variabel diubah.**

### Reverb MATI
```
UI  → {"emits":[{"event":"alert","params":["error","Failed"]}]}
      URL tidak redirect, tetap di halaman checkout
DB  → orders = 1 (tidak bertambah), order_subscribes = 2 (tidak bertambah)
LOG → Proses subsrcibe gagal ["Pusher error: cURL error 7: Failed to connect
       to 127.0.0.1:8090"]
```

### Reverb HIDUP
```
UI  → redirect ke /member/transaction/payment/354aa845-…
DB  → orders = 2, order_subscribes = 3   ✅
```

Member sudah mengisi form dan menekan tombol bayar, dapat kata **"Failed"**, dan
sistem tidak mencatat apa pun.

Akar masalah `OrderSubscribeWire.php:149-222`:

```php
return DB::transaction(function () {
    Order::create($dt);                                    // :166 ✅
    OrderSubscribe::create($dt);                            // :177 ✅
    $paymentService->createPayment(...);                    // :200 ✅
    Notification::send($user, new OrderNotif(...));         // :204 ← lempar
    send_email([...]);                                      // :205 ← juga di dalam
});
// :224 catch → emit "Failed" → ROLLBACK semua di atas
```

`OrderNotif implements ShouldBroadcast` + `QUEUE_CONNECTION=sync` → broadcast
inline, masih di dalam transaksi, jadi satu kegagalan jaringan membatalkan commit.

**Catatan produksi:** Reverb **tidak ada auto-start**. Kalau restart/timeout, setiap
checkout yang sedang berjalan akan hilang. Belum teratasi, hanya tidak terpicu.

Perbaikan: pindahkan `Notification::send()` + `send_email()` keluar dari closure,
atau queue + `afterCommit()`, plus try-catch terpisah.

## #17 — `unique_code` 3 digit

**TERBUKTI — ditunda atas keputusan sendiri.**

`PaymentService.php:33` `rand(100, 999)` → tagihan naik s/d Rp999 tanpa penjelasan
di ringkasan checkout. Order Rp5.000 bisa naik ~10%.

Catatan revisi 2 Okt sudah benar: di **halaman payment** sudah dirinci
(`Amount`/`Unique Code`/`Admin Fee`/`Total Payment`). Yang tersisa hanya
ketidakcocokan ringkasan checkout vs bill akhir.

Fungsi unique code memang untuk rekonsiliasi, jadi **tidak bisa dihapus** — hanya
rentang + tampilan yang perlu dibetulkan. Rendah prioritas, aman ditunda.

Masalah lebih besar yang muncul saat #15 diuji: `unique_code` di-generate **di
dalam** transaksi, dan pengecekan hanya menolak code yang dipakai order `pending`
**hari ini**. Order `paid`/`expired` membebaskan lagi code-nya → dua order aktif
bisa dapat code sama (→ terkait #27/#28).

## #18 — `identity-key` terkirim `null`

**TERBUKTI — patch sudah dibatalkan atas permintaan.**

`ProccessSubscribeWire.php` mengirim header autentikasi dari atribut yang tidak ada:

```php
'headers' => ['identity-key' => $cekstok->stock_bm_id]   // atribut tidak ada
```

`stock_ads_accounts` tidak punya kolom `stock_bm_id` (punya relasi `stock_bm()`).

Reproduksi di browser (`admin/transaction/process?_t=subscribe&_p=…`):

```
=== RENAME & ASSIGN BM PARTNER === {"stock_ads_account_id":"3e0a8985-…",
                                    "stock_bm_id":null, …}
=== API RESPONSE === {"status_code":422,"error":"Validation failed"}
=== OPERATION FAILED ===
```

### Koreksi penting atas diagnosanya

`identity-key` yang `null` **tidak akan menghasilkan 422**. Di API,
`Facebook.php:85` → `throw new \Exception('Invalid Business Manager')` = 500.
422 datang dari validasi lain: `member_bm_id => 'required|exists:member_bm,id'`.

Tabel rujukan di MySQL API **kosong**:

```
MySQL  konco_api_db : member_bm = 0, stock_bm = 0, stock_ads_accounts = 0
PostgreSQL fbads    : member_bm = 16, stock_bm = 2
```

### Peta rantai (hasil 4 percobaan, 1 variabel tiap kali)

| Percobaan | Hasil |
|---|---|
1 | kondisi awal → `422 Validation failed` |
2 | patch `identity-key` = `stock_bm.id` | tetap `422` |
3 | tabel MySQL API diisi | → `400 Facebook token is not active` |
4 | `stock_ads_accounts.facebook_token_id` ditautkan | → `400 Business Manager ID not found` |

Jadi patch **berfungsi** (`identity_key` valid, 5 guard terlewati: validasi
`member_bm_id` → `Str::isUuid` → `StockBM::find()` → `fbToken->is_active` →
`getValidToken()`), tapi tidak membuat alur lulus. Patch dibatalkan.

### Blocker tersisa

| # | Blocker | Jenis |
|---|---|---|
1 | `member_bm.bm_id` tidak ada di MySQL `konco_api_db` (`FacebookAdAccountController.php:461`) | butuh migration |
2 | Tabel MySQL API kosong, tidak ada job sync PostgreSQL→MySQL | arsitektur/data |
3 | Token Meta asli | butuh OAuth real |

**Alur "berhasil" tidak dapat diuji lokal** — setelah `bm_id` masih ada panggilan
`graph.facebook.com/v22.0` yang butuh access token asli.

---

## Temuan sampingan (di luar daftar bug)

### Cacat UX pada `admin/transaction/process`
1. **Pesan error tidak muncul di layar.** Toast `alert` dari Livewire tidak
   terlihat setelah "Lanjutkan"; dropdown select2 justru **ter-reset** ke
   "Silakan Pilih" — seolah tidak terjadi apa-apa. Padahal backend sudah `400`.
   Dugaan: `$.confirm` menutup dialog sebelum toast dirender, atau
   `Livewire.on('alert')` tidak terpasang di halaman ini.
2. **Stok habis tanpa pesan jelas** — `ProccessSubscribeWire.php:114`
   `return 'Ads Account in use'`. `scopeAvailable()`
   (`StockAdsAccount.php:67-70`, `doesntHave('ads_account')`) mengembalikan nol
   baris tanpa menjelaskan stoknya habis.

### Prasyarat tersembunyi pada alur proses subscribe
1. `order.flag` harus `paid` — `TransactionController.php:111`. Order pending →
   `findOrFail` → **404**.
2. `member_bm_id` wajib (`required|uuid`) — kalau kosong, submit gagal diam-diam.

### #24 ternyata SUDAH teratasi di DB lokal
`konco_api_db.stock_bm` **sudah punya** kolom `deleted_at` (dicek via
`information_schema`), jadi blocker SoftDeletes yang tertulis di dokumen tidak
terjadi di lingkungan ini. Perlu dikonfirmasi di DB produksi.

---

## Perubahan sementara yang perlu dibersihkan

| Item | Lokasi |
|---|---|
Password owner lokal diganti `*****` (hash asli hilang) | user `hendrytime2beat@gmail.com` |
`API_URL` → `http://127.0.0.1:8082/api/` (dari 8001) | `dazo-whitelist/.env` |
`FRONTEND_URL` → `http://localhost:8005` (dari 8000) | `dazo-whitelist-api/.env` |
Data seed buatan: 3 baris + `facebook_token_id` | MySQL `konco_api_db` |
`ads_accounts.stock_ads_account_id` di-set `NULL` | `KC APP-Uji-001` |
`orders.flag` → `paid` | order uji `354aa845-…` |

Data seed MySQL sebaiknya dihapus — kalau tidak, pemeriksaan berikutnya bisa
menganggap tabel sudah terisi padahal di lingkungan lain kosong.

---

## Service lokal yang harus hidup

| Service | Port | Cara start |
|---|---|---|
| dazo-whitelist | 8005 | `php artisan serve --host=127.0.0.1 --port=8005` |
| dazo-whitelist-api | 8082 | `php artisan serve --host=127.0.0.1 --port=8082` |
| PostgreSQL 18 | 5432 | brew (aktif) |
| Redis | 6379 | brew (aktif) |
| **MySQL 8.4** | 3306 | **perlu `LD_LIBRARY_PATH` khusus, lihat catatan** |
| **Reverb** | 8090 | `php artisan reverb:start --host=0.0.0.0 --port=8090` — **tidak ada auto-start** |

### Catatan MySQL
`mysqld` di Homebrew rusak: tertaut ke `libabsl_die_if_null.so.2601.0.0`, yang
terpasang `.2608.0.0`. Start manual:

```bash
cd /home/linuxbrew/.linuxbrew/var/mysql   # wajib: mysqld menolak cwd yang dihapus
LD_LIBRARY_PATH=/home/linuxbrew/.linuxbrew/Cellar/abseil/20260107.1/lib \
  setsid nohup /home/linuxbrew/.linuxbrew/opt/mysql@8.4/bin/mysqld \
  --datadir=/home/linuxbrew/.linuxbrew/var/mysql \
  --port=3306 --bind-address=127.0.0.1 \
  --socket=/tmp/opencode/mysql.sock \
  --pid-file=/tmp/opencode/mysqld.pid --mysqlx=0 \
  > /tmp/opencode/mysqld.log 2>&1 < /dev/null &
```

---

## Gotcha `agent-browser` (tambahan dari yang sudah ada)

- `getAttribute('wire:model')` **null** kalau atribut punya modifier
  (`wire:model.debounce.1000ms`). Cara yang andal: iterasi semua atribut, cari yang
  `name.startsWith('wire:')`.
- `document.querySelectorAll('[wire\\:click="…"]')` sering gagal tergantung
  escaping lewat shell. Lebih andal: iterasi `element.attributes`.
- `Livewire.find(document.querySelector('[wire\\:id]').getAttribute('wire:id'))`
  lalu `.set()` / `.call()` bypass UI tapi **tidak** memicu `$.confirm` — harus
  `dispatchEvent(new Event('confirm-submit'))` dulu.
- Kolom `#expired` menolak nilai programmatically (datepicker) — tidak masalah,
  validasinya di-comment-out (`// 'expired' => 'required|date_format:Y-m-d'`).
- Ada command native: `agent-browser select`, `fill`, `click`, `type`, `snapshot`.

---

# Tambahan sesi 2 — #19 s.d. #25

> #19 – #23 dilewati atas keputusan sendiri (cluster siklus hidup transaksi).

## #19 — "Ulang proses bayar" tidak ada

**TERBUKTI.** Diuji pada order nyata `354aa845` (`flag=open`, payment `pending`,
`expired_at 08:25:42`, sudah lewat 69 menit).

`transaction-account-member-wire.blade.php:466` dan `:900` (dua kali):
```blade
@if (!$selectedTransaction->orderPayment?->isExpired() && $selectedTransaction->flag=='open')
    … "Bayar dalam MM:SS" …
    <a href="…payment.detail…">Bayar Sekarang</a>
@endif
```
Begitu `isExpired()` true, tombol hilang — **tidak ada cabang `@else`**.

Bukti:
- `/member/transaction` — menu baris hanya `info` + `file_download`
- panel detail — blok pembayaran tidak ada sama sekali
- `/member/transaction/payment/{id}` — "Waktu Pembayaran Habis", satu-satunya
  tombol "Lihat Transaksi"

Satu-satunya `retry` di sisi member adalah `PaymentDetailWire.php:298
updateRetryCountdown()` — itu **rate-limit cooldown** untuk "Check Transaction
Status", bukan bayar ulang. Di sisi admin ada `RequestWire.php:2053
retryProcess()`, tapi itu untuk admin.

**Catatan:** keputusan bisnis — lihat "Catatan user's" di bawah. Yang hilang hanya
**penanda di UI** bahwa langganan ini buntu dengan sengaja.

## #20 — Order pending menggantung selamanya

**TERBUKTI SEBAGIAN — 1 dari 2 klaim salah.**

(a) Tidak ada guard order ganda — **terbukti**:
```php
// OrderSubscribeWire.php:65-70
function rules() { return ['selectedTopUp' => 'nullable|numeric|min:100000']; }
```

(b) "menggantung selamanya" — **SALAH**, ada scheduled job:
```
Kernel.php:21                  $schedule->command('expiry:order')
OrderExpiryCheckCommand:27     Order::whereNotIn('flag',['paid','failed'])
OrderExpiryCheckCommand:82     $lockedOrder->update(['flag' => 'failed'])
```
Hasil menjalankan `php artisan expiry:order`:
```
Processing order INV-261006082042 → Order marked as failed
Orders marked as failed: 1
SEBELUM : flag=open   payment=pending
SESUDAH : flag=failed payment=expired
```
Order tidak menggantung — hanya **scheduler Laravel tidak jalan di lokal** (tidak ada
`schedule:work`/cron aktif).

(b2) Tidak ada UI admin/TS force `paid` — **terbukti**. Semua penulisan `flag`:

| File | Flag | Konteks |
|---|---|---|
`RefundProcessWire.php:109` | `paid` | refund |
`RefundProcessWire.php:89,101,185` | `declined` | refund |
`TransactionController.php:171` | `declined` | refund |
`OrderExpiryCheckCommand.php:82` | `failed` | scheduler |
`TransferBalanceMemberWire.php:454` | `paid` | transfer saldo |

Tidak ada `markAsPaid`/`forcePaid` di `Livewire/Admin/` maupun `Controllers/Admin/`.
`transaction-wire.blade.php` hanya **menampilkan** badge `flag === 'paid'`.

## #21 — "Report via Dashboard?" tidak ada

**TERBUKTI.**

- `AdminRequest.php:49-58` — semua `TYPE_*` adalah alur bisnis (subscribe, topup,
  withdraw, renew, assign…). **Tidak ada** `TYPE_REPORT`/`TYPE_PENDING`. Jadi kotak
  "Notifikasi di Dashboard Superadmin" tidak punya jalur data sama sekali.
- `transaction-account-member-wire.blade.php` — seluruh `wire:click`/`href`
  enumerated: `selectedTransaction`, unduh invoice, **bayar** (2×, hanya saat
  `!isExpired`), `closeOffcanvas`, `closeModal`, `applyDateRange`. **Nol** aksi lapor.
- Member **punya** sistem tiket (`member/ticket`, `create`, `show`) tapi harus
  navigasi manual dan tidak ada preset "kategori: transaksi macet".

Koreksi kecil: `grep "report.*transaksi"` tidak kosong — hasilnya dari
`AdsProblemReportWire.php` / `AdsProblemController.php` yang **melaporkan masalah
akun iklan** (flow AME), bukan transaksi pending. Kesimpulan dokumen benar.

## #22 — Renewal: tiga angka berbeda

**TERBUKTI** dari analisis kode (tidak diuji end-to-end).

Satu harga dipakai dari tiga sumber berbeda di satu file:

| Untuk | Sumber | Nilai |
|---|---|---|
Tampilan `checkout_multiple_wire.blade.php:100` | `nominal` | Rp10.000 |
Tagihan `CheckoutMultipleWire.php:403` | `nominal` | Rp10.000 |
Item order `CheckoutMultipleWire.php:542` | `$subscribe->price` | Rp9.000 |

```php
// blade:100
Rp{{nominal($ads['selected_subscribe']['nominal'])}}
// :403
$this->totalSubscribe += $ads['selected_subscribe']['nominal'];
// :542
'amount' => $subscribe->price,
```

Lebih buruk dari #15: **data di DB sendiri bertentangan** — `orders.total` = 10.000
vs `order_subscribes.amount` = 9.000.

⚠️ **File berbeda dari #15**, jadi memperbaiki #15 tidak menutup #22.

Tambahan: `Subscribe` punya **dua accessor untuk konsep sama** —
`getPriceAttribute()` (`:37`) dan `getFinalPriceAttribute()` (`:57`). Bom waktu.

## #23 — "Monitoring Durasi Langganan" tidak ada

**TERBUKTI.**

- `route:list | grep admin/(monitor|valid|expire|durasi|subscription)` → **kosong**
- `admin/subscribe*` = **CRUD paket** (`subscribes`), bukan monitoring member
- "Durasi Berlangganan" = 5 kemunculan, semuanya label display
  (`request/index-wire.blade.php:948,1184,2136`;
  `transaction-wire.blade.php:1073,1136`)
- `grep "not.*updated|belum.*terupdate"` di `Admin/` → nol

**`/admin/request` BUKAN monitoring.** Tab-nya `setTab('subscribe'|'topup'|
'transfer_balance'|'withdraw'|'account_error'|'business_manager')` — itu tipe
`admin_requests`. Dan `OrderSubscribeWire.php:218-219` `createFromOrder` **dikomentari**:
```php
// $adminRequestService->app(...)->createFromOrder($order);
```
→ order gagal proses **tidak pernah masuk** ke halaman Permintaan, jadi
`retryProcess` (`RequestWire.php:2053`) tidak pernah muncul.

Akar masalah yang paling mungkin — `ProcessOrderJob.php:118` **dikomentari**:
```php
//   $result = $orderService->processOrderData($this->order->id);
```
Dan `ApiDuitku.php:159` memanggil `$this->adsAccountService->processOrderData($order->id)`
— nama variabel `$order`, padahal `OrderService.php:17` menerima scalar `$orderId`.
Kalau scope itu objek model, proses diam-diam tidak jalan tanpa error.

Tidak bisa dibuktikan end-to-end: `orders` dengan `flag='paid'` di DB lokal = **0 baris**.

## #24 — `StockBm` SoftDeletes vs tabel `stock_bm`

**TIDAK TERBUKTI. Klaim salah + tidak berlaku.**

Dokumen: *"tabel `stock_bm` tidak punya `deleted_at`"*, *"seluruh Meta Graph API mati"*.

Faktanya:
```
MySQL konco_api_db.stock_bm → describe
  … updated_at timestamp NULL
    deleted_at timestamp NULL        ← ★ ADA

StockBm::count() = 1                 ← tidak error
information_schema count(deleted_at) = 1
```

Dan klaimnya **mustahil** secara logika:
```php
// dazo-whitelist-api/app/Models/StockBm.php:14
class StockBm extends Model     // ← tidak ada `use SoftDeletes`
```
Tanpa `SoftDeletes`, kode tidak akan pernah menambahkan `deleted_at` ke query.

### Koreksi besar terhadap diri sendiri

Saya sempat menulis "kolom `deleted_at` tidak ada definisinya di repo, di kedua
project". Itu **SALAH** untuk `dazo-whitelist`:
```php
// dazo-whitelist/database/migrations/2024_09_20_015521_create_stock_bms_table.php:43
$table->softDeletes(); // Soft delete support    ← SUDAH ADA
```
Saya grep literal `deleted_at` dan melewatkan helper `softDeletes()`.
**Pola yang sama seperti #11 (grep folder salah) dan #13 (lupa filter guard).**

Status per project:

| | `dazo-whitelist` | `dazo-whitelist-api` |
|---|---|---|
Migration | 127, semua **Ran** | 11 |
`create stock_bm` punya `softDeletes()` | ✅ `:43` | ❌ tidak ada migration sama sekali |
Tabel inti punya definisi migration | ✅ semua | ❌ nol |
`migrate:fresh` | ✅ aman | ❌ **hapus seluruh DB API** |

### Yang sudah dikerjakan (edit, bukan file baru)

`dazo-whitelist-api/database/migrations/2026_01_29_082431_add_disabled_at_to_stock_ads_accounts_table.php`
— `up()` ditambah 3 blok ber-`hasTable`/`hasColumn`:
- `stock_bm.deleted_at`
- `stock_ads_accounts.deleted_at`
- `member_bm.bm_id` ← ini yang dibaca `FacebookAdAccountController.php:461`,
  penyebab `400 Business Manager ID not found` di #18

`down()` sengaja tidak drop kolom baru. Semua no-op di DB sekarang, idempoten.

Belum dijalankan — `php artisan migrate` **belum dieksekusi**.

### Yang benar-benar hidup dari #24

Hanya **gagasan arsitekturnya** — `Facebook.php:83-86` memang single point of failure:
```php
$secret = request()->header('identity-key');
if (!$secret || !Str::isUuid($secret)) throw new \Exception('Invalid Business Manager');
$business = StockBM::find($secret);
if (!$business) throw new \Exception('Invalid Business Manager');
```
Ini terbukti di #18: tabel kosong → `400`, token kosong → `400`, `bm_id` NULL → `400`.

## #25 — `getSingleData($raw=true)` balas `JsonResponse`

**TERBUKTI. Lokasi di dokumen salah — yang benar baris `:81`.**

`getSingleData()` punya 3 jalur keluar; hanya 1 lupa menjaga `$raw`:

| Jalur | Baris | Hormati `$raw`? |
|---|---|---|
Validasi gagal | `:49` | ✅ `return $raw ? ['error' => …] : error_response(…)` |
**Panggil Meta gagal** | **`:81`** | ❌ **`return $this->error_response(400, …)`** |
Exception | `:143` | ✅ `return $raw ? ['error' => …] : error_response(…)` |

### Reproduksi (4 tahap, 1 variabel tiap kali)

 Lewat jalur RSA + Passport asli dari `dazo-whitelist`:

| Percobaan | Keluaran |
|---|---|
`account_id = 'act_uji_001'` (non-numeric) | `TIPE: array` · `{"_status_code":400,"error":"The account id field must be a number."}`` → `:49` **benar** |
`account_id = '123456'` (numeric, diproses Meta) | `{"_status_code":500,"error":"Cannot use object of type Illuminate\\Http\\JsonResponse as array"}` → `:81` **rusak** |

**Pesan asli hilang total.** Seharusnya muncul `"Invalid Business Manager"`
(dari `Facebook.php:86`), yang muncul justru fatal PHP. Kode status juga berubah
**400 → 500**.

### Uji browser (member topup) — sisi member normal

`/member/topup` → pilih akun → nominal Rp100.000 → Transfer Bank → `processPayment`:
```
order d243771a-c2c4-4d77-acc7-ac39244544a9 dibuat
Harga Rp100.000 · Kode Unik 142 · Biaya Admin 0 · Total Pembayaran Rp100.142
```
Pemanggilan `account/topup` ke Meta terjadi saat **proses order**, bukan saat
checkout — jadi #25 tidak bisa dipicukan dari halaman member.

### Dampak

`getSingleData($request, true)` dipanggil **7×** di `AccountController.php`:
`:365, :413, :465, :537, :608, :711, :813` — topup, withdraw, spendcap, rename,
balance, sync.

Satu baris membuat **seluruh endpoint Meta tidak bisa didiagnosis**: setiap error
asli berubah jadi pesan yang sama dan status jadi 500. Tidak ada guard
`is_array($retreive)` di seluruh file (`grep` = 0).

Ini yang membuat #18 sulit didiagnosis — `"Invalid Business Manager"` tidak pernah
terlihat, hanya fatal PHP.

### Perbaikan
```php
// :81
if ($response['error'] ?? null) {
    return $raw ? ['error' => $response['error']] : $this->error_response(400, $response['error']);
}
```
+ guard `if (! is_array($retreive)) return $retreive;` di 7 pemanggil.

## Temuan sampingan baru

### #26 terverifikasi tanpa sengaja

Ditemukan saat membaca `config/api.php` untuk #25:

```php
// config/api.php:19-20
'account_topup'    => env('API_ACCOUNT_TOPUP', 'account/topup'),
'account_withdraw' => env('API_ACCOUNT_TOPUP', 'account/withdraw'),   // ← env var SALAH
```

**TERBUKTI.** `.env` hanya punya 5 `API_*` (`API_URL`, `API_KEY`, `API_SECRET`,
`API_PRIVATE`, `API_PUBLIC`) — `API_ACCOUNT_TOPUP` **tidak diset**, jadi default
terpakai dan kebetulan benar:

```
account_topup    => account/topup
account_withdraw => account/withdraw    ← benar secara tidak sengaja
account_refund   => account/refund
```

**Bahaya:** begitu siapa pun menyet `API_ACCOUNT_TOPUP` (deploy produksi, atau
"merapikan" konfigurasi), withdraw ikut mengikuti:

```
'account_topup'    => 'account/topup'
'account_withdraw' => 'account/topup'     ← dari env YANG SAMA
```

Dua endpoint itu **bukan no-op**:
```
routes/api.php:56   'topup'    → AccountController@topupSpendCap()    :580
routes/api.php:57   'withdraw' → AccountController@withdrawSpendCap() :683
```

`OrderService.php:442` mengirim `'amount' => $withdraw->amount` → nominal withdraw
dipakai sebagai **penambahan** spend cap. Jadi saldo limit di Meta **naik** saat
member menarik — salah arah, tanpa exception, tanpa log.

Perbaikan: `env('API_ACCOUNT_WITHDRAW', 'account/withdraw')`. Satu karakter, nol
risiko (kalau env kosong hasilnya identik dengan sekarang).

Sudah dicek: `config/api.php` punya **21** pemanggilan `env('API_*')` dan ini
**satu-satunya duplikat**.

⚠️ Ironi: bug yang sekarang tidak merusak justru karena `.env` belum lengkap.
Merapikan `.env` justru **mengaktifkan** bug-nya.

### Prasyarat alur proses subscribe admin
1. `order.flag` harus `paid` — `TransactionController.php:111`, kalau tidak → **404**
2. `stock_ads_accounts` harus ada yang `available()` — `StockAdsAccount.php:67-70`
   (`doesntHave('ads_account')`); kalau tidak → `'Ads Account in use'` tanpa
   penjelasan stok habis
3. `member_bm_id` wajib (`required|uuid`) — kalau kosong submit gagal **diam-diam**

### Cacat UX di `admin/transaction/process`
- **Pesan error tidak muncul di layar** setelah "Lanjutkan"; select2 malah
  ter-reset ke "Silakan Pilih" — seolah tidak terjadi apa-apa, padahal backend sudah
  `400`. Dugaan: `$.confirm` menutup dialog sebelum toast dirender, atau
  `Livewire.on('alert')` tidak terpasang di halaman ini.
- Gridiklan tidak tampil di `/member/ads_manager` & `/member/topup` kalau
  `ads_accounts.stock_ads_account_id` NULL — `AdsAccount::scopeActive`
  mensyaratkan `whereNotNull('stock_ads_account_id')` + `stock_ads_account.is_active`.

### `Subscribe` punya 2 accessor untuk konsep sama
`getPriceAttribute()` (`:37`) dan `getFinalPriceAttribute()` (`:57`) — keduanya
harga setelah promo. Pilih satu.

---

## Keputusan yang perlu kamu konfirmasi

| # | Pertanyaan |
|---|---|
#19 | Kalau memang by-design, PDF h.4 perlu diupdate atau kotak "Ulang proses bayar" diisi guideline + tombol "Buat Tiket" |
#20 | Konfirmasi `schedule:work`/cron jalan di produksi — kalau tidak, order menggantung memang |
#24 | Konfirmasi `SHOW COLUMNS FROM stock_bm LIKE 'deleted_at'` di produksi |
#24 | Migration yang sudah diedit — jalankan atau tunggu konfirmasi `member_bm.bm_id` produksi? |

---

## Yang belum diuji

Sisa **#26 – #46**, plus #5 – #9 yang seksi detailnya hilang (lihat catatan
"anchor `end` menimpa seluruh blok" di `bug-konco.md`). Untuk #5 – #9 hanya baris
ringkasan di tabel yang tersisa:

| # | Ringkas | Lokasi |
|---|---|---|
5 | Admin baru tidak bisa login — 500 LIVE | `AuthController.php:141` |
6 | `dd()` tertinggal — halaman permission mati TOTAL | `RoleController.php:89` |
7 | Guard `web` tidak ada — API mati total | `config/auth.php` vs `sanctum.php:36` + `passport.php:16` |
8 | Balasan tiket admin 500 (Pusher) | `TicketController.php:68` |
9 | Export PDF dashboard → 404 | `DashboardWire.php:246` + `AdminController.php:35` |

#1 – #4 sudah di-audit ulang dan **dicoret** (lihat bagian bawah `bug-konco.md`).