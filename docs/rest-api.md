# Spa Management REST API

Token-based REST API for the Myspa Spa Management ERP. Use it to build mobile apps, integrations, and internal tools on top of your spa's data — authentication, business switching, customers, and consultation records.


---

## Table of contents

- [Quick start](#quick-start)
- [Base URL](#base-url)
- [Conventions](#conventions)
  - [Response envelope](#response-envelope)
  - [Multi-tenancy](#multi-tenancy)
  - [Pagination](#pagination)
- [Errors](#errors)
- [Rate limiting](#rate-limiting)
- [Authentication](#1-authentication)
  - [Login](#11-login--post-apiauthlogin)
  - [Current user](#12-current-user--get-apiauthme-)
  - [Logout](#13-logout--post-apiauthlogout-)
- [Businesses](#2-businesses)
  - [List authorized businesses](#21-list-authorized-businesses--get-apibusinesses-)
  - [Switch active business](#22-switch-active-business--post-apibusinessesbusinessidswitch-)
- [Customers](#3-customers)
  - [List customers](#31-list-customers--get-apicustomers-)
  - [Get one customer](#32-get-one-customer--get-apicustomersid-)
  - [All consultations, grouped](#33-all-consultations-grouped--get-apicustomersidconsultations-)
  - [Consultations by type, paginated](#34-consultations-by-type-paginated--get-apicustomersidconsultationstype-)
- [Typical client flow](#4-typical-client-flow)
- [Endpoint summary](#5-endpoint-summary)
- [Security best practices](#6-security-best-practices)

---

## Quick start

```bash
# 1. Log in and grab a token
curl -X POST https://<your-domain>/api/auth/login \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"email": "jane@spa.com", "password": "secret", "device_name": "my-app"}'

# 2. Call any protected endpoint with the token
curl https://<your-domain>/api/customers \
  -H "Authorization: Bearer <token>" \
  -H "Accept: application/json"
```

That's the whole loop: **login → token → Bearer header on every request**. Read on for the details.

---

## Base URL

```
https://<your-domain>/api
```

Replace `<your-domain>` with your deployment's domain. All paths in this document are relative to the domain root (they already include the `/api` prefix).

---

## Conventions

- All requests and responses are JSON. Always send both headers:

  ```
  Accept: application/json
  Content-Type: application/json
  ```

- All protected endpoints (marked 🔒) require a Bearer token:

  ```
  Authorization: Bearer <token>
  ```

### Response envelope

Every response follows this envelope:

```json
{
  "success": true,
  "message": "Optional human-readable message",
  "data": { }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `success` | boolean | `true` on success, `false` on failure |
| `message` | string | Optional human-readable message |
| `data` | object/array | The payload (a resource, list, or paginator) |

### Multi-tenancy

Every user belongs to one or more businesses, with exactly one **active** business at a time. All data endpoints (customers, consultations, etc.) are automatically scoped to the token user's **active business**.

>  **Isolation guarantee:** a resource belonging to another business returns `404` — the API never reveals whether the resource exists. Data never leaks across businesses.

To work with a different business, use the [switch endpoint](#22-switch-active-business--post-apibusinessesbusinessidswitch-).

### Pagination

List endpoints return a standard **Laravel paginator** inside `data`:

```json
{
  "success": true,
  "data": {
    "current_page": 1,
    "data": [ ],
    "per_page": 20,
    "total": 57,
    "last_page": 3,
    "next_page_url": "https://<your-domain>/api/customers?page=2",
    "prev_page_url": null
  }
}
```

Control it with query parameters:

| Param | Type | Notes |
|-------|------|-------|
| `per_page` | integer | 1–100, default 20 |
| `page` | integer | Page number, starting at 1 |

To iterate through all results, keep requesting `page + 1` until `next_page_url` is `null`.

---

## Errors

### Common HTTP status codes

| Code | Meaning |
|------|---------|
| `200` | Success |
| `401` | Missing or invalid token |
| `403` | Authenticated but not allowed (e.g. switching to a business you don't belong to) |
| `404` | Resource not found — including resources owned by another business |
| `422` | Validation error (bad input, wrong credentials, invalid consultation type) |
| `429` | Rate limit exceeded (login is limited to 10 attempts/minute) |

### Validation errors (`422`)

Validation errors use Laravel's standard shape — a top-level `message` plus a per-field `errors` map:

```json
{
  "message": "The provided credentials are incorrect.",
  "errors": {
    "email": ["The provided credentials are incorrect."]
  }
}
```

### Handling errors in your client

- Treat any non-2xx status as a failure; don't rely on `success` alone.
- On `401`, your token is missing, malformed, or revoked — re-authenticate.
- On `422`, show the messages in `errors` to the user; they are safe, human-readable strings.
- On `429`, back off and retry after a delay (see [Rate limiting](#rate-limiting)).

---

## Rate limiting

| Endpoint | Limit |
|----------|-------|
| `POST /api/auth/login` | **10 requests per minute** |

Exceeding the limit returns `429 Too Many Requests`. Implement retry-with-backoff rather than hammering the endpoint — and cache your token instead of logging in on every request.

---

## 1. Authentication

### 1.1 Login — `POST /api/auth/login`

Exchange email + password for a Bearer token. **Rate limited: 10 requests per minute.**

**Body**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `email` | string | yes | |
| `password` | string | yes | |
| `device_name` | string | no | Label for the token (e.g. `"mobile-app"`). Defaults to `"api"`. |

**Request**

```bash
curl -X POST https://<your-domain>/api/auth/login \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"email": "jane@spa.com", "password": "secret", "device_name": "mobile-app"}'
```

**Response `200`**

```json
{
  "success": true,
  "message": "Login successful.",
  "data": {
    "token": "1|dGhpcyBpcyBhIHNhbXBsZSB0b2tlbg...",
    "token_type": "Bearer",
    "user": {
      "id": 1,
      "name": "Jane Wanjiku",
      "email": "jane@spa.com",
      "user_type": "user"
    },
    "active_business": {
      "id": 3,
      "name": "Nairobi Spa",
      "email": "info@nairobispa.com",
      "phone": "+254712345678",
      "country": "Kenya",
      "currency": "KES",
      "is_active": true
    },
    "businesses": [
      { "id": 3, "name": "Nairobi Spa", "is_active": true, "...": "..." },
      { "id": 7, "name": "Mombasa Branch", "is_active": false, "...": "..." }
    ]
  }
}
```

| Field | Description |
|-------|-------------|
| `token` | The Bearer token — store it securely and send it on every protected request |
| `active_business` | The business currently selected for this user (`null` if none) |
| `businesses` | Every business the user is authorized to access |

**Errors**

- `422` — wrong credentials, or account status is not active (`"Your account is not active. Please contact support."`).
- `429` — too many attempts.

> ℹ️ Tokens do not expire by default. Store them securely and revoke them with the [logout endpoint](#13-logout--post-apiauthlogout-) when done. Use a distinct `device_name` per app/device so tokens are easy to identify.

### 1.2 Current user — `GET /api/auth/me` 

Returns the authenticated user and their active business. Useful for restoring app state on launch and verifying a stored token is still valid.

```bash
curl https://<your-domain>/api/auth/me \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"
```

**Response `200`**

```json
{
  "success": true,
  "data": {
    "id": 1,
    "name": "Jane Wanjiku",
    "email": "jane@spa.com",
    "user_type": "user",
    "status": "active",
    "active_business": { "id": 3, "name": "Nairobi Spa", "is_active": true, "...": "..." }
  }
}
```

### 1.3 Logout — `POST /api/auth/logout` 

Revokes **the token used on this request**. Other tokens/devices stay logged in.

```bash
curl -X POST https://<your-domain>/api/auth/logout \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"
```

**Response `200`**

```json
{ "success": true, "message": "Logged out successfully." }
```

---

## 2. Businesses

### 2.1 List authorized businesses — `GET /api/businesses` 

All businesses the token user may access. Exactly one has `is_active: true`.

```bash
curl https://<your-domain>/api/businesses \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"
```

**Response `200`**

```json
{
  "success": true,
  "data": [
    {
      "id": 3,
      "name": "Nairobi Spa",
      "email": "info@nairobispa.com",
      "phone": "+254712345678",
      "country": "Kenya",
      "currency": "KES",
      "is_active": true
    },
    {
      "id": 7,
      "name": "Mombasa Branch",
      "email": "mombasa@nairobispa.com",
      "phone": "+254798765432",
      "country": "Kenya",
      "currency": "KES",
      "is_active": false
    }
  ]
}
```

### 2.2 Switch active business — `POST /api/businesses/{businessId}/switch` 

Makes `{businessId}` the user's active business.

>  **The switch is account-wide.** All subsequent API calls — with **any** of the user's tokens — are scoped to the new business. It also affects the user's web session, matching the behavior of switching business in the web UI. If you run parallel jobs against different businesses, be aware they share this single active-business state.

```bash
curl -X POST https://<your-domain>/api/businesses/7/switch \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"
```

**Response `200`**

```json
{
  "success": true,
  "message": "Switched to Mombasa Branch.",
  "data": {
    "active_business": {
      "id": 7,
      "name": "Mombasa Branch",
      "is_active": true,
      "...": "..."
    }
  }
}
```

**Errors**

- `403` — the user is not a member of that business. Nothing is changed; the current business stays active.

```json
{ "success": false, "message": "You are not authorized to access this business." }
```

---

## 3. Customers

All customer endpoints are scoped to the **active business**. A customer ID belonging to another business returns `404`.

### 3.1 List customers — `GET /api/customers` 

Paginated, searchable customer list.

**Query parameters**

| Param | Type | Notes |
|-------|------|-------|
| `search` | string | Matches name, email, phone, etc. |
| `status` | string | One of `active`, `inactive`, `dormant` |
| `per_page` | integer | 1–100, default 20 |
| `page` | integer | Page number |

```bash
curl "https://<your-domain>/api/customers?search=jane&per_page=10" \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"
```

**Response `200`** — `data` is a standard [Laravel paginator](#pagination):

```json
{
  "success": true,
  "data": {
    "current_page": 1,
    "data": [
      {
        "id": 12,
        "first_name": "Jane",
        "last_name": "Doe",
        "full_name": "Jane Doe",
        "email": "jane@example.com",
        "phone": "0712345678",
        "status": "active",
        "age": 29,
        "avatar_url": "https://...",
        "waxing_consultations_count": 2,
        "therapeutic_massage_consultations_count": 0,
        "skin_consultations_count": 1,
        "nail_consultations_count": 0,
        "laser_consultations_count": 3
      }
    ],
    "per_page": 10,
    "total": 1,
    "last_page": 1,
    "next_page_url": null,
    "prev_page_url": null
  }
}
```

### 3.2 Get one customer — `GET /api/customers/{id}` 

Returns the full customer record, the assigned therapist (`id`, `name`, `email`), and a count per consultation type.

```bash
curl https://<your-domain>/api/customers/12 \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"
```

**Errors:** `404` if the customer doesn't exist **or belongs to another business**.

### 3.3 All consultations, grouped — `GET /api/customers/{id}/consultations` 

Every consultation document for the customer, grouped by type, newest first (by `consultation_date`). Each consultation includes its therapist (`id`, `name`).

Best for a customer profile screen showing everything at once. For customers with many documents of one type, prefer [3.4](#34-consultations-by-type-paginated--get-apicustomersidconsultationstype-).

```bash
curl https://<your-domain>/api/customers/12/consultations \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"
```

**Response `200`**

```json
{
  "success": true,
  "data": {
    "customer": {
      "id": 12,
      "full_name": "Jane Doe",
      "email": "jane@example.com",
      "phone": "0712345678"
    },
    "consultations": {
      "waxing": [ { "id": 5, "consultation_date": "2026-06-30", "therapist": { "id": 2, "name": "Mary" }, "...": "..." } ],
      "therapeutic-massage": [],
      "skin": [],
      "nail": [],
      "laser": []
    }
  }
}
```

### 3.4 Consultations by type, paginated — `GET /api/customers/{id}/consultations/{type}` 

Paginated consultation documents of a single type. Use this instead of [3.3](#33-all-consultations-grouped--get-apicustomersidconsultations-) when a customer has many documents of one type.

**`{type}` must be one of:** `waxing`, `therapeutic-massage`, `skin`, `nail`, `laser`

**Query parameters:** `per_page` (1–100, default 20), `page`

```bash
curl "https://<your-domain>/api/customers/12/consultations/waxing?per_page=10" \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"
```

**Response `200`** — `data` is a [Laravel paginator](#pagination) of consultation documents (each with its `therapist`).

**Errors**

- `422` — invalid type:

```json
{
  "success": false,
  "message": "Invalid consultation type. Valid types: waxing, therapeutic-massage, skin, nail, laser"
}
```

---

## 4. Typical client flow

```text
1. POST /api/auth/login                         → store data.token
2. Read data.active_business + data.businesses from the login response
3. (optional) POST /api/businesses/{id}/switch  → change the working business
4. GET /api/customers                           → list customers of the active business
5. GET /api/customers/{id}/consultations        → pull a customer's documents
6. POST /api/auth/logout                        → revoke the token when done
```

On subsequent app launches, skip step 1 if you have a stored token — call `GET /api/auth/me` to validate it and restore state; re-login only if it returns `401`.

---

## 5. Endpoint summary

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/auth/login` | — | Get a Bearer token (throttled 10/min) |
| GET | `/api/auth/me` | 🔒 | Current user + active business |
| POST | `/api/auth/logout` | 🔒 | Revoke the current token |
| GET | `/api/businesses` | 🔒 | Businesses the user can access |
| POST | `/api/businesses/{id}/switch` | 🔒 | Change the active business |
| GET | `/api/customers` | 🔒 | Paginated, searchable customer list |
| GET | `/api/customers/{id}` | 🔒 | Single customer + consultation counts |
| GET | `/api/customers/{id}/consultations` | 🔒 | All consultation documents, grouped by type |
| GET | `/api/customers/{id}/consultations/{type}` | 🔒 | Paginated documents of one type |

🔒 = requires `Authorization: Bearer <token>`

---

## 6. Security best practices

- **Store tokens securely** — use the platform keychain/keystore on mobile; never commit tokens to source control or log them.
- **Never embed credentials in client-side web code.** Browser JavaScript can't keep a secret; proxy through your own backend if needed.
- **Use one token per device/app** via `device_name`, so a compromised device can be revoked without logging everyone out.
- **Revoke tokens you no longer need** with `POST /api/auth/logout` — tokens do not expire by default.
- **Handle `401` by re-authenticating**, not by retrying the same token.

---

## Support

Questions about API access? Contact support at [support@myspa.co.ke](mailto:support@myspa.co.ke).
