# Bug yang Sudah Diperbaiki

10 bug selesai. Tujuh dari sesi sebelumnya, tiga dari sesi ini.

**Status git:** belum di-commit — masih di working tree
- `dazo-whitelist` (branch `fix/bug-audit-konco`): 20 file, +184 −52
- `dazo-whitelist-api` (branch `staging`): 3 file, +33 −3

---

#5 — Admin baru tidak bisa login

**Penjelasan:** Admin yang dibuat dari panel `/admin/user` tidak punya record `Member` di tabel `members`. Saat login, kode melakukan `$user->member->isMember()` — karena `member` bernilai `null`, PHP error fatal. Semua admin baru tidak bisa login sama sekali.

**Perbaikan:** Ubah `$user->member->isMember()` menjadi `$user->member?->isMember()`. Tanda `?` membuat kode berhenti di `null` dan mengembalikan `false` alih-alih error.

---

#6 — Halaman permission mati total

**Penjelasan:** Di `RoleController.php:89` ada perintah `dd($item)` — fungsi PHP yang menghentikan seluruh program dan menampilkan isi variabel. Setiap kali halaman `/admin/role/permission` dibuka, halaman hanya menampilkan dump variabel, tidak pernah sampai ke daftar permission. Untuk SEMUA role, tanpa kecuali.

**Perbaikan:** Hapus baris `dd($item);`. Kode di bawahnya sudah benar.

---

#7 — Halaman member error "guard web not defined"

**Penjelasan:** `config/sanctum.php` dan `config/passport.php` merujuk ke guard bernama `'web'`, tapi guard itu tidak ada di `config/auth.php` — yang ada hanya `admin` dan `member`. Setiap request yang lewat Sanctum/Passport gagal dengan `InvalidArgumentException: Auth guard [web] is not defined`.

**Perbaikan:** Ganti `'guard' => 'web'` → `'member'` (passport.php) dan `'guard' => ['web']` → `['member', 'admin']` (sanctum.php).

---

#8 — Balasan tiket admin 500 kalau Pusher down

**Penjelasan:** Di `TicketController.php:68`, setelah pesan berhasil disimpan ke database, kode mengirim notifikasi real-time via Pusher dengan `broadcast(...)` — tanpa try-catch. Kalau Pusher/Reverb tidak terjangkau, exception dilempar, seluruh request jadi 500, padahal pesan **sudah tersimpan**. User melihat error dan mengira pesannya hilang, lalu menulis ulang.

**Perbaikan:** Bungkus dengan helper baru `safe_broadcast()` yang menangkap error dan hanya mencatat sebagai warning.

---

#11 — Notifikasi verifikasi gagal, status tidak tersimpan

**Penjelasan:** Pola identik dengan #8, tapi di `VerifyWire.php:78` dan `VerificationWire.php:65,120`. `MemberVerificationPendingNotification` mengimplementasikan `ShouldBroadcast`, jadi `notify()` ikut menyentuh Reverb. Kalau Reverb mati, exception terjadi **setelah** status terupdate — jadi status sudah ditulis, tapi request tetap 500 dan member mengira verifikasi gagal.

**Perbaikan:** Ubah `$user->notify(...)` → `safe_notify($user, ...)`.

---

#16 — Notifikasi membatalkan order langganan

**Penjelasan:** Di `OrderSubscribeWire.php:149-222`, seluruh pembuatan order dibungkus `DB::transaction()`. Di dalam transaksi itu (`:204`) dipanggil `Notification::send()`, dan `:205` `send_email()`. Karena `OrderNotif implements ShouldBroadcast` + `QUEUE_CONNECTION=sync`, notifikasi dikirim inline — masih di dalam transaksi.

Member sudah mengisi form, tekan "Lanjutkan Pembayaran", lalu dapat kata **"Failed"** karena satu `curl` ke Reverb gagal → **rollback** → order yang sah hilang total dari database.

**Bukti A/B test:**
- Reverb MATI → alert "Failed", order tidak bertambah
- Reverb HIDUP → redirect ke halaman payment, order tersimpan

**Perbaikan:** Buat dua helper di `helpers.php` — `safe_notify()` dan `safe_broadcast()` — yang menangkap error broadcast dan hanya mencatat warning, tanpa melempar exception.

**Catatan:** Alur ini masih memakai `Notification::send()` polos di `:204`, jadi **belum tuntas** untuk alur checkout member.

---

#25 — Error Meta tersamar jadi "500 JsonResponse as array"

**Penjelasan:** `AccountController::getSingleData()` punya 3 jalur keluar. Dua sudah benar (`:49` dan `:143` pakai `$raw ? ['error' => …]`), tapi `:81` lupa — selalu mengembalikan objek `JsonResponse` walau pemanggil meminta mode array. Pemanggilnya mengira selalu dapat array dan melakukan `$retreive['spend_cap']` → fatal.

Pesan error asli (misal `"Invalid Business Manager"`) hilang, diganti `"Cannot use object of type JsonResponse as array"`, dan status berubah dari 400 ke 500.

**Dampak:** `getSingleData()` dipanggil 7× di file itu — topup, withdraw, spendcap, rename, balance, sync. Semua endpoint Meta jadi tidak bisa didiagnosis.

**Perbaikan:** Ubah `:81` jadi `return $raw ? ['error' => $response['error']] : $this->error_response(400, $response['error']);`

---

#34 — Member bisa transfer saldonya ke akun iklan milik member lain

