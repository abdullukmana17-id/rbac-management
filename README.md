# Dokumentasi RESTful API

API manajemen autentikasi, pengguna, grup, permission, token akses, dan HMAC key.

- **Base URL:** `http://localhost:8080/api`
- **Format:** JSON (`Accept: application/json`, `Content-Type: application/json` untuk request ber-body)
- **Autentikasi:** Bearer token (`Authorization: Bearer <token>`)
- **Zona waktu timestamp:** `YYYY-MM-DD HH:mm:ss`

---

## Daftar Isi

1. [Konvensi Umum](#1-konvensi-umum)
2. [Auth](#2-auth)
3. [Users](#3-users)
4. [Status Akun (Ban, Aktivasi, Force Reset)](#4-status-akun)
5. [Grup Pengguna](#5-grup-pengguna)
6. [Permission Pengguna](#6-permission-pengguna)
7. [Access Tokens](#7-access-tokens)
8. [HMAC Keys](#8-hmac-keys)
9. [Ringkasan Endpoint](#9-ringkasan-endpoint)
10. [Catatan Temuan Pengujian](#10-catatan-temuan-pengujian)

---

## 1. Konvensi Umum

### Header

| Header          | Nilai                  | Keterangan                                   |
|-----------------|------------------------|----------------------------------------------|
| `Accept`        | `application/json`     | Wajib                                        |
| `Content-Type`  | `application/json`     | Wajib untuk `POST`/`PUT`/`PATCH` ber-body    |
| `Authorization` | `Bearer <token>`       | Wajib untuk endpoint terproteksi             |

### Format Respons Sukses

Resource tunggal:

```json
{ "data": { "id": 4, "username": "..." } }
```

Koleksi dengan paginasi:

```json
{
  "data": [ ],
  "meta":  { "page": 1, "perPage": 10, "total": 1, "totalPages": 1 },
  "links": { "self": "...", "first": "...", "last": "...", "next": null, "previous": null }
}
```

### Format Respons Error

Error validasi / aplikasi:

```json
{
  "status": 400,
  "error": 400,
  "messages": { "field": "Pesan error" }
}
```

Error token tidak valid (dari lapisan autentikasi):

```json
{ "message": "The access token is invalid." }
```

Error wajib ganti password:

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": { "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password." }
}
```

### Kode Status HTTP

| Kode | Arti                                                                      |
|------|---------------------------------------------------------------------------|
| 200  | Berhasil                                                                  |
| 201  | Resource berhasil dibuat                                                  |
| 204  | Berhasil, tanpa body                                                      |
| 400  | Validasi gagal                                                            |
| 401  | Kredensial salah / token tidak valid / tidak ada token                    |
| 403  | Akun dibanned, atau wajib ganti password                                  |
| 404  | Resource tidak ditemukan                                                  |
| 429  | Terlalu banyak request (rate limit)                                       |

### Objek User

| Field                     | Tipe     | Keterangan                                            |
|---------------------------|----------|-------------------------------------------------------|
| `id`                      | integer  | ID pengguna                                           |
| `username`                | string   |                                                       |
| `email`                   | string   |                                                       |
| `active`                  | boolean  | Status aktif                                          |
| `banned`                  | boolean  | Status banned                                         |
| `requires_password_reset` | boolean  | `true` jika wajib ganti password                      |
| `groups`                  | string[] | Daftar grup, mis. `["user"]`                          |
| `created_at`              | string   | Waktu pembuatan                                       |
| `ban_message`             | string / null | Alasan ban (hanya pada detail/aksi tertentu)     |
| `user_permissions`        | string[] | Permission langsung milik user (hanya pada detail)    |

### Objek Token

| Field          | Tipe          | Keterangan                                    |
|----------------|---------------|-----------------------------------------------|
| `id`           | integer       |                                               |
| `name`         | string        | Nama token                                    |
| `scopes`       | string[]      | Scope, `["*"]` berarti semua                  |
| `key`          | string / null | Terisi hanya untuk HMAC key                   |
| `last_used_at` | string / null | Terakhir digunakan                            |
| `expires_at`   | string / null | Waktu kedaluwarsa (`null` = tidak kedaluwarsa)|
| `created_at`   | string        |                                               |

---

## 2. Auth

### 2.1 Register

`POST /auth/register` — *publik*

**Body**

| Field         | Tipe   | Wajib | Aturan                          |
|---------------|--------|-------|---------------------------------|
| `username`    | string | ya    |                                 |
| `email`       | string | ya    | format email valid              |
| `password`    | string | ya    | minimal 8 karakter              |
| `device_name` | string | tidak | Nama perangkat untuk token      |

**Respons `201 Created`**

```json
{
  "user": {
    "id": 5,
    "username": "apireg332150",
    "email": "apireg.1792918332150@example.com",
    "active": true,
    "banned": false,
    "requires_password_reset": false,
    "groups": ["user"],
    "created_at": "2026-10-25 08:52:19"
  },
  "token": "<token>"
}
```

**Respons `400 Bad Request`**

```json
{
  "status": 400,
  "error": 400,
  "messages": {
    "email": "The email field must contain a valid email address.",
    "password": "The password field must be at least 8 characters in length."
  }
}
```

---

### 2.2 Login

`POST /auth/login` — *publik*

**Body**

| Field         | Tipe   | Wajib | Keterangan              |
|---------------|--------|-------|-------------------------|
| `email`       | string | ya    |                         |
| `password`    | string | ya    |                         |
| `device_name` | string | tidak | Nama token yang dibuat  |

**Contoh request**

```json
{
  "email": "abdullukmana17@gmail.com",
  "password": "********",
  "device_name": "api-test"
}
```

**Respons `200 OK`**

```json
{
  "user": {
    "id": 1,
    "username": "abdullukmana",
    "email": "abdullukmana17@gmail.com",
    "active": true,
    "banned": false,
    "requires_password_reset": false,
    "groups": ["superadmin"],
    "created_at": "2026-10-25 05:14:23"
  },
  "token": "<token>",
  "requires_password_reset": false
}
```

**Respons error**

| Status | Kondisi              | Body                                                                                                                                         |
|--------|----------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| 400    | Validasi gagal       | `{"status":400,"error":400,"messages":{"email":"The email field must contain a valid email address.","password":"The password field is required."}}` |
| 401    | Kredensial salah     | `{"status":401,"error":401,"messages":{"error":"Unable to log you in. Please check your password."}}`                                        |
| 403    | Akun dibanned        | `{"status":403,"error":403,"messages":{"error":"Akun dibanned: Melanggar ketentuan layanan"}}`                                               |

---

### 2.3 Profil Saya

`GET /auth/me` — *Bearer token*

**Respons `200 OK`**

```json
{
  "data": {
    "id": 4,
    "username": "apitest332150x",
    "email": "apitest.1792918332150@example.com",
    "active": true,
    "banned": false,
    "requires_password_reset": false,
    "groups": ["user"],
    "created_at": "2026-10-25 08:52:13",
    "ban_message": null,
    "user_permissions": []
  }
}
```

**Respons `401 Unauthorized`** (tanpa token / token tidak valid)

```json
{ "message": "The access token is invalid." }
```

---

### 2.4 Ganti Password

`POST /auth/change-password` — *Bearer token*

Endpoint ini tetap dapat diakses ketika akun ditandai wajib reset password.

**Body**

| Field              | Tipe   | Wajib | Keterangan            |
|--------------------|--------|-------|-----------------------|
| `current_password` | string | ya    | Password saat ini     |
| `new_password`     | string | ya    | Password baru         |

**Respons `200 OK`**

```json
{
  "message": "Password berhasil diubah.",
  "token": "<token baru>"
}
```

**Respons `401 Unauthorized`** — password saat ini salah

```json
{
  "status": 401,
  "error": 401,
  "messages": { "error": "Password saat ini salah." }
}
```

---

### 2.5 Logout

`POST /auth/logout` — *Bearer token*

Mencabut token yang sedang digunakan.

**Respons** — lihat [catatan temuan](#10-catatan-temuan-pengujian); pada pengujian endpoint ini mengembalikan `429`.

---

## 3. Users

> Seluruh endpoint di bagian 3–6 **hanya untuk admin** (mis. grup `superadmin`).

### 3.1 Buat User

`POST /users`

**Body**

| Field                  | Tipe    | Wajib | Aturan                              |
|------------------------|---------|-------|-------------------------------------|
| `username`             | string  | ya    |                                     |
| `email`                | string  | ya    | format email valid                  |
| `password`             | string  | ya    | minimal 8 karakter                  |
| `force_password_reset` | boolean | tidak | Wajib ganti password saat login     |

**Respons `201 Created`**

```json
{
  "data": {
    "id": 4,
    "username": "apitest332150",
    "email": "apitest.1792918332150@example.com",
    "active": true,
    "banned": false,
    "requires_password_reset": false,
    "groups": ["user"],
    "created_at": "2026-10-25 08:52:13",
    "ban_message": null,
    "user_permissions": []
  }
}
```

**Respons `400 Bad Request`**

```json
{
  "status": 400,
  "error": 400,
  "messages": {
    "email": "The email field must contain a valid email address.",
    "password": "The password field must be at least 8 characters in length."
  }
}
```

---

### 3.2 Daftar User

`GET /users`

**Query parameter**

| Parameter  | Tipe    | Default | Keterangan                        |
|------------|---------|---------|-----------------------------------|
| `page`     | integer | 1       | Halaman                           |
| `per_page` | integer | —       | Jumlah item per halaman           |
| `q`        | string  | —       | Kata kunci pencarian              |

**Contoh:** `GET /users?page=1&per_page=10&q=apitest332150`

**Respons `200 OK`**

```json
{
  "data": [
    {
      "id": 4,
      "username": "apitest332150",
      "email": "apitest.1792918332150@example.com",
      "active": true,
      "banned": false,
      "requires_password_reset": false,
      "groups": ["user"],
      "created_at": "2026-10-25 08:52:13"
    }
  ],
  "meta": { "page": 1, "perPage": 10, "total": 1, "totalPages": 1 },
  "links": {
    "self": "http://localhost:8080/api/users?page=1&per_page=10&q=apitest332150",
    "first": "http://localhost:8080/api/users?page=1&per_page=10&q=apitest332150",
    "last": "http://localhost:8080/api/users?page=1&per_page=10&q=apitest332150",
    "next": null,
    "previous": null
  }
}
```

---

### 3.3 Detail User

`GET /users/{id}`

**Respons `200 OK`** — objek user lengkap (termasuk `ban_message` dan `user_permissions`) dalam `data`.

**Respons `404 Not Found`**

```json
{
  "status": 404,
  "error": 404,
  "messages": { "error": "User 99999999 tidak ditemukan." }
}
```

---

### 3.4 Ubah User

`PATCH /users/{id}`

**Body** (parsial; kirim field yang diubah)

```json
{ "username": "apitest332150x" }
```

**Respons `200 OK`** — objek user terbaru dalam `data`.

---

### 3.5 Hapus User

`DELETE /users/{id}`

| Query  | Keterangan                                              |
|--------|---------------------------------------------------------|
| `hard` | `1` untuk hapus permanen. Tanpa parameter = *soft delete* |

**Contoh:** `DELETE /users/5` (soft delete), `DELETE /users/4?hard=1` (permanen)

**Respons `200 OK`**

```json
{ "id": 5 }
```

**Respons `404 Not Found`**

```json
{
  "status": 404,
  "error": 404,
  "messages": { "error": "User 99999999 tidak ditemukan." }
}
```

---

## 4. Status Akun

### 4.1 Ban User

`POST /users/{id}/ban`

**Body**

| Field     | Tipe   | Keterangan    |
|-----------|--------|---------------|
| `message` | string | Alasan ban    |

**Respons `200 OK`**

```json
{
  "data": {
    "id": 4,
    "username": "apitest332150x",
    "email": "apitest.1792918332150@example.com",
    "active": true,
    "banned": true,
    "requires_password_reset": false,
    "groups": ["user"],
    "created_at": "2026-10-25 08:52:13",
    "ban_message": "Melanggar ketentuan layanan",
    "user_permissions": []
  }
}
```

User yang dibanned akan menerima `403` saat login.

### 4.2 Unban User

`DELETE /users/{id}/ban`

**Respons `200 OK`** — objek user dengan `banned: false` dan `ban_message: null`.

### 4.3 Nonaktifkan User

`POST /users/{id}/deactivate`

**Respons `200 OK`** — objek user dalam `data`.

### 4.4 Aktifkan User

`POST /users/{id}/activate`

**Respons `200 OK`** — objek user dalam `data` dengan `active: true`.

### 4.5 Tandai Wajib Reset Password (satu user)

`POST /users/{id}/force-reset`

**Respons `200 OK`** — objek user dengan `requires_password_reset: true`.

Selama status ini aktif, user hanya boleh memanggil `POST /auth/change-password`; endpoint lain (mis. `GET /tokens`) mengembalikan:

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": { "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password." }
}
```

### 4.6 Batalkan Wajib Reset Password (satu user)

`DELETE /users/{id}/force-reset`

**Respons `200 OK`** — objek user dengan `requires_password_reset: false`.

### 4.7 Tandai Wajib Reset Password (beberapa user)

`POST /users/force-reset`

**Body**

| Field      | Tipe      | Wajib | Keterangan             |
|------------|-----------|-------|------------------------|
| `user_ids` | integer[] | ya    | Tidak boleh kosong     |

```json
{ "user_ids": [4] }
```

**Respons `200 OK`**

```json
{
  "message": "Force reset diterapkan.",
  "user_ids": [4]
}
```

**Respons `400 Bad Request`**

```json
{
  "status": 400,
  "error": 400,
  "messages": { "user_ids": "user_ids wajib berupa array." }
}
```

---

## 5. Grup Pengguna

### 5.1 Daftar Grup User

`GET /users/{id}/groups`

**Respons `200 OK`**

```json
{ "data": ["user"] }
```

### 5.2 Tambah Grup

`POST /users/{id}/groups`

**Body**

```json
{ "groups": ["beta"] }
```

**Respons `201 Created`** — daftar grup terbaru

```json
{ "data": ["user", "beta"] }
```

**Respons `400 Bad Request`** — grup tidak valid

```json
{
  "status": 400,
  "error": 400,
  "messages": { "groups": "grup-tidak-ada is not a valid group." }
}
```

### 5.3 Sinkronisasi Grup

`PUT /users/{id}/groups`

Mengganti seluruh grup user dengan daftar yang dikirim.

**Body**

```json
{ "groups": ["user", "beta"] }
```

**Respons `200 OK`**

```json
{ "data": ["user", "beta"] }
```

### 5.4 Hapus Grup

`DELETE /users/{id}/groups/{group}`

**Contoh:** `DELETE /users/4/groups/beta`

**Respons `200 OK`**

```json
{ "data": ["user"] }
```

---

## 6. Permission Pengguna

### 6.1 Daftar Permission

`GET /users/{id}/permissions`

**Respons `200 OK`**

```json
{ "data": [] }
```

### 6.2 Sinkronisasi Permission

`PUT /users/{id}/permissions`

Mengganti seluruh permission langsung milik user. Kirim array kosong untuk menghapus semua.

**Body**

```json
{ "permissions": ["beta.access"] }
```

**Respons `200 OK`**

```json
{ "data": ["beta.access"] }
```

### 6.3 Cek Permission

`POST /users/{id}/permissions/check`

**Body**

```json
{ "permission": "beta.access" }
```

**Respons `200 OK`**

```json
{ "permission": "beta.access", "allowed": true }
```

Jika tidak dimiliki:

```json
{ "permission": "users.delete", "allowed": false }
```

---

## 7. Access Tokens

Token milik user yang sedang login (berdasarkan Bearer token).

### 7.1 Daftar Token

`GET /tokens`

**Respons `200 OK`**

```json
{
  "data": [
    {
      "id": 17,
      "name": "api-test-user",
      "scopes": ["*"],
      "key": null,
      "last_used_at": "2026-10-25 08:52:15",
      "expires_at": null,
      "created_at": "2026-10-25 08:52:14"
    }
  ]
}
```

### 7.2 Buat Token

`POST /tokens`

**Body**

| Field        | Tipe     | Wajib | Keterangan                              |
|--------------|----------|-------|-----------------------------------------|
| `name`       | string   | ya    | Nama token                              |
| `scopes`     | string[] | tidak | Mis. `["posts.manage"]`                 |
| `expires_at` | string   | tidak | `YYYY-MM-DD HH:mm:ss`                   |

```json
{
  "name": "Mobile App",
  "scopes": ["posts.manage"],
  "expires_at": "2026-12-31 23:59:59"
}
```

**Respons `201 Created`**

```json
{
  "data": {
    "id": 18,
    "name": "Mobile App",
    "scopes": ["posts.manage"],
    "key": null,
    "last_used_at": null,
    "expires_at": "2026-12-31 23:59:59",
    "created_at": "2026-10-25 08:52:15"
  },
  "token": "<token>"
}
```

> Nilai `token` hanya ditampilkan **sekali** saat pembuatan. Simpan dengan aman.

**Respons `400 Bad Request`**

```json
{
  "status": 400,
  "error": 400,
  "messages": { "name": "The name field is required." }
}
```

### 7.3 Ubah Kedaluwarsa Token

`PATCH /tokens/{id}/expiration`

**Body**

```json
{ "expires_at": "2027-01-15 00:00:00" }
```

**Respons `200 OK`** — objek token terbaru dalam `data`.

### 7.4 Cabut Satu Token

`DELETE /tokens/{id}`

**Respons `200 OK`**

```json
{ "id": 18 }
```

**Respons `404 Not Found`**

```json
{
  "status": 404,
  "error": 404,
  "messages": { "error": "Token tidak ditemukan." }
}
```

### 7.5 Cabut Semua Token

`DELETE /tokens`

**Respons `204 No Content`** (tanpa body).

Setelah semua token dicabut, token yang dipakai tidak berlaku lagi dan request berikutnya menghasilkan `401`:

```json
{ "message": "The access token is invalid." }
```

---

## 8. HMAC Keys

Kunci HMAC untuk autentikasi server-to-server.

### 8.1 Daftar HMAC Key

`GET /hmac-keys`

**Respons `200 OK`**

```json
{ "data": [] }
```

### 8.2 Buat HMAC Key

`POST /hmac-keys`

**Body**

| Field    | Tipe     | Wajib | Keterangan            |
|----------|----------|-------|-----------------------|
| `name`   | string   | ya    | Nama key              |
| `scopes` | string[] | tidak | Mis. `["*"]`          |

**Respons `201 Created`**

```json
{
  "data": {
    "id": 19,
    "name": "Partner API",
    "scopes": ["*"],
    "key": "52331f94474fb1133355df49b3fe98b0",
    "last_used_at": null,
    "expires_at": null,
    "created_at": "2026-10-25 08:52:16"
  },
  "key": "52331f94474fb1133355df49b3fe98b0",
  "secret_key": "<secret>"
}
```

> `secret_key` hanya ditampilkan **sekali** saat pembuatan.

### 8.3 Cabut Satu HMAC Key

`DELETE /hmac-keys/{key}`

**Contoh:** `DELETE /hmac-keys/52331f94474fb1133355df49b3fe98b0`

**Respons `200 OK`**

```json
{ "key": "52331f94474fb1133355df49b3fe98b0" }
```

### 8.4 Cabut Semua HMAC Key

`DELETE /hmac-keys`

**Respons `204 No Content`** (tanpa body).

---

## 9. Ringkasan Endpoint

| Method | Endpoint                          | Akses        | Deskripsi                         |
|--------|-----------------------------------|--------------|-----------------------------------|
| POST   | `/auth/register`                  | Publik       | Registrasi                        |
| POST   | `/auth/login`                     | Publik       | Login                             |
| GET    | `/auth/me`                        | Token        | Profil saya                       |
| POST   | `/auth/change-password`           | Token        | Ganti password                    |
| POST   | `/auth/logout`                    | Token        | Logout                            |
| GET    | `/users`                          | Admin        | Daftar user (paginasi, pencarian) |
| POST   | `/users`                          | Admin        | Buat user                         |
| GET    | `/users/{id}`                     | Admin        | Detail user                       |
| PATCH  | `/users/{id}`                     | Admin        | Ubah user                         |
| DELETE | `/users/{id}`                     | Admin        | Hapus user (`?hard=1` permanen)   |
| POST   | `/users/{id}/ban`                 | Admin        | Ban user                          |
| DELETE | `/users/{id}/ban`                 | Admin        | Unban user                        |
| POST   | `/users/{id}/activate`            | Admin        | Aktifkan user                     |
| POST   | `/users/{id}/deactivate`          | Admin        | Nonaktifkan user                  |
| POST   | `/users/{id}/force-reset`         | Admin        | Wajib reset password              |
| DELETE | `/users/{id}/force-reset`         | Admin        | Batalkan wajib reset              |
| POST   | `/users/force-reset`              | Admin        | Wajib reset untuk banyak user     |
| GET    | `/users/{id}/groups`              | Admin        | Daftar grup                       |
| POST   | `/users/{id}/groups`              | Admin        | Tambah grup                       |
| PUT    | `/users/{id}/groups`              | Admin        | Sinkronisasi grup                 |
| DELETE | `/users/{id}/groups/{group}`      | Admin        | Hapus grup                        |
| GET    | `/users/{id}/permissions`         | Admin        | Daftar permission                 |
| PUT    | `/users/{id}/permissions`         | Admin        | Sinkronisasi permission           |
| POST   | `/users/{id}/permissions/check`   | Admin        | Cek permission                    |
| GET    | `/tokens`                         | Token        | Daftar token                      |
| POST   | `/tokens`                         | Token        | Buat token                        |
| PATCH  | `/tokens/{id}/expiration`         | Token        | Ubah kedaluwarsa token            |
| DELETE | `/tokens/{id}`                    | Token        | Cabut satu token                  |
| DELETE | `/tokens`                         | Token        | Cabut semua token                 |
| GET    | `/hmac-keys`                      | Token        | Daftar HMAC key                   |
| POST   | `/hmac-keys`                      | Token        | Buat HMAC key                     |
| DELETE | `/hmac-keys/{key}`                | Token        | Cabut satu HMAC key               |
| DELETE | `/hmac-keys`                      | Token        | Cabut semua HMAC key              |

---

## 10. Catatan Temuan Pengujian

Hasil pengujian otomatis (57 skenario) menunjukkan beberapa perilaku yang perlu ditinjau sebelum dokumentasi ini dianggap final:

1. **User non-admin mengakses `GET /users` menghasilkan `302` (redirect ke `http://localhost:8080`) dengan body HTML**, bukan `403` JSON. Filter otorisasi sebaiknya mengembalikan `403` berformat JSON untuk request API.
2. **`POST /auth/logout` mengembalikan `429` dengan body HTML** (debugbar CodeIgniter) pada dua pemanggilan berturut-turut, termasuk `cleanup: logout admin`. Kemungkinan terkena *throttle*; respons rate limit sebaiknya berupa JSON dan perilaku logout yang sukses perlu didokumentasikan setelah diperbaiki.
3. **`POST /users/{id}/deactivate` mengembalikan `active: true`** pada respons, padahal seharusnya `false`. Periksa apakah data respons diambil sebelum perubahan disimpan.
4. **Format error tidak seragam**: ada `{"status","error","messages"}`, `{"message"}` (token tidak valid), dan `{"status","code","messages"}` (wajib reset). Disarankan menyeragamkan.
5. **Bahasa pesan campuran** (Inggris dari validator/auth, Indonesia dari logika aplikasi).
6. **Default `per_page`** dan batas maksimumnya belum terlihat dari hasil pengujian.
7. **Endpoint `/tokens` dan `/hmac-keys`** tampak beroperasi pada token milik user yang sedang login; nilai `token`/`secret_key` hanya muncul saat pembuatan.
8. Pada lingkungan pengembangan, respons error non-JSON menyertakan *debug toolbar*; pastikan dinonaktifkan di produksi.
