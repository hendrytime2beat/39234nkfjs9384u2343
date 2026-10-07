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

File : app/Http/Controllers/Admin/AuthController.php:141

Status : SUDAH DIPERBAIKI di lokal — belum dideploy ke server dev. Di server dev masih BELUM diperbaiki.

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

Status : SUDAH DIPERBAIKI di lokal — belum dideploy. Perbaikan hanya berlaku kalau permission yang namanya tidak berformat "prefix-action" sudah dihapus. Permission uji "auditddtest" masih ada di server dev, jadi halaman di sana masih mati total.

---

BUG #7 — Guard "web" tidak ditemukan, halaman API error

Penjelasan: Aplikasi ini punya dua pintu masuk. Form login biasa (/login ke /member ke /admin) berjalan normal. Tapi pintu kedua, yaitu alamat yang diawali /api/, semuanya mati.

Setiap kali ada program atau fitur yang memanggil alamat /api/*, server langsung gagal dengan error "Auth guard [web] is not defined."

Penyebabnya: konfigurasi memakai nama guard "web", tapi guard "web" tidak terdaftar di sistem. Yang terdaftar hanya "member" dan "admin".

Url : https://whitelist.dazo.dev/api/user

SS : ![BUG7 api user 500.png](img/BUG7%20api%20user%20500.png)

File : config/sanctum.php:36, config/passport.php:16

Sudah diuji di server dev. Hasil: HTTP 500 dengan pesan "Auth guard [web] is not defined. (500 Internal Server Error)"

Status : SUDAH DIPERBAIKI di lokal — belum dideploy. Di server dev masih BELUM diperbaiki, semua alamat /api/* masih 500.

---

BUG #46 — Halaman Employee selalu 500

Penjelasan: Route dan permission sudah ada di sistem (employee-R/C/U/D), dan method index() di controller sebenarnya ada. Yang tidak ada adalah isi $data['index'] — tempat tabel beserta kolom-kolomnya didefinisikan. Karena itu halaman langsung crash begitu dibuka.

Url : https://whitelist.dazo.dev/admin/employee

SS : ![BUG46 employee 500.png](img/BUG46%20employee%20500.png)

File : app/Http/Controllers/Admin/EmployeeController.php, app/Http/Livewire/Com/TableWire.php:75

Status : SUDAH DIPERBAIKI sementara di lokal — route dinonaktifkan, 500 jadi 404. Di server dev masih BELUM diperbaiki. Fitur CRUD employee belum dibuat.

Bukti di server dev:

GET /admin/employee -> HTTP 500
ViewException
Undefined array key "content"
(View: /home/dazo-dev-whitelist/dazo.whitelist/resources/views/layout/table.blade.php)

Dediagnosis dari EmployeeController.php:

class EmployeeController extends Controller {
    protected static $data = [
        'model' => 'Member',
        'wire'  => 'admin.member-wire',
    ];
}

Dibanding controller yang jalan (RoleController):

EmployeeController : keys = model, wire
RoleController     : keys = model, wire, index

TabelWire.php:75 menjalankan $datasend['content'] = list_name($datasend['content'], true);

Karena EmployeeController tidak punya key 'index', tidak ada data content, dan TableWire langsung crash.

Koreksi terhadap dokumen lama: error aslinya bukan "Call to undefined method EmployeeController::index()", tapi "Undefined array key content" di TableWire.php:75. Letak bug tetap sama — EmployeeController.

Catatan keamanan: route /admin/role/permission (baris 62 di routes/_admin.php) tidak memakai middleware permission sama sekali, padahal 5 route lain di group yang sama memakai. Route /admin/employee memakai middleware employee-R, jadi tidak bisa diakses tanpa permission itu.
