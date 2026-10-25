# Dokumentasi REST API — CodeIgniter 4 + Shield

Dokumentasi **setiap request**: header, parameter, body, contoh cURL, response sukses, dan response error. Total **34 endpoint**.

## Daftar Isi

- [1. Informasi Umum](#1-informasi-umum)
- [2. Autentikasi](#2-autentikasi)
- [3. Format Response & Kode Status](#3-format-response--kode-status)
- [4. Auth](#4-auth)
  - [POST /auth/register](#41-post-authregister--registrasi) — Registrasi
  - [POST /auth/login](#42-post-authlogin--login) — Login
  - [GET /auth/me](#43-get-authme--profil-saya) — Profil Saya
  - [POST /auth/logout](#44-post-authlogout--logout) — Logout
  - [POST /auth/change-password](#45-post-authchange-password--ganti-password) — Ganti Password
- [5. Access Token](#5-access-token)
  - [GET /tokens](#51-get-tokens--daftar-access-token) — Daftar Access Token
  - [POST /tokens](#52-post-tokens--buat-access-token) — Buat Access Token
  - [PATCH /tokens/{id}/expiration](#53-patch-tokensidexpiration--ubah-kedaluwarsa-token) — Ubah Kedaluwarsa Token
  - [DELETE /tokens/{id}](#54-delete-tokensid--cabut-satu-access-token) — Cabut Satu Access Token
  - [DELETE /tokens](#55-delete-tokens--cabut-semua-access-token) — Cabut Semua Access Token
- [6. HMAC Key](#6-hmac-key)
  - [GET /hmac-keys](#61-get-hmac-keys--daftar-hmac-key) — Daftar HMAC Key
  - [POST /hmac-keys](#62-post-hmac-keys--buat-hmac-key) — Buat HMAC Key
  - [DELETE /hmac-keys/{key}](#63-delete-hmac-keyskey--cabut-satu-hmac-key) — Cabut Satu HMAC Key
  - [DELETE /hmac-keys](#64-delete-hmac-keys--cabut-semua-hmac-key) — Cabut Semua HMAC Key
- [7. Users](#7-users)
  - [GET /users](#71-get-users--daftar-user) — Daftar User
  - [GET /users/{id}](#72-get-usersid--detail-user) — Detail User
  - [POST /users](#73-post-users--buat-user) — Buat User
  - [PATCH /users/{id}](#74-patch-usersid--ubah-user) — Ubah User
  - [DELETE /users/{id}](#75-delete-usersid--hapus-user) — Hapus User
- [8. Ban, Aktivasi & Force Reset](#8-ban-aktivasi--force-reset)
  - [POST /users/{id}/ban](#81-post-usersidban--ban-user) — Ban User
  - [DELETE /users/{id}/ban](#82-delete-usersidban--unban-user) — Unban User
  - [POST /users/{id}/activate](#83-post-usersidactivate--aktifkan-user) — Aktifkan User
  - [POST /users/{id}/deactivate](#84-post-usersiddeactivate--nonaktifkan-user) — Nonaktifkan User
  - [POST /users/{id}/force-reset](#85-post-usersidforce-reset--paksa-reset-password-satu-user) — Paksa Reset Password (Satu User)
  - [DELETE /users/{id}/force-reset](#86-delete-usersidforce-reset--batalkan-paksa-reset-password) — Batalkan Paksa Reset Password
  - [POST /users/force-reset](#87-post-usersforce-reset--paksa-reset-password-beberapa-user) — Paksa Reset Password (Beberapa User)
  - [POST /users/force-reset-all](#88-post-usersforce-reset-all--paksa-reset-password-semua-user) — Paksa Reset Password (Semua User)
- [9. Grup & Permission](#9-grup--permission)
  - [GET /users/{id}/groups](#91-get-usersidgroups--daftar-grup-user) — Daftar Grup User
  - [POST /users/{id}/groups](#92-post-usersidgroups--tambah-grup) — Tambah Grup
  - [PUT /users/{id}/groups](#93-put-usersidgroups--sinkronisasi-grup) — Sinkronisasi Grup
  - [DELETE /users/{id}/groups/{group}](#94-delete-usersidgroupsgroup--hapus-grup) — Hapus Grup
  - [GET /users/{id}/permissions](#95-get-usersidpermissions--daftar-permission-langsung) — Daftar Permission Langsung
  - [PUT /users/{id}/permissions](#96-put-usersidpermissions--sinkronisasi-permission) — Sinkronisasi Permission
  - [POST /users/{id}/permissions/check](#97-post-usersidpermissionscheck--cek-permission) — Cek Permission
- [10. Permission yang Dibutuhkan](#10-permission-yang-dibutuhkan)
- [11. Catatan Penting](#11-catatan-penting)

---

## 1. Informasi Umum

| Item | Nilai |
|---|---|
| Base URL | `http://localhost:8080/api` |
| Format | JSON |
| Header standar | `Accept: application/json`, dan `Content-Type: application/json` untuk request ber-body |
| Autentikasi | `Authorization: Bearer <token>` |

Keterangan kolom **Akses**: *Publik* = tanpa token; *Token* = user yang login; *Admin* = grup `admin`/`superadmin` plus permission yang disebut; *Superadmin* = grup `superadmin`.

## 2. Autentikasi

**Bearer Token** — dapatkan lewat `POST /auth/login`, lalu kirim di setiap request:

```http
Authorization: Bearer 5f4c2a9e7b...
```

Token mentah hanya tampil **sekali** saat dibuat; server hanya menyimpan hash (SHA-256).

**HMAC-SHA256 (opsional)** — bila filter `hmac` dipasang pada rute tertentu:

```http
Authorization: HMAC-SHA256 <key>:<hmac-sha256-dari-body-request>
```

```php
$signature = hash_hmac('sha256', $requestBody, $secretKey);
```

**Force password reset** — jika akun wajib reset password, semua endpoint **kecuali** `/auth/me`, `/auth/logout`, dan `/auth/change-password` membalas `403` dengan `code: PASSWORD_RESET_REQUIRED` (contoh tertera di tiap endpoint terkait).

## 3. Format Response & Kode Status

Response sukses berbeda per endpoint (lihat masing-masing). Semua error dari controller memakai format:

```json
{ "status": 404, "code": 404, "messages": { "error": "User 99 tidak ditemukan." } }
```

Error validasi (`400`) berisi pesan per field di `messages`.

| Kode | Arti |
|---|---|
| 200 | OK |
| 201 | Created |
| 204 | No Content (tanpa body) |
| 400 | Validasi gagal |
| 401 | Token tidak ada / tidak valid / kredensial salah |
| 403 | Dilarang: diban, belum aktif, tidak punya izin, atau wajib reset password |
| 404 | Tidak ditemukan |
| 429 | Terkena rate limit |

---
## 4. Auth

### 4.1 POST /auth/register — Registrasi

Mendaftarkan user baru dan langsung mengembalikan access token. Hanya aktif jika `Auth::$allowRegistration = true`.

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `http://localhost:8080/api/auth/register` |
| **Akses** | Publik |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Content-Type` | `application/json` | Ya |

**Body (JSON)**

| Field | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `username` | string | Tidak | 3–30 karakter, unik |
| `email` | string | Ya | Email valid, unik |
| `password` | string | Ya | Minimal 8 karakter |
| `device_name` | string | Tidak | Nama token (default `default`) |

**Contoh Body**

```json
{
  "username": "budi",
  "email": "budi@example.com",
  "password": "rahasia123",
  "device_name": "web"
}
```

**Contoh Request**

```bash
curl -X POST "http://localhost:8080/api/auth/register" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"username": "budi", "email": "budi@example.com", "password": "rahasia123", "device_name": "web"}'
```

**Response Sukses — `201 Created`**

```json
{
  "user": {
    "id": 5,
    "username": "budi",
    "email": "budi@example.com",
    "active": true,
    "banned": false,
    "requires_password_reset": false,
    "groups": [
      "user"
    ],
    "created_at": "2026-10-25 06:00:00"
  },
  "token": "5f4c2a9e7b1d..."
}
```

**Error — `400` Validasi gagal**

```json
{
  "status": 400,
  "code": 400,
  "messages": {
    "email": "This email is already registered."
  }
}
```

**Error — `403` Registrasi dinonaktifkan**

```json
{
  "status": 403,
  "code": 403,
  "messages": {
    "error": "Registrasi dinonaktifkan."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `429` — terkena rate limit `auth-rates`.

---

### 4.2 POST /auth/login — Login

Memvalidasi email dan password lalu menerbitkan access token baru.

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `http://localhost:8080/api/auth/login` |
| **Akses** | Publik |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Content-Type` | `application/json` | Ya |

**Body (JSON)**

| Field | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `email` | string | Ya | Email valid |
| `password` | string | Ya | Password akun |
| `device_name` | string | Tidak | Nama token (default `api-login`) |

**Contoh Body**

```json
{
  "email": "admin@example.com",
  "password": "secret123",
  "device_name": "web"
}
```

**Contoh Request**

```bash
curl -X POST "http://localhost:8080/api/auth/login" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@example.com", "password": "secret123", "device_name": "web"}'
```

**Response Sukses — `200 OK`**

```json
{
  "user": {
    "id": 1,
    "username": "admin",
    "email": "admin@example.com",
    "active": true,
    "banned": false,
    "requires_password_reset": false,
    "groups": [
      "admin"
    ],
    "created_at": "2026-10-25 06:00:00"
  },
  "token": "5f4c2a9e7b1d...",
  "requires_password_reset": false
}
```

**Error — `400` Validasi gagal**

```json
{
  "status": 400,
  "code": 400,
  "messages": {
    "email": "The email field must contain a valid email address."
  }
}
```

**Error — `401` Kredensial salah**

```json
{
  "status": 401,
  "code": 401,
  "messages": {
    "error": "Email atau password tidak cocok."
  }
}
```

**Error — `403` Akun diban**

```json
{
  "status": 403,
  "code": 403,
  "messages": {
    "error": "Akun dibanned: Melanggar ToS"
  }
}
```

**Error — `403` Belum aktif**

```json
{
  "status": 403,
  "code": 403,
  "messages": {
    "error": "Akun belum diaktifkan."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `429` — terkena rate limit `auth-rates`.

---

### 4.3 GET /auth/me — Profil Saya

Mengambil data user pemilik token.

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `http://localhost:8080/api/auth/me` |
| **Akses** | Token |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Contoh Request**

```bash
curl -X GET "http://localhost:8080/api/auth/me" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>"
```

**Response Sukses — `200 OK`**

```json
{
  "data": {
    "id": 1,
    "username": "admin",
    "email": "budi@example.com",
    "active": true,
    "banned": false,
    "requires_password_reset": false,
    "groups": [
      "admin"
    ],
    "created_at": "2026-10-25 06:00:00",
    "ban_message": null,
    "user_permissions": [
      "beta.access"
    ]
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas).

---

### 4.4 POST /auth/logout — Logout

Mencabut token yang dipakai pada request ini.

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `http://localhost:8080/api/auth/logout` |
| **Akses** | Token |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Body:** tidak ada.

**Contoh Request**

```bash
curl -X POST "http://localhost:8080/api/auth/logout" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>"
```

**Response Sukses — `204 No Content`**

Tanpa body.

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas).

---

### 4.5 POST /auth/change-password — Ganti Password

Mengganti password. Semua token lama dicabut dan token baru diterbitkan. Flag force reset dihapus. Tetap bisa diakses saat akun wajib reset password.

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `http://localhost:8080/api/auth/change-password` |
| **Akses** | Token |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Content-Type` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Body (JSON)**

| Field | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `current_password` | string | Ya | Password saat ini |
| `new_password` | string | Ya | 8–255 karakter, berbeda dari password lama |
| `device_name` | string | Tidak | Nama token baru |

**Contoh Body**

```json
{
  "current_password": "secret123",
  "new_password": "passwordBaru456"
}
```

**Contoh Request**

```bash
curl -X POST "http://localhost:8080/api/auth/change-password" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"current_password": "secret123", "new_password": "passwordBaru456"}'
```

**Response Sukses — `200 OK`**

```json
{
  "message": "Password berhasil diubah.",
  "token": "a91c0d77e2..."
}
```

**Error — `400` Validasi gagal**

```json
{
  "status": 400,
  "code": 400,
  "messages": {
    "new_password": "The new_password field must differ from the current_password field."
  }
}
```

**Error — `401` Password salah**

```json
{
  "status": 401,
  "code": 401,
  "messages": {
    "error": "Password saat ini salah."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas).

---

## 5. Access Token

### 5.1 GET /tokens — Daftar Access Token

Daftar access token milik user yang login.

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `http://localhost:8080/api/tokens` |
| **Akses** | Token |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Contoh Request**

```bash
curl -X GET "http://localhost:8080/api/tokens" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>"
```

**Response Sukses — `200 OK`**

```json
{
  "data": [
    {
      "id": 3,
      "name": "Work Laptop",
      "scopes": [
        "*"
      ],
      "key": null,
      "last_used_at": null,
      "expires_at": null,
      "created_at": "2026-10-25 06:15:00"
    }
  ]
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas).

---

### 5.2 POST /tokens — Buat Access Token

Membuat access token baru dengan scope dan kedaluwarsa opsional.

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `http://localhost:8080/api/tokens` |
| **Akses** | Token |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Content-Type` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Body (JSON)**

| Field | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `name` | string | Ya | Maks. 255 karakter |
| `scopes` | array<string> | Tidak | Default `["*"]`; tidak bisa diubah kemudian |
| `expires_at` | string|null | Tidak | Mis. `2026-12-31 23:59:59` atau `2026-12-31T23:59` |

**Contoh Body**

```json
{
  "name": "Mobile App",
  "scopes": [
    "posts.manage"
  ],
  "expires_at": "2026-12-31 23:59:59"
}
```

**Contoh Request**

```bash
curl -X POST "http://localhost:8080/api/tokens" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"name": "Mobile App", "scopes": ["posts.manage"], "expires_at": "2026-12-31 23:59:59"}'
```

**Response Sukses — `201 Created`**

```json
{
  "data": {
    "id": 4,
    "name": "Mobile App",
    "scopes": [
      "posts.manage"
    ],
    "key": null,
    "last_used_at": null,
    "expires_at": "2026-12-31 23:59:59",
    "created_at": "2026-10-25 06:15:00"
  },
  "token": "c3b1e5f0a2..."
}
```

**Error — `400` Validasi gagal**

```json
{
  "status": 400,
  "code": 400,
  "messages": {
    "name": "The name field is required."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas).

---

### 5.3 PATCH /tokens/{id}/expiration — Ubah Kedaluwarsa Token

Mengubah tanggal kedaluwarsa token. Kirim `null` untuk menghapusnya.

| | |
|---|---|
| **Method** | `PATCH` |
| **URL** | `http://localhost:8080/api/tokens/{id}/expiration` |
| **Akses** | Token |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Content-Type` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Path Parameter**

| Nama | Tipe | Keterangan |
|---|---|---|
| `id` | integer | ID token |

**Body (JSON)**

| Field | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `expires_at` | string|null | Ya | Tanggal baru, atau `null` untuk menghapus |

**Contoh Body**

```json
{
  "expires_at": "2027-01-15 00:00:00"
}
```

**Contoh Request**

```bash
curl -X PATCH "http://localhost:8080/api/tokens/4/expiration" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"expires_at": "2027-01-15 00:00:00"}'
```

**Response Sukses — `200 OK`**

```json
{
  "data": {
    "id": 4,
    "name": "Mobile App",
    "scopes": [
      "posts.manage"
    ],
    "key": null,
    "last_used_at": null,
    "expires_at": "2027-01-15 00:00:00",
    "created_at": "2026-10-25 06:15:00"
  }
}
```

**Error — `404` Tidak ditemukan**

```json
{
  "status": 404,
  "code": 404,
  "messages": {
    "error": "Token tidak ditemukan."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas).

---

### 5.4 DELETE /tokens/{id} — Cabut Satu Access Token

Mencabut (menghapus) satu token.

| | |
|---|---|
| **Method** | `DELETE` |
| **URL** | `http://localhost:8080/api/tokens/{id}` |
| **Akses** | Token |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Path Parameter**

| Nama | Tipe | Keterangan |
|---|---|---|
| `id` | integer | ID token |

**Contoh Request**

```bash
curl -X DELETE "http://localhost:8080/api/tokens/4" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>"
```

**Response Sukses — `200 OK`**

```json
{
  "id": 4
}
```

**Error — `404` Tidak ditemukan**

```json
{
  "status": 404,
  "code": 404,
  "messages": {
    "error": "Token tidak ditemukan."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas).

---

### 5.5 DELETE /tokens — Cabut Semua Access Token

Mencabut semua token user, termasuk token yang sedang dipakai.

| | |
|---|---|
| **Method** | `DELETE` |
| **URL** | `http://localhost:8080/api/tokens` |
| **Akses** | Token |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Contoh Request**

```bash
curl -X DELETE "http://localhost:8080/api/tokens" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>"
```

**Response Sukses — `204 No Content`**

Tanpa body.

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas).

---

## 6. HMAC Key

### 6.1 GET /hmac-keys — Daftar HMAC Key

Daftar HMAC key milik user. `key` selalu tampil; secret key tidak.

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `http://localhost:8080/api/hmac-keys` |
| **Akses** | Token |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Contoh Request**

```bash
curl -X GET "http://localhost:8080/api/hmac-keys" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>"
```

**Response Sukses — `200 OK`**

```json
{
  "data": [
    {
      "id": 7,
      "name": "Partner API",
      "scopes": [
        "*"
      ],
      "key": "a6c460151b4cabbe1c1d73e08915ce8e",
      "last_used_at": null,
      "expires_at": null,
      "created_at": "2026-10-25 06:15:00"
    }
  ]
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas).

---

### 6.2 POST /hmac-keys — Buat HMAC Key

Membuat sepasang key dan secret key. Pastikan `authtoken.hmacEncryptionKeys` sudah diisi di `.env`.

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `http://localhost:8080/api/hmac-keys` |
| **Akses** | Token |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Content-Type` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Body (JSON)**

| Field | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `name` | string | Ya | Maks. 255 karakter |
| `scopes` | array<string> | Tidak | Default `["*"]` |
| `expires_at` | string|null | Tidak | Tanggal kedaluwarsa |

**Contoh Body**

```json
{
  "name": "Partner API",
  "scopes": [
    "*"
  ]
}
```

**Contoh Request**

```bash
curl -X POST "http://localhost:8080/api/hmac-keys" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"name": "Partner API", "scopes": ["*"]}'
```

**Response Sukses — `201 Created`**

```json
{
  "data": {
    "id": 7,
    "name": "Partner API",
    "scopes": [
      "*"
    ],
    "key": "a6c460151b4cabbe1c1d73e08915ce8e",
    "last_used_at": null,
    "expires_at": null,
    "created_at": "2026-10-25 06:15:00"
  },
  "key": "a6c460151b4cabbe1c1d73e08915ce8e",
  "secret_key": "56c85232f0e5b55c05015476cd132c8d"
}
```

**Error — `400` Validasi gagal**

```json
{
  "status": 400,
  "code": 400,
  "messages": {
    "name": "The name field is required."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas).

---

### 6.3 DELETE /hmac-keys/{key} — Cabut Satu HMAC Key

Mencabut satu HMAC key. Gunakan nilai `key`, bukan `id`.

| | |
|---|---|
| **Method** | `DELETE` |
| **URL** | `http://localhost:8080/api/hmac-keys/{key}` |
| **Akses** | Token |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Path Parameter**

| Nama | Tipe | Keterangan |
|---|---|---|
| `key` | string | Nilai `key` HMAC |

**Contoh Request**

```bash
curl -X DELETE "http://localhost:8080/api/hmac-keys/a6c460151b4cabbe1c1d73e08915ce8e" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>"
```

**Response Sukses — `200 OK`**

```json
{
  "key": "a6c460151b4cabbe1c1d73e08915ce8e"
}
```

**Error — `404` Tidak ditemukan**

```json
{
  "status": 404,
  "code": 404,
  "messages": {
    "error": "HMAC key tidak ditemukan."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas).

---

### 6.4 DELETE /hmac-keys — Cabut Semua HMAC Key

Mencabut semua HMAC key milik user.

| | |
|---|---|
| **Method** | `DELETE` |
| **URL** | `http://localhost:8080/api/hmac-keys` |
| **Akses** | Token |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Contoh Request**

```bash
curl -X DELETE "http://localhost:8080/api/hmac-keys" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>"
```

**Response Sukses — `204 No Content`**

Tanpa body.

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas).

---

## 7. Users

### 7.1 GET /users — Daftar User

Daftar user dengan paginasi. Relasi identitas, grup, dan permission di-preload.

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `http://localhost:8080/api/users` |
| **Akses** | Admin (permission `users.read`) |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Query Parameter**

| Nama | Tipe | Keterangan |
|---|---|---|
| `page` | integer | Nomor halaman (default 1) |
| `per_page` | integer | 1–100 (default 20) |
| `q` | string | Cari username (LIKE) |

**Contoh Request**

```bash
curl -X GET "http://localhost:8080/api/users?page=1&per_page=10&q=budi" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>"
```

**Response Sukses — `200 OK`**

```json
{
  "data": [
    {
      "id": 5,
      "username": "budi",
      "email": "budi@example.com",
      "active": true,
      "banned": false,
      "requires_password_reset": false,
      "groups": [
        "user"
      ],
      "created_at": "2026-10-25 06:00:00"
    }
  ],
  "meta": {
    "page": 1,
    "perPage": 10,
    "total": 1,
    "totalPages": 1
  },
  "links": {
    "self": "http://localhost:8080/api/users?page=1&per_page=10&q=budi",
    "first": "http://localhost:8080/api/users?page=1&per_page=10&q=budi",
    "last": "http://localhost:8080/api/users?page=1&per_page=10&q=budi",
    "next": null,
    "previous": null
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas); `403` — user bukan admin / tidak punya permission yang dibutuhkan (lihat catatan filter di bagian 11).

---

### 7.2 GET /users/{id} — Detail User

Detail satu user, termasuk alasan ban dan permission langsung.

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `http://localhost:8080/api/users/{id}` |
| **Akses** | Admin (permission `users.read`) |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Path Parameter**

| Nama | Tipe | Keterangan |
|---|---|---|
| `id` | integer | ID user |

**Contoh Request**

```bash
curl -X GET "http://localhost:8080/api/users/5" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>"
```

**Response Sukses — `200 OK`**

```json
{
  "data": {
    "id": 5,
    "username": "budi",
    "email": "budi@example.com",
    "active": true,
    "banned": false,
    "requires_password_reset": false,
    "groups": [
      "user"
    ],
    "created_at": "2026-10-25 06:00:00",
    "ban_message": null,
    "user_permissions": []
  }
}
```

**Error — `404` Tidak ditemukan**

```json
{
  "status": 404,
  "code": 404,
  "messages": {
    "error": "User 5 tidak ditemukan."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas); `403` — user bukan admin / tidak punya permission yang dibutuhkan (lihat catatan filter di bagian 11).

---

### 7.3 POST /users — Buat User

Membuat user baru; otomatis masuk grup default.

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `http://localhost:8080/api/users` |
| **Akses** | Admin (permission `users.create`) |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Content-Type` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Body (JSON)**

| Field | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `username` | string | Tidak | 3–30 karakter, unik |
| `email` | string | Ya | Email valid, unik |
| `password` | string | Ya | Min. 8 karakter |
| `force_password_reset` | boolean | Tidak | `true` = wajib ganti password saat login |

**Contoh Body**

```json
{
  "username": "siti",
  "email": "siti@example.com",
  "password": "sementara123",
  "force_password_reset": true
}
```

**Contoh Request**

```bash
curl -X POST "http://localhost:8080/api/users" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"username": "siti", "email": "siti@example.com", "password": "sementara123", "force_password_reset": true}'
```

**Response Sukses — `201 Created`**

```json
{
  "data": {
    "id": 5,
    "username": "siti",
    "email": "siti@example.com",
    "active": true,
    "banned": false,
    "requires_password_reset": true,
    "groups": [
      "user"
    ],
    "created_at": "2026-10-25 06:00:00",
    "ban_message": null,
    "user_permissions": []
  }
}
```

**Error — `400` Validasi gagal**

```json
{
  "status": 400,
  "code": 400,
  "messages": {
    "email": "The email field must contain a valid email address."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas); `403` — user bukan admin / tidak punya permission yang dibutuhkan (lihat catatan filter di bagian 11).

---

### 7.4 PATCH /users/{id} — Ubah User

Mengubah sebagian data user. `PUT /users/{id}` memiliki perilaku yang sama. Hanya field yang dikirim yang diubah.

| | |
|---|---|
| **Method** | `PATCH` |
| **URL** | `http://localhost:8080/api/users/{id}` |
| **Akses** | Admin (permission `users.edit`) |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Content-Type` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Path Parameter**

| Nama | Tipe | Keterangan |
|---|---|---|
| `id` | integer | ID user |

**Body (JSON)**

| Field | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `username` | string | Tidak | 3–30 karakter, unik |
| `email` | string | Tidak | Email valid |
| `password` | string | Tidak | Min. 8 karakter |

**Contoh Body**

```json
{
  "username": "budi_baru",
  "email": "budi.baru@example.com"
}
```

**Contoh Request**

```bash
curl -X PATCH "http://localhost:8080/api/users/5" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"username": "budi_baru", "email": "budi.baru@example.com"}'
```

**Response Sukses — `200 OK`**

```json
{
  "data": {
    "id": 5,
    "username": "budi_baru",
    "email": "budi.baru@example.com",
    "active": true,
    "banned": false,
    "requires_password_reset": false,
    "groups": [
      "user"
    ],
    "created_at": "2026-10-25 06:00:00",
    "ban_message": null,
    "user_permissions": []
  }
}
```

**Error — `400` Validasi gagal**

```json
{
  "status": 400,
  "code": 400,
  "messages": {
    "username": "The username field must contain a unique value."
  }
}
```

**Error — `404` Tidak ditemukan**

```json
{
  "status": 404,
  "code": 404,
  "messages": {
    "error": "User 5 tidak ditemukan."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas); `403` — user bukan admin / tidak punya permission yang dibutuhkan (lihat catatan filter di bagian 11).

---

### 7.5 DELETE /users/{id} — Hapus User

Menghapus user. Default soft delete; gunakan `hard=1` untuk hapus permanen.

| | |
|---|---|
| **Method** | `DELETE` |
| **URL** | `http://localhost:8080/api/users/{id}` |
| **Akses** | Admin (permission `users.delete`) |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Path Parameter**

| Nama | Tipe | Keterangan |
|---|---|---|
| `id` | integer | ID user |

**Query Parameter**

| Nama | Tipe | Keterangan |
|---|---|---|
| `hard` | integer | `1` = hapus permanen |

**Contoh Request**

```bash
curl -X DELETE "http://localhost:8080/api/users/5?hard=1" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>"
```

**Response Sukses — `200 OK`**

```json
{
  "id": 5
}
```

**Error — `403` Hapus akun sendiri**

```json
{
  "status": 403,
  "code": 403,
  "messages": {
    "error": "Tidak bisa menghapus akun sendiri."
  }
}
```

**Error — `404` Tidak ditemukan**

```json
{
  "status": 404,
  "code": 404,
  "messages": {
    "error": "User 5 tidak ditemukan."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas); `403` — user bukan admin / tidak punya permission yang dibutuhkan (lihat catatan filter di bagian 11).

---

## 8. Ban, Aktivasi & Force Reset

### 8.1 POST /users/{id}/ban — Ban User

Memblokir user dan mencabut semua token-nya.

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `http://localhost:8080/api/users/{id}/ban` |
| **Akses** | Admin (permission `users.ban`) |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Content-Type` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Path Parameter**

| Nama | Tipe | Keterangan |
|---|---|---|
| `id` | integer | ID user |

**Body (JSON)**

| Field | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `message` | string | Tidak | Alasan ban |

**Contoh Body**

```json
{
  "message": "Melanggar ketentuan layanan"
}
```

**Contoh Request**

```bash
curl -X POST "http://localhost:8080/api/users/5/ban" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"message": "Melanggar ketentuan layanan"}'
```

**Response Sukses — `200 OK`**

```json
{
  "data": {
    "id": 5,
    "username": "budi",
    "email": "budi@example.com",
    "active": true,
    "banned": true,
    "requires_password_reset": false,
    "groups": [
      "user"
    ],
    "created_at": "2026-10-25 06:00:00",
    "ban_message": "Melanggar ketentuan layanan",
    "user_permissions": []
  }
}
```

**Error — `403` Ban akun sendiri**

```json
{
  "status": 403,
  "code": 403,
  "messages": {
    "error": "Tidak bisa memblokir akun sendiri."
  }
}
```

**Error — `404` Tidak ditemukan**

```json
{
  "status": 404,
  "code": 404,
  "messages": {
    "error": "User 5 tidak ditemukan."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas); `403` — user bukan admin / tidak punya permission yang dibutuhkan (lihat catatan filter di bagian 11).

---

### 8.2 DELETE /users/{id}/ban — Unban User

Mencabut ban user.

| | |
|---|---|
| **Method** | `DELETE` |
| **URL** | `http://localhost:8080/api/users/{id}/ban` |
| **Akses** | Admin (permission `users.ban`) |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Path Parameter**

| Nama | Tipe | Keterangan |
|---|---|---|
| `id` | integer | ID user |

**Contoh Request**

```bash
curl -X DELETE "http://localhost:8080/api/users/5/ban" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>"
```

**Response Sukses — `200 OK`**

```json
{
  "data": {
    "id": 5,
    "username": "budi",
    "email": "budi@example.com",
    "active": true,
    "banned": false,
    "requires_password_reset": false,
    "groups": [
      "user"
    ],
    "created_at": "2026-10-25 06:00:00",
    "ban_message": null,
    "user_permissions": []
  }
}
```

**Error — `404` Tidak ditemukan**

```json
{
  "status": 404,
  "code": 404,
  "messages": {
    "error": "User 5 tidak ditemukan."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas); `403` — user bukan admin / tidak punya permission yang dibutuhkan (lihat catatan filter di bagian 11).

---

### 8.3 POST /users/{id}/activate — Aktifkan User

Mengaktifkan akun user.

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `http://localhost:8080/api/users/{id}/activate` |
| **Akses** | Admin (permission `users.edit`) |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Path Parameter**

| Nama | Tipe | Keterangan |
|---|---|---|
| `id` | integer | ID user |

**Body:** tidak ada.

**Contoh Request**

```bash
curl -X POST "http://localhost:8080/api/users/5/activate" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>"
```

**Response Sukses — `200 OK`**

```json
{
  "data": {
    "id": 5,
    "username": "budi",
    "email": "budi@example.com",
    "active": true,
    "banned": false,
    "requires_password_reset": false,
    "groups": [
      "user"
    ],
    "created_at": "2026-10-25 06:00:00"
  }
}
```

**Error — `404` Tidak ditemukan**

```json
{
  "status": 404,
  "code": 404,
  "messages": {
    "error": "User 5 tidak ditemukan."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas); `403` — user bukan admin / tidak punya permission yang dibutuhkan (lihat catatan filter di bagian 11).

---

### 8.4 POST /users/{id}/deactivate — Nonaktifkan User

Menonaktifkan akun user dan mencabut semua token-nya.

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `http://localhost:8080/api/users/{id}/deactivate` |
| **Akses** | Admin (permission `users.edit`) |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Path Parameter**

| Nama | Tipe | Keterangan |
|---|---|---|
| `id` | integer | ID user |

**Body:** tidak ada.

**Contoh Request**

```bash
curl -X POST "http://localhost:8080/api/users/5/deactivate" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>"
```

**Response Sukses — `200 OK`**

```json
{
  "data": {
    "id": 5,
    "username": "budi",
    "email": "budi@example.com",
    "active": false,
    "banned": false,
    "requires_password_reset": false,
    "groups": [
      "user"
    ],
    "created_at": "2026-10-25 06:00:00"
  }
}
```

**Error — `404` Tidak ditemukan**

```json
{
  "status": 404,
  "code": 404,
  "messages": {
    "error": "User 5 tidak ditemukan."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas); `403` — user bukan admin / tidak punya permission yang dibutuhkan (lihat catatan filter di bagian 11).

---

### 8.5 POST /users/{id}/force-reset — Paksa Reset Password (Satu User)

Menandai user wajib mengganti password.

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `http://localhost:8080/api/users/{id}/force-reset` |
| **Akses** | Admin (permission `users.edit`) |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Path Parameter**

| Nama | Tipe | Keterangan |
|---|---|---|
| `id` | integer | ID user |

**Body:** tidak ada.

**Contoh Request**

```bash
curl -X POST "http://localhost:8080/api/users/5/force-reset" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>"
```

**Response Sukses — `200 OK`**

```json
{
  "data": {
    "id": 5,
    "username": "budi",
    "email": "budi@example.com",
    "active": true,
    "banned": false,
    "requires_password_reset": true,
    "groups": [
      "user"
    ],
    "created_at": "2026-10-25 06:00:00"
  }
}
```

**Error — `404` Tidak ditemukan**

```json
{
  "status": 404,
  "code": 404,
  "messages": {
    "error": "User 5 tidak ditemukan."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas); `403` — user bukan admin / tidak punya permission yang dibutuhkan (lihat catatan filter di bagian 11).

---

### 8.6 DELETE /users/{id}/force-reset — Batalkan Paksa Reset Password

Menghapus tanda wajib ganti password.

| | |
|---|---|
| **Method** | `DELETE` |
| **URL** | `http://localhost:8080/api/users/{id}/force-reset` |
| **Akses** | Admin (permission `users.edit`) |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Path Parameter**

| Nama | Tipe | Keterangan |
|---|---|---|
| `id` | integer | ID user |

**Contoh Request**

```bash
curl -X DELETE "http://localhost:8080/api/users/5/force-reset" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>"
```

**Response Sukses — `200 OK`**

```json
{
  "data": {
    "id": 5,
    "username": "budi",
    "email": "budi@example.com",
    "active": true,
    "banned": false,
    "requires_password_reset": false,
    "groups": [
      "user"
    ],
    "created_at": "2026-10-25 06:00:00"
  }
}
```

**Error — `404` Tidak ditemukan**

```json
{
  "status": 404,
  "code": 404,
  "messages": {
    "error": "User 5 tidak ditemukan."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas); `403` — user bukan admin / tidak punya permission yang dibutuhkan (lihat catatan filter di bagian 11).

---

### 8.7 POST /users/force-reset — Paksa Reset Password (Beberapa User)

Menandai banyak user sekaligus wajib ganti password.

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `http://localhost:8080/api/users/force-reset` |
| **Akses** | Admin (permission `users.edit`) |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Content-Type` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Body (JSON)**

| Field | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `user_ids` | array<integer> | Ya | Daftar ID user, tidak boleh kosong |

**Contoh Body**

```json
{
  "user_ids": [
    1,
    2,
    3,
    4
  ]
}
```

**Contoh Request**

```bash
curl -X POST "http://localhost:8080/api/users/force-reset" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"user_ids": [1, 2, 3, 4]}'
```

**Response Sukses — `200 OK`**

```json
{
  "message": "Force reset diterapkan.",
  "user_ids": [
    1,
    2,
    3,
    4
  ]
}
```

**Error — `400` Validasi gagal**

```json
{
  "status": 400,
  "code": 400,
  "messages": {
    "user_ids": "user_ids wajib berupa array."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas); `403` — user bukan admin / tidak punya permission yang dibutuhkan (lihat catatan filter di bagian 11).

---

### 8.8 POST /users/force-reset-all — Paksa Reset Password (Semua User)

Menandai **semua** user wajib ganti password. Hanya untuk grup `superadmin`.

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `http://localhost:8080/api/users/force-reset-all` |
| **Akses** | Superadmin |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Body:** tidak ada.

**Contoh Request**

```bash
curl -X POST "http://localhost:8080/api/users/force-reset-all" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>"
```

**Response Sukses — `200 OK`**

```json
{
  "message": "Force reset diterapkan ke semua user."
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas); `403` — user bukan admin / tidak punya permission yang dibutuhkan (lihat catatan filter di bagian 11).

---

## 9. Grup & Permission

### 9.1 GET /users/{id}/groups — Daftar Grup User

Daftar grup milik user.

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `http://localhost:8080/api/users/{id}/groups` |
| **Akses** | Admin (permission `users.read`) |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Path Parameter**

| Nama | Tipe | Keterangan |
|---|---|---|
| `id` | integer | ID user |

**Contoh Request**

```bash
curl -X GET "http://localhost:8080/api/users/5/groups" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>"
```

**Response Sukses — `200 OK`**

```json
{
  "data": [
    "admin",
    "beta"
  ]
}
```

**Error — `404` Tidak ditemukan**

```json
{
  "status": 404,
  "code": 404,
  "messages": {
    "error": "User 5 tidak ditemukan."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas); `403` — user bukan admin / tidak punya permission yang dibutuhkan (lihat catatan filter di bagian 11).

---

### 9.2 POST /users/{id}/groups — Tambah Grup

Menambahkan satu atau lebih grup tanpa menghapus grup yang ada.

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `http://localhost:8080/api/users/{id}/groups` |
| **Akses** | Admin (permission `users.manage-admins`) |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Content-Type` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Path Parameter**

| Nama | Tipe | Keterangan |
|---|---|---|
| `id` | integer | ID user |

**Body (JSON)**

| Field | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `groups` | array<string> | Ya | Nama grup yang terdaftar di `AuthGroups` |

**Contoh Body**

```json
{
  "groups": [
    "beta"
  ]
}
```

**Contoh Request**

```bash
curl -X POST "http://localhost:8080/api/users/5/groups" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"groups": ["beta"]}'
```

**Response Sukses — `201 Created`**

```json
{
  "data": [
    "user",
    "beta"
  ]
}
```

**Error — `400` Validasi gagal**

```json
{
  "status": 400,
  "code": 400,
  "messages": {
    "groups": "Group 'xyz' is not a valid group."
  }
}
```

**Error — `403` Grup superadmin**

```json
{
  "status": 403,
  "code": 403,
  "messages": {
    "error": "Hanya superadmin yang dapat mengelola grup superadmin."
  }
}
```

**Error — `404` Tidak ditemukan**

```json
{
  "status": 404,
  "code": 404,
  "messages": {
    "error": "User 5 tidak ditemukan."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas); `403` — user bukan admin / tidak punya permission yang dibutuhkan (lihat catatan filter di bagian 11).

---

### 9.3 PUT /users/{id}/groups — Sinkronisasi Grup

Mengganti **seluruh** grup user dengan daftar yang dikirim.

| | |
|---|---|
| **Method** | `PUT` |
| **URL** | `http://localhost:8080/api/users/{id}/groups` |
| **Akses** | Admin (permission `users.manage-admins`) |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Content-Type` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Path Parameter**

| Nama | Tipe | Keterangan |
|---|---|---|
| `id` | integer | ID user |

**Body (JSON)**

| Field | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `groups` | array<string> | Ya | Daftar grup final |

**Contoh Body**

```json
{
  "groups": [
    "admin",
    "beta"
  ]
}
```

**Contoh Request**

```bash
curl -X PUT "http://localhost:8080/api/users/5/groups" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"groups": ["admin", "beta"]}'
```

**Response Sukses — `200 OK`**

```json
{
  "data": [
    "admin",
    "beta"
  ]
}
```

**Error — `400` Validasi gagal**

```json
{
  "status": 400,
  "code": 400,
  "messages": {
    "groups": "groups wajib berupa array."
  }
}
```

**Error — `403` Grup superadmin**

```json
{
  "status": 403,
  "code": 403,
  "messages": {
    "error": "Hanya superadmin yang dapat mengelola grup superadmin."
  }
}
```

**Error — `404` Tidak ditemukan**

```json
{
  "status": 404,
  "code": 404,
  "messages": {
    "error": "User 5 tidak ditemukan."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas); `403` — user bukan admin / tidak punya permission yang dibutuhkan (lihat catatan filter di bagian 11).

---

### 9.4 DELETE /users/{id}/groups/{group} — Hapus Grup

Menghapus satu grup dari user.

| | |
|---|---|
| **Method** | `DELETE` |
| **URL** | `http://localhost:8080/api/users/{id}/groups/{group}` |
| **Akses** | Admin (permission `users.manage-admins`) |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Path Parameter**

| Nama | Tipe | Keterangan |
|---|---|---|
| `id` | integer | ID user |
| `group` | string | Nama grup |

**Contoh Request**

```bash
curl -X DELETE "http://localhost:8080/api/users/5/groups/beta" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>"
```

**Response Sukses — `200 OK`**

```json
{
  "data": [
    "user"
  ]
}
```

**Error — `403` Grup superadmin**

```json
{
  "status": 403,
  "code": 403,
  "messages": {
    "error": "Hanya superadmin yang dapat mengelola grup superadmin."
  }
}
```

**Error — `404` Tidak ditemukan**

```json
{
  "status": 404,
  "code": 404,
  "messages": {
    "error": "User 5 tidak ditemukan."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas); `403` — user bukan admin / tidak punya permission yang dibutuhkan (lihat catatan filter di bagian 11).

---

### 9.5 GET /users/{id}/permissions — Daftar Permission Langsung

Permission yang diberikan langsung ke user (tanpa permission dari grup).

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `http://localhost:8080/api/users/{id}/permissions` |
| **Akses** | Admin (permission `users.read`) |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Path Parameter**

| Nama | Tipe | Keterangan |
|---|---|---|
| `id` | integer | ID user |

**Contoh Request**

```bash
curl -X GET "http://localhost:8080/api/users/5/permissions" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>"
```

**Response Sukses — `200 OK`**

```json
{
  "data": [
    "users.create",
    "beta.access"
  ]
}
```

**Error — `404` Tidak ditemukan**

```json
{
  "status": 404,
  "code": 404,
  "messages": {
    "error": "User 5 tidak ditemukan."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas); `403` — user bukan admin / tidak punya permission yang dibutuhkan (lihat catatan filter di bagian 11).

---

### 9.6 PUT /users/{id}/permissions — Sinkronisasi Permission

Mengganti seluruh permission langsung user.

| | |
|---|---|
| **Method** | `PUT` |
| **URL** | `http://localhost:8080/api/users/{id}/permissions` |
| **Akses** | Admin (permission `users.manage-admins`) |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Content-Type` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Path Parameter**

| Nama | Tipe | Keterangan |
|---|---|---|
| `id` | integer | ID user |

**Body (JSON)**

| Field | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `permissions` | array<string> | Ya | Permission yang terdaftar di `AuthGroups::$permissions` |

**Contoh Body**

```json
{
  "permissions": [
    "admin.access",
    "beta.access"
  ]
}
```

**Contoh Request**

```bash
curl -X PUT "http://localhost:8080/api/users/5/permissions" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"permissions": ["admin.access", "beta.access"]}'
```

**Response Sukses — `200 OK`**

```json
{
  "data": [
    "admin.access",
    "beta.access"
  ]
}
```

**Error — `400` Validasi gagal**

```json
{
  "status": 400,
  "code": 400,
  "messages": {
    "permissions": "Permission 'x.y' does not exist."
  }
}
```

**Error — `404` Tidak ditemukan**

```json
{
  "status": 404,
  "code": 404,
  "messages": {
    "error": "User 5 tidak ditemukan."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas); `403` — user bukan admin / tidak punya permission yang dibutuhkan (lihat catatan filter di bagian 11).

---

### 9.7 POST /users/{id}/permissions/check — Cek Permission

Mengecek apakah user memiliki permission (langsung, dari grup, atau wildcard).

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `http://localhost:8080/api/users/{id}/permissions/check` |
| **Akses** | Admin (permission `users.read`) |

**Header**

| Header | Nilai | Wajib |
|---|---|---|
| `Accept` | `application/json` | Ya |
| `Content-Type` | `application/json` | Ya |
| `Authorization` | `Bearer <token>` | Ya |

**Path Parameter**

| Nama | Tipe | Keterangan |
|---|---|---|
| `id` | integer | ID user |

**Body (JSON)**

| Field | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `permission` | string | Ya | Mis. `users.delete` |

**Contoh Body**

```json
{
  "permission": "users.delete"
}
```

**Contoh Request**

```bash
curl -X POST "http://localhost:8080/api/users/5/permissions/check" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"permission": "users.delete"}'
```

**Response Sukses — `200 OK`**

```json
{
  "permission": "users.delete",
  "allowed": true
}
```

**Error — `404` Tidak ditemukan**

```json
{
  "status": 404,
  "code": 404,
  "messages": {
    "error": "User 5 tidak ditemukan."
  }
}
```

**Error — `403` Wajib ganti password**

```json
{
  "status": 403,
  "code": "PASSWORD_RESET_REQUIRED",
  "messages": {
    "error": "Anda harus mengganti password. Gunakan POST /api/auth/change-password."
  }
}
```

**Error umum lainnya:** `401` — token tidak ada, salah, atau sudah kedaluwarsa (body dihasilkan filter Shield dan bisa berbeda dari format di atas); `403` — user bukan admin / tidak punya permission yang dibutuhkan (lihat catatan filter di bagian 11).

---

## 10. Permission yang Dibutuhkan

| Permission | Deskripsi | `superadmin` | `admin` |
|---|---|:---:|:---:|
| `users.read` | Lihat daftar & detail user, grup, permission | ✔ | ✔ |
| `users.create` | Buat user | ✔ | ✔ |
| `users.edit` | Ubah user, aktivasi, force reset | ✔ | ✔ |
| `users.delete` | Hapus user | ✔ | ✔ |
| `users.ban` | Ban / unban | ✔ | ✔ |
| `users.manage-admins` | Kelola grup & permission | ✔ | ✘ |

`superadmin` mendapat semuanya lewat wildcard `users.*`. Endpoint `force-reset-all` mensyaratkan **grup** `superadmin`.

## 11. Catatan Penting

1. **Token mentah hanya tampil sekali** (`token`, `secret_key`). Simpan segera.
2. **Rate limiting:** rute `/auth/*` memakai filter `auth-rates` dan dapat membalas `429`.
3. **Filter Shield:** `tokens` membalas `401` bila token tidak valid. Pada beberapa versi Shield, filter `group` dan `permission` melakukan redirect `302`, bukan JSON `403`. Jika klien menerima `302`, buat filter kustom yang membalas JSON `403` (contoh: `ApiForceResetFilter`). Perilaku ini perlu dicek di versi Shield Anda.
4. **Scope token** tidak dapat diubah setelah token dibuat. `["*"]` berarti semua scope.
5. **Kedaluwarsa vs masa pakai:** `expires_at` adalah tanggal pasti berhentinya token; terpisah dari `unusedTokenLifetime` (default 1 tahun sejak terakhir dipakai).
6. **Soft delete:** `DELETE /users/{id}` tanpa `hard=1` hanya menandai user terhapus.
7. **Format tanggal:** `Y-m-d H:i:s`; untuk input `expires_at`, ISO 8601 (`2026-12-31T23:59`) juga diterima.
8. Nilai pada contoh response (ID, token, tanggal) hanya ilustrasi.