**Penjelasan (IDOR):** Di `TransferBalanceMemberWire.php:350`, akun **tujuan** dicari tanpa filter kepemilikan. Guard hanya cek "akun ini ada?", bukan "akun ini milik saya?". Karena `destinationAccount` adalah public property Livewire, nilainya bisa dikirim langsung dari browser — UI menyembunyikan opsi lain, tapi server tidak memaksa.

Query akun **sumber** di titik lain sudah memakai `where_member()`; yang bocor hanya sisi tujuan.

**Bukti eksploitasi (sebelum patch):**
```
Login member A → set destination = UUID akun member B → confirmTransfer()
→ Transfer Balance Request terkirim ke API:
   "destination_ads_account_id": "07ecfe3a-…" ← akun member lain
   "destination_identity_key": "de9143c5-…"      ← BM key ikut terbawa
```

**Perbaikan (3 titik):**

| Baris | Perubahan |
|---|---|
`:350` `confirmTransfer()` | tambah `->where_member(member_owner()->id)` |
`:197` `saveDestination()` | validasi ownership sebelum set, plus tolak self-transfer |
`:505` `buildSourceAccountsData()` | tambah `where_member()` — sumber juga bocor (temuan tambahan) |

**Verifikasi:**

| Uji | Sebelum | Sesudah |
|---|---|---|
`confirmTransfer()` dgn UUID member lain | request terkirim | `Exception: "Akun tujuan tidak valid"`, request tidak dikirim |
`saveDestination()` dgn UUID member lain | diterima | `null` (ditolak) |
`saveDestination()` akun sendiri | diterima | diterima (tidak false-positive) |

---

#45 — Tombol suspend bisa membekukan member yang salah

**Penjelasan:** Di `Member.php:121`, `scopeOwner` hanya memasang filter ID kalau `$id` terisi (`if ($id) $q->where('id', $id);`). Kalau `$id` kosong, filter dilewati dan `firstOrFail()` mengembalikan **owner pertama di database** — bukan member yang dituju. Tidak ada error, flash `'success'` tetap muncul.

**Bukti:**
```
find(null) [lama] : ADA -> 02b043e2-…  (BAHAYA — owner pertama)
find(null) [baru] : null  ✅ AMAN
```

**Perbaikan (4 metode di `MemberController.php`):**

| Baris | Metode |
|---|---|
`:236` | `suspend()` |
`:245` | `unsuspend()` |
`:220` | `send_warning()` — blacklist (temuan tambahan) |
`:57` | `show()` (temuan tambahan) |

Semua diubah dari `Member::owner($id)->firstOrFail()` → `Member::owner()->find($id)` + cek `null` eksplisit. Plus `Notification::send()` → `safe_notify()` di `:226`.

Hasil: `firstOrFail()` di file tersebut sekarang **0**.

---

#46 — Halaman Employee selalu 500

**Penjelasan:** Di `routes/_admin.php:122-128` ada 4 route terdaftar, dan permission-nya sudah di-seed di database (`employee-R/C/U/D`). Tapi `EmployeeController.php` cuma **15 baris tanpa satu pun method** — dan parent class `Admin\Controller` juga tidak punya `index/create/edit/destroy`.

Jadi `/admin/employee` → `Call to undefined method EmployeeController::index()` → **HTTP 500**.

Ini bukan "fitur belum jadi" — route dan izin sudah disiapkan, tapi method-nya tidak pernah ada.

**Perbaikan:** 4 route di-nonaktifkan (di-comment) dengan komentar penjelas. Verifikasi: 500 → **404**.

**Belum dikerjakan:** implementasi CRUD employee. Butuh keputusan lebih dulu — employee dibuat dari panel admin atau dari halaman member, dan siapa yang boleh mengelolanya.

---

## Ringkasan

| No | Bug |
|---|---|
#5 | Admin baru tidak bisa login |
#6 | Halaman permission mati total |
#7 | Halaman member error "guard web not defined" |
#8 | Balasan tiket admin 500 kalau Pusher down |
#11 | Notifikasi verifikasi gagal, status tidak tersimpan |
#16 | Notifikasi membatalkan order langganan |
#25 | Error Meta tersamar jadi "500 JsonResponse as array" |
#34 | Member bisa transfer saldonya ke akun iklan milik member lain |
#45 | Tombol suspend bisa membekukan member yang salah |
#46 | Halaman Employee selalu 500 |

## Yang belum dikerjakan

| No | Bug | Kendala |
|---|---|---|
#14 | Halaman Book Demo tidak ada | Perlu keputusan fitur |
#26 | Withdraw baca env var TOPUP | 1 karakter — belum sempat |
#27 | Payment dicocokkan hanya lewat amount | Butuh koordinasi gateway |
#29 | Fee per-user vs fee per-metode | Perlu keputusan model bisnis |
#36 | Filter "tanpa kampanye" tidak cek kampanye | Perlu cek siapa yang mengisi `active_campaign_count` |
#40 | `identity-key` tidak dibaca API | Perlu keputusan: header perlu atau tidak |
#41 | `member_bm.bm_id` tidak ada di MySQL | Migration sudah siap, belum dijalankan |
#43 | Role Technical Support tidak ada | Operasional — bisa dibuat via superadmin |

## DIBATALKAN (tidak perlu diperbaiki)

| No | Alasan dibatalkan |
|---|---|
#1–#4 | Klaim awal salah, sudah diuji ulang |
#11 | ~~Form verifikasi member tidak ada~~ — sebenarnya ada, grep salah folder |
#12 | Tes tidak adil — permission PIC memang kurang |
#13 | Duplikasi hanya lintas guard, query sudah terfilter |
#24 | `stock_bm.deleted_at` sudah ada di DB |
#33 | Ketiga kolom yang diklaim hilang ternyata ada |