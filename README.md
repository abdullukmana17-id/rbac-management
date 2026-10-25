# Dokumentasi REST API — CodeIgniter 4 + Shield

Dokumentasi endpoint, request, dan response untuk API manajemen user, autentikasi token, HMAC, ban, force reset, grup, dan permission.

## Daftar Isi

1. [Informasi Umum](#1-informasi-umum)
2. [Autentikasi](#2-autentikasi)
3. [Format Response & Kode Status](#3-format-response--kode-status)
4. [Auth](#4-auth)
5. [Access Token](#5-access-token)
6. [HMAC Key](#6-hmac-key)
7. [Users (Admin)](#7-users-admin)
8. [Ban, Aktivasi & Force Reset (Admin)](#8-ban-aktivasi--force-reset-admin)
9. [Grup & Permission (Admin)](#9-grup--permission-admin)
10. [Permission yang Dibutuhkan](#10-permission-yang-dibutuhkan)
11. [Catatan Penting](#11-catatan-penting)

---

## 1. Informasi Umum

| Item | Nilai |
|---|---|
| Base URL | `http://localhost:8080/api` |
| Format | JSON |
| Header wajib | `Content-Type: application/json`, `Accept: application/json` |
| Autentikasi | `Authorization: Bearer <token>` |
| Zona waktu | Mengikuti konfigurasi server |

### Ringkasan Endpoint

| Method | Path | Akses | Deskripsi |
|---|---|---|---|
| POST | `/auth/register` | Publik | Registrasi user baru |
| POST | `/auth/login` | Publik | Login, mendapatkan access token |
| GET | `/auth/me` | Token | Profil user saat ini |
| POST | `/auth/logout` | Token | Cabut token yang sedang dipakai |
| POST | `/auth/change-password` | Token | Ganti password |
| GET | `/tokens` | Token | Daftar access token milik sendiri |
| POST | `/tokens` | Token | Buat access token |
| DELETE | `/tokens` | Token | Cabut semua access token |
| PATCH | `/tokens/{id}/expiration` | Token | Ubah/hapus kedaluwarsa token |
| DELETE | `/tokens/{id}` | Token | Cabut satu access token |
| GET | `/hmac-keys` | Token | Daftar HMAC key |
| POST | `/hmac-keys` | Token | Buat HMAC key |
| DELETE | `/hmac-keys` | Token | Cabut semua HMAC key |
| DELETE | `/hmac-keys/{key}` | Token | Cabut satu HMAC key |
| GET | `/users` | `users.read` | Daftar user (paginasi) |
| POST | `/users` | `users.create` | Buat user |
| GET | `/users/{id}` | `users.read` | Detail user |
| PUT/PATCH | `/users/{id}` | `users.edit` | Ubah user |
| DELETE | `/users/{id}` | `users.delete` | Hapus user |
| POST | `/users/{id}/ban` | `users.ban` | Ban user |
| DELETE | `/users/{id}/ban` | `users.ban` | Unban user |
| POST | `/users/{id}/activate` | `users.edit` | Aktifkan user |
| POST | `/users/{id}/deactivate` | `users.edit` | Nonaktifkan user |
| POST | `/users/{id}/force-reset` | `users.edit` | Paksa reset password |
| DELETE | `/users/{id}/force-reset` | `users.edit` | Batalkan paksa reset |
| POST | `/users/force-reset` | `users.edit` | Paksa reset beberapa user |
| POST | `/users/force-reset-all` | grup `superadmin` | Paksa reset semua user |
| GET | `/users/{id}/groups` | `users.read` | Daftar grup user |
| POST | `/users/{id}/groups` | `users.manage-admins` | Tambah grup |
| PUT | `/users/{id}/groups` | `users.manage-admins` | Sinkronisasi grup |
| DELETE | `/users/{id}/groups/{group}` | `users.manage-admins` | Hapus grup |
| GET | `/users/{id}/permissions` | `users.read` | Permission level user |
| PUT | `/users/{id}/permissions` | `users.manage-admins` | Sinkronisasi permission |
| POST | `/users/{id}/permissions/check` | `users.read` | Cek apakah user punya permission |

---

## 2. Autentikasi

### 2.1 Bearer Token (utama)

Dapatkan token lewat `POST /auth/login`, lalu sertakan di setiap request:

```http
Authorization: Bearer 5f4c2a9e7b...
```

Token mentah hanya ditampilkan **sekali** saat dibuat. Server hanya menyimpan hash (SHA-256).

### 2.2 HMAC-SHA256 (opsional)

Bila Anda memasang filter `hmac` pada rute tertentu, request ditandatangani dengan shared secret:

```http
Authorization: HMAC-SHA256 <key>:<hmac-sha256-dari-body-request>
```

Contoh pembuatan signature (PHP):

```php
$signature = hash_hmac('sha256', $requestBody, $secretKey);
header("Authorization: HMAC-SHA256 {$key}:{$signature}");
```

Key dan secret key dibuat lewat [`POST /hmac-keys`](#62-buat-hmac-key).

### 2.3 Force Password Reset

Jika akun ditandai wajib reset password, semua endpoint **kecuali** `/auth/me`, `/auth/logout`, dan `/auth/change-password` akan menolak request:

```http
HTTP/1.1 403 Forbidden
```
```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

---

## 3. Format Response & Kode Status

### 3.1 Response Sukses

Bentuk response sukses bervariasi per endpoint (lihat tiap endpoint). Pola umum:

```json
{ "data": { } }
```

Response daftar dengan paginasi:

```json
{
  "data": [ ],
  "meta":  { "page": 1, "perPage": 10, "total": 1, "totalPages": 1 },
  "links": { "self": "...", "first": "...", "last": "...", "next": null, "previous": null }
}
```

### 3.2 Response Error

Semua error dari controller memakai format `fail*()` bawaan CodeIgniter:

```json
{
  "status": 404,
  "code": 404,
  "messages": {
    "error": "User 99 tidak ditemukan."
  }
}
```

Error validasi (`400`) berisi pesan per field:

```json
{
  "status": 400,
  "code": 400,
  "messages": {
    "email": "The email field must contain a valid email address.",
    "password": "The password field must be at least 8 characters in length."
  }
}
```

### 3.3 Kode Status HTTP

| Kode | Arti | Dipakai saat |
|---|---|---|
| 200 | OK | Request berhasil / resource dihapus |
| 201 | Created | Resource berhasil dibuat |
| 204 | No Content | Berhasil, tanpa body (logout, cabut semua token) |
| 400 | Bad Request | Validasi gagal |
| 401 | Unauthorized | Token tidak ada/tidak valid, atau kredensial salah |
| 403 | Forbidden | Akun diban/belum aktif, tidak punya izin, atau wajib reset password |
| 404 | Not Found | Resource tidak ditemukan |
| 429 | Too Many Requests | Terkena rate limit (`auth-rates`) |

---

## 4. Auth

### 4.1 Registrasi

`POST /auth/register` — Publik (dibatasi rate limit)

Aktif hanya jika `Auth::$allowRegistration = true`; jika tidak, mengembalikan `403`.

**Body**

| Field | Tipe | Wajib | Aturan |
|---|---|---|---|
| `username` | string | Tidak | 3–30 karakter, unik |
| `email` | string | Ya | Email valid, unik |
| `password` | string | Ya | Minimal 8 karakter |
| `device_name` | string | Tidak | Nama token, default `default` |

```json
{
  "username": "budi",
  "email": "budi@example.com",
  "password": "rahasia123",
  "device_name": "web"
}
```

**Response `201 Created`**

```json
{
  "user": {
    "id": 2,
    "username": "budi",
    "email": "budi@example.com",
    "active": true,
    "banned": false,
    "requires_password_reset": false,
    "groups": ["user"],
    "created_at": "2026-10-25 06:00:00"
  },
  "token": "5f4c2a9e7b1d..."
}
```

**Error:** `400` validasi, `403` registrasi dinonaktifkan.

---

### 4.2 Login

`POST /auth/login` — Publik (dibatasi rate limit)

**Body**

| Field | Tipe | Wajib |
|---|---|---|
| `email` | string | Ya |
| `password` | string | Ya |
| `device_name` | string | Tidak (default `api-login`) |

```json
{ "email": "admin@example.com", "password": "secret123", "device_name": "web" }
```

**Response `200 OK`**

```json
{
  "user": {
    "id": 1,
    "username": "admin",
    "email": "admin@example.com",
    "active": true,
    "banned": false,
    "requires_password_reset": false,
    "groups": ["admin"],
    "created_at": "2026-10-25 05:14:23"
  },
  "token": "5f4c2a9e7b1d...",
  "requires_password_reset": false
}
```

Jika `requires_password_reset` bernilai `true`, arahkan user ke halaman ganti password.

**Error**

| Kode | Penyebab |
|---|---|
| 400 | Field kosong / email tidak valid |
| 401 | Email atau password salah |
| 403 | `Akun dibanned: <alasan>` atau `Akun belum diaktifkan.` |
| 429 | Terlalu banyak percobaan |

---

### 4.3 Profil Saya

`GET /auth/me` — Token

**Response `200 OK`**

```json
{
  "data": {
    "id": 1,
    "username": "admin",
    "email": "admin@example.com",
    "active": true,
    "banned": false,
    "requires_password_reset": false,
    "groups": ["admin"],
    "created_at": "2026-10-25 05:14:23",
    "ban_message": null,
    "user_permissions": ["beta.access"]
  }
}
```

`user_permissions` hanya berisi permission yang diberikan langsung ke user (bukan dari grup).

---

### 4.4 Logout

`POST /auth/logout` — Token

Mencabut token yang sedang dipakai pada request ini.

**Response `204 No Content`** — tanpa body.

---

### 4.5 Ganti Password

`POST /auth/change-password` — Token (tetap bisa diakses saat wajib reset)

**Body**

| Field | Tipe | Wajib | Aturan |
|---|---|---|---|
| `current_password` | string | Ya | Harus cocok dengan password saat ini |
| `new_password` | string | Ya | 8–255 karakter, berbeda dari password lama |
| `device_name` | string | Tidak | Nama token baru |

```json
{ "current_password": "secret123", "new_password": "passwordBaru456" }
```

**Response `200 OK`**

```json
{
  "message": "Password berhasil diubah.",
  "token": "a91c0d77e2..."
}
```

Semua token lama dicabut. Simpan `token` yang baru. Flag *force reset* ikut dihapus.

**Error:** `400` validasi, `401` `Password saat ini salah.`

---

## 5. Access Token

Semua endpoint di bagian ini membutuhkan `Authorization: Bearer <token>` dan hanya mengelola token **milik user yang sedang login**.

### Objek Token

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | integer | ID token |
| `name` | string | Nama token |
| `scopes` | array | Daftar scope, default `["*"]` |
| `key` | null | Selalu `null` untuk access token |
| `last_used_at` | string\|null | Terakhir dipakai |
| `expires_at` | string\|null | Tanggal kedaluwarsa (`null` = tidak ada) |
| `created_at` | string | Dibuat pada |

### 5.1 Daftar Token

`GET /tokens`

**Response `200 OK`**

```json
{
  "data": [
    {
      "id": 3,
      "name": "Work Laptop",
      "scopes": ["*"],
      "key": null,
      "last_used_at": "2026-10-25 06:10:00",
      "expires_at": null,
      "created_at": "2026-10-25 05:20:00"
    }
  ]
}
```

### 5.2 Buat Token

`POST /tokens`

| Field | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `name` | string | Ya | Maks. 255 karakter |
| `scopes` | array | Tidak | Default `["*"]` |
| `expires_at` | string\|null | Tidak | Contoh `2026-12-31 23:59:59` atau `2026-12-31T23:59` |

```json
{
  "name": "Mobile App",
  "scopes": ["posts.manage", "forums.manage"],
  "expires_at": "2026-12-31 23:59:59"
}
```

**Response `201 Created`**

```json
{
  "data": {
    "id": 4,
    "name": "Mobile App",
    "scopes": ["posts.manage", "forums.manage"],
    "key": null,
    "last_used_at": null,
    "expires_at": "2026-12-31 23:59:59",
    "created_at": "2026-10-25 06:15:00"
  },
  "token": "c3b1e5f0a2..."
}
```

> `token` hanya muncul sekali. Scope tidak bisa diubah setelah token dibuat.

### 5.3 Ubah Kedaluwarsa

`PATCH /tokens/{id}/expiration`

```json
{ "expires_at": "2027-01-15 00:00:00" }
```

Kirim `{ "expires_at": null }` untuk menghapus kedaluwarsa.

**Response `200 OK`** — `{ "data": { ...objek token terbaru... } }`

**Error:** `404` `Token tidak ditemukan.`

### 5.4 Cabut Satu Token

`DELETE /tokens/{id}`

**Response `200 OK`**

```json
{ "id": 4 }
```

**Error:** `404` `Token tidak ditemukan.`

### 5.5 Cabut Semua Token

`DELETE /tokens`

**Response `204 No Content`**

> Token yang sedang dipakai ikut dicabut, sehingga request berikutnya akan `401`.

---

## 6. HMAC Key

Struktur sama dengan access token, tetapi menghasilkan sepasang **key** (publik) dan **secret key** (rahasia, dipakai untuk menandatangani request). `key` selalu ditampilkan di daftar, sedangkan `secret_key` hanya sekali saat dibuat.

> Wajib mengisi `authtoken.hmacEncryptionKeys` di `.env` sebelum memakai HMAC.

### 6.1 Daftar HMAC Key

`GET /hmac-keys`

```json
{
  "data": [
    {
      "id": 7,
      "name": "Partner API",
      "scopes": ["*"],
      "key": "a6c460151b4cabbe1c1d73e08915ce8e",
      "last_used_at": null,
      "expires_at": null,
      "created_at": "2026-10-25 06:20:00"
    }
  ]
}
```

### 6.2 Buat HMAC Key

`POST /hmac-keys` — Body sama dengan [5.2](#52-buat-token).

**Response `201 Created`**

```json
{
  "data": {
    "id": 7,
    "name": "Partner API",
    "scopes": ["*"],
    "key": "a6c460151b4cabbe1c1d73e08915ce8e",
    "last_used_at": null,
    "expires_at": null,
    "created_at": "2026-10-25 06:20:00"
  },
  "key": "a6c460151b4cabbe1c1d73e08915ce8e",
  "secret_key": "56c85232f0e5b55c05015476cd132c8d"
}
```

### 6.3 Cabut Satu HMAC Key

`DELETE /hmac-keys/{key}` — `{key}` adalah nilai field `key`, bukan `id`.

**Response `200 OK`**

```json
{ "key": "a6c460151b4cabbe1c1d73e08915ce8e" }
```

**Error:** `404` `HMAC key tidak ditemukan.`

### 6.4 Cabut Semua HMAC Key

`DELETE /hmac-keys` → **`204 No Content`**

---

## 7. Users (Admin)

Membutuhkan grup `admin` atau `superadmin` **dan** permission pada tiap endpoint (lihat [bagian 10](#10-permission-yang-dibutuhkan)).

### Objek User

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | integer | ID user |
| `username` | string\|null | Username |
| `email` | string | Email |
| `active` | boolean | Status aktivasi |
| `banned` | boolean | Status ban |
| `requires_password_reset` | boolean | Wajib ganti password |
| `groups` | array | Grup user |
| `created_at` | string | Dibuat pada |
| `ban_message` | string\|null | *Hanya di detail* — alasan ban |
| `user_permissions` | array | *Hanya di detail* — permission langsung |

### 7.1 Daftar User

`GET /users` — `users.read`

**Query**

| Parameter | Default | Keterangan |
|---|---|---|
| `page` | 1 | Nomor halaman |
| `per_page` | 20 | 1–100 |
| `q` | — | Cari berdasarkan username (LIKE) |

**Response `200 OK`**

```json
{
  "data": [
    {
      "id": 1,
      "username": "abdullukmana",
      "email": "abdullukmana17@gmail.com",
      "active": true,
      "banned": false,
      "requires_password_reset": false,
      "groups": ["admin"],
      "created_at": "2026-10-25 05:14:23"
    }
  ],
  "meta": { "page": 1, "perPage": 10, "total": 1, "totalPages": 1 },
  "links": {
    "self": "http://localhost:8080/api/users?page=1&per_page=10&q=",
    "first": "http://localhost:8080/api/users?page=1&per_page=10&q=",
    "last": "http://localhost:8080/api/users?page=1&per_page=10&q=",
    "next": null,
    "previous": null
  }
}
```

`data` selalu berupa **array**. Pada halaman pertama `previous` bernilai `null`; pada halaman terakhir `next` bernilai `null`.

### 7.2 Detail User

`GET /users/{id}` — `users.read`

```json
{
  "data": {
    "id": 5,
    "username": "budi",
    "email": "budi@example.com",
    "active": true,
    "banned": true,
    "requires_password_reset": false,
    "groups": ["user"],
    "created_at": "2026-10-25 06:00:00",
    "ban_message": "Melanggar ToS",
    "user_permissions": []
  }
}
```

**Error:** `404` `User 5 tidak ditemukan.`

### 7.3 Buat User

`POST /users` — `users.create`

| Field | Tipe | Wajib | Aturan |
|---|---|---|---|
| `username` | string | Tidak | 3–30 karakter, unik |
| `email` | string | Ya | Email valid, unik |
| `password` | string | Ya | Min. 8 karakter |
| `force_password_reset` | boolean | Tidak | Jika `true`, user wajib ganti password saat login |

```json
{
  "username": "siti",
  "email": "siti@example.com",
  "password": "sementara123",
  "force_password_reset": true
}
```

**Response `201 Created`** — `{ "data": { ...objek user detail... } }`. User otomatis masuk grup default (`user`).

### 7.4 Ubah User

`PUT /users/{id}` atau `PATCH /users/{id}` — `users.edit`

Semua field opsional; hanya field yang dikirim yang diubah.

| Field | Aturan |
|---|---|
| `username` | 3–30 karakter, unik (mengabaikan user ini) |
| `email` | Email valid |
| `password` | Min. 8 karakter |

```json
{ "username": "siti_baru", "email": "siti.baru@example.com" }
```

**Response `200 OK`** — `{ "data": { ...objek user detail... } }`

**Error:** `400` validasi, `404` tidak ditemukan.

### 7.5 Hapus User

`DELETE /users/{id}` — `users.delete`

| Query | Keterangan |
|---|---|
| `hard=1` | Hapus permanen. Tanpa ini, memakai *soft delete*. |

**Response `200 OK`**

```json
{ "id": 5 }
```

**Error:** `403` `Tidak bisa menghapus akun sendiri.`, `404` tidak ditemukan.

---

## 8. Ban, Aktivasi & Force Reset (Admin)

Semua endpoint di bagian ini mengembalikan `{ "data": { ...objek user... } }` saat sukses (`200 OK`), dan `404` bila user tidak ditemukan.

### 8.1 Ban User

`POST /users/{id}/ban` — `users.ban`

```json
{ "message": "Melanggar ketentuan layanan" }
```

`message` opsional. Semua access token user otomatis dicabut.

**Response `200 OK`** — `data.banned = true`, `data.ban_message = "Melanggar ketentuan layanan"`.

**Error:** `403` `Tidak bisa memblokir akun sendiri.`

### 8.2 Unban User

`DELETE /users/{id}/ban` — `users.ban`

**Response `200 OK`** — `data.banned = false`, `data.ban_message = null`.

### 8.3 Aktifkan / Nonaktifkan

| Endpoint | Efek |
|---|---|
| `POST /users/{id}/activate` | `active = true` |
| `POST /users/{id}/deactivate` | `active = false` dan semua token dicabut |

Permission: `users.edit`.

### 8.4 Paksa Reset Password (Satu User)

| Endpoint | Efek |
|---|---|
| `POST /users/{id}/force-reset` | `requires_password_reset = true` |
| `DELETE /users/{id}/force-reset` | `requires_password_reset = false` |

Permission: `users.edit`.

### 8.5 Paksa Reset Beberapa User

`POST /users/force-reset` — `users.edit`

```json
{ "user_ids": [1, 2, 3, 4] }
```

**Response `200 OK`**

```json
{
  "message": "Force reset diterapkan.",
  "user_ids": [1, 2, 3, 4]
}
```

**Error:** `400` bila `user_ids` kosong atau bukan array.

### 8.6 Paksa Reset Semua User

`POST /users/force-reset-all` — hanya grup `superadmin`

Tanpa body.

**Response `200 OK`**

```json
{ "message": "Force reset diterapkan ke semua user." }
```

---

## 9. Grup & Permission (Admin)

### 9.1 Daftar Grup User

`GET /users/{id}/groups` — `users.read`

```json
{ "data": ["admin", "beta"] }
```

### 9.2 Tambah Grup

`POST /users/{id}/groups` — `users.manage-admins`

```json
{ "groups": ["beta"] }
```

**Response `201 Created`**

```json
{ "data": ["user", "beta"] }
```

### 9.3 Sinkronisasi Grup

`PUT /users/{id}/groups` — `users.manage-admins`

Mengganti **seluruh** grup user dengan daftar yang dikirim.

```json
{ "groups": ["admin", "beta"] }
```

**Response `200 OK`**

```json
{ "data": ["admin", "beta"] }
```

### 9.4 Hapus Grup

`DELETE /users/{id}/groups/{group}` — `users.manage-admins`

**Response `200 OK`** — `{ "data": [ ...grup tersisa... ] }`

### Error Umum Grup

| Kode | Penyebab |
|---|---|
| 400 | `groups` bukan array, atau grup tidak terdaftar di `AuthGroups::$groups` |
| 403 | Non-superadmin mencoba menambah/mengubah/menghapus grup `superadmin` |
| 404 | User tidak ditemukan |

### 9.5 Permission Langsung User

`GET /users/{id}/permissions` — `users.read`

```json
{ "data": ["users.create", "beta.access"] }
```

Hanya permission level user, **tidak** termasuk permission dari grup.

### 9.6 Sinkronisasi Permission

`PUT /users/{id}/permissions` — `users.manage-admins`

```json
{ "permissions": ["admin.access", "beta.access"] }
```

**Response `200 OK`**

```json
{ "data": ["admin.access", "beta.access"] }
```

**Error:** `400` bila `permissions` bukan array atau ada permission yang tidak terdaftar di `AuthGroups::$permissions`.

### 9.7 Cek Permission

`POST /users/{id}/permissions/check` — `users.read`

```json
{ "permission": "users.delete" }
```

**Response `200 OK`**

```json
{ "permission": "users.delete", "allowed": true }
```

Memeriksa permission langsung dan permission dari seluruh grup user, termasuk wildcard (mis. `users.*`).

---

## 10. Permission yang Dibutuhkan

Konfigurasi default pada `AuthGroups::$matrix`:

| Permission | Deskripsi | `superadmin` | `admin` |
|---|---|:---:|:---:|
| `users.read` | Lihat daftar & detail user | ✔ | ✔ |
| `users.create` | Buat user | ✔ | ✔ |
| `users.edit` | Ubah user, aktivasi, force reset | ✔ | ✔ |
| `users.delete` | Hapus user | ✔ | ✔ |
| `users.ban` | Ban / unban | ✔ | ✔ |
| `users.manage-admins` | Kelola grup & permission | ✔ | ✘ |

`superadmin` memperoleh semuanya melalui wildcard `users.*`. Endpoint `force-reset-all` mensyaratkan **grup** `superadmin`, bukan sekadar permission.

---

## 11. Catatan Penting

1. **Token mentah hanya tampil sekali** (`token`, `secret_key`). Simpan segera.
2. **Rate limiting:** rute `/auth/*` memakai filter `auth-rates` dan dapat merespons `429`.
3. **Respons filter Shield:** filter `tokens` mengembalikan `401` bila token hilang atau tidak valid. Pada beberapa versi Shield, filter `group` dan `permission` mengalihkan (redirect `302`) alih-alih mengembalikan JSON `403`. Bila klien Anda menerima `302`, buat filter kustom yang mengembalikan JSON `403`, seperti `ApiForceResetFilter`.
4. **Scope token:** `["*"]` memberi akses ke semua scope. Scope tidak dapat diubah setelah token dibuat.
5. **Kedaluwarsa vs masa pakai:** `expires_at` adalah tanggal pasti berhentinya token. Terpisah dari `unusedTokenLifetime` (default 1 tahun sejak terakhir dipakai).
6. **Soft delete:** `DELETE /users/{id}` tanpa `hard=1` hanya menandai user terhapus dan tidak menghapus datanya.
7. **Format tanggal:** `Y-m-d H:i:s`. Untuk input `expires_at`, format ISO 8601 (`2026-12-31T23:59`) juga diterima.

### Contoh cURL

```bash
# Login
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"secret123"}'

# Daftar user
curl "http://localhost:8080/api/users?page=1&per_page=10" \
  -H "Authorization: Bearer <token>"

# Ban user
curl -X POST http://localhost:8080/api/users/5/ban \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"message":"Melanggar ToS"}'

# Sinkronisasi grup
curl -X PUT http://localhost:8080/api/users/5/groups \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"groups":["admin","beta"]}'
```
