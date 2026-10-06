URL => https://whitelist.dazo.dev/

Credential

1. Superadmin

- username : audit.dev@dazo.id
- password : *****

2. User

- username : audit.member@dazo.id
- password : *****

---

BUG #5 — Admin baru tidak bisa login

Penjelasan: Ketika create user pengguna untuk admin dan gak mencentang checkbox superadmin maka akan terjadi error

Url : https://whitelist.dazo.dev/admin/user/create

SS : ![Pasted image 20261006151344.png](img/Pasted%20image%2020261006151344.png)

---

BUG #6 — Halaman permission mati total

Penjelasan: Ketika buka halaman pengaturan -> peran -> permission terdapat dd()

Url : https://whitelist.dazo.dev/admin/role/permission?_i={id_role}

SS : ![BUG6 permission dd.png](img/BUG6%20permission%20dd.png)

File : app/Http/Controllers/Admin/RoleController.php:89

Catatan QA: Halaman ini butuh parameter _i = id role. Contoh: /admin/role/permission?_i=a111b8f8-e38c-4cfb-91fe-3a3d136b6b67

Bug ini HANYA muncul kalau ada satu permission dengan guard "admin" yang namanya tidak berformat "prefix-action" (misal "employee-R", "ads_account-C"). Kalau namanya "auditddtest" tanpa tanda -, atau "a-b-c" dengan dua tanda -, halaman ini LANGSUNG MATI TOTAL — isinya hanya dump: "auditddtest" // ...RoleController.php:89

Sudah diuji di server dev dengan permission uji "auditddtest" dan bugnya TERBUKTI. Permission uji masih ada di database dev — selama masih ada, halaman ini tidak bisa dipakai.

Cara bersihkan: hapus permission "auditddtest" dari tabel permissions (guard admin), lalu php artisan permission:cache-clear

---

BUG #7 — Guard "web" tidak ditemukan, halaman API error

Penjelasan: Aplikasi ini punya dua pintu masuk. Form login biasa (/login ke /member ke /admin) berjalan normal. Tapi pintu kedua, yaitu alamat yang diawali /api/, semuanya mati.

Setiap kali ada program atau fitur yang memanggil alamat /api/*, server langsung gagal dengan error "Auth guard [web] is not defined."

Penyebabnya: konfigurasi memakai nama guard "web", tapi guard "web" tidak terdaftar di sistem. Yang terdaftar hanya "member" dan "admin".

Url : https://whitelist.dazo.dev/api/user

SS : ![BUG7 api user 500.png](img/BUG7%20api%20user%20500.png)

File : config/sanctum.php:36, config/passport.php:16

Sudah diuji di server dev. Hasil: HTTP 500 dengan pesan "Auth guard [web] is not defined. (500 Internal Server Error)"

---

BUG #34 — Member bisa transfer saldonya ke akun iklan milik member lain

Penjelasan: Member bisa menulis ID akun iklan orang lain di browser, lalu sistem mentransfer saldonya ke sana. UI tidak menawarkan opsi ini, jadi tidak akan ketahuan kalau diuji lewat browser — yang diuji tampilan, bukan server-nya. Akun tujuan beserta BM key-nya ikut terkirim, sehingga sistem memproses transfer ke akun orang lain atas nama peminta.

Url : https://whitelist.dazo.dev/member/transfer_balance

SS : ![BUG34 idor destination.png](img/BUG34%20idor%20destination.png)

File : app/Http/Livewire/Member/TransferBalanceMemberWire.php

Status : SUDAH DIPERBAIKI — where_member() ditambahkan di 3 titik (confirmTransfer, saveDestination, buildSourceAccountsData)

---

BUG #45 — Tombol suspend bisa membekukan member yang salah

Penjelasan: Ketika ID member kosong, filter ID dilewati dan sistem membekukan owner PERTAMA di database — bukan member yang dituju. Tidak ada error, halaman tetap menampilkan "success". Bisa membekukan member yang lagi aktif dipakai.

Url : https://whitelist.dazo.dev/admin/member

SS : (tidak disertakan)

File : app/Http/Controllers/Admin/MemberController.php

Status : SUDAH DIPERBAIKI — 4 metode diubah: suspend, unsuspend, send_warning, show

---

BUG #46 — Halaman Employee selalu 500

Penjelasan: Route dan permission sudah ada di sistem (employee-R/C/U/D), tapi method-nya tidak pernah ditulis. Jadi begitu halaman dibuka, muncul error "Call to undefined method EmployeeController::index()". Tombolnya kelihatan siap dipakai padahal belum ada isinya.

Url : https://whitelist.dazo.dev/admin/employee

SS : (tidak disertakan)

File : routes/_admin.php