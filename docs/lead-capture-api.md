# Lead Capture API

Push leads into your CRM from any external source — website contact forms, landing pages, marketing funnels, and more. Once submitted, leads appear instantly in **CRM → Leads**, attributed to your business automatically.

## Overview

The Lead Capture API lets you programmatically create leads for your business. Each business (spa) has its own unique API key. Leads created with your key are automatically attributed to your business, and you will only ever see your own leads — the platform is fully multi-tenant and isolated between businesses.

**Base URL:** `https://app.myspa.co.ke`

| | |
|---|---|
| **Endpoint** | `POST /api/leads` |
| **Content-Type** | `application/json` |
| **Accept** | `application/json` |
| **Rate limit** | 60 requests per minute |

---

## Authentication

Every request must include your business API key, using **one** of the following headers:

| Header | Example | Notes |
|---|---|---|
| `X-Api-Key` | `X-Api-Key: spa_XXXXXXXXXXXXXXXXXXXX` | Recommended |
| `Authorization` | `Authorization: Bearer spa_XXXXXXXXXXXXXXXXXXXX` | Alternative |

Your API key looks like: `spa_XXXXXXXXXXXXXXXXXXXX...`

> ⚠️ **Keep your API key secret.** Anyone who has it can create leads on behalf of your business. Do not expose it in client-side code (e.g. browser JavaScript) — call this API from your server or a form backend instead. If a key is ever compromised, regenerate it and update your integrations immediately.

---

## Endpoint reference

### `POST /api/leads`

Creates a new lead for the business associated with the API key used.

#### Request fields

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string (max 255) | Yes | Full name of the lead. |
| `email` | string (email) | Yes | Lead's email address. Must be unique **within your business** — the same email can exist for a different business, but not twice for yours. |
| `source` | string | Yes | Where the lead came from, e.g. `website`, `referral`, `advertisement`, `cold call`, `walk-in`. |
| `phone` | string (max 20) | No | Lead's phone number. |
| `status` | string enum | No | One of `new`, `converted`, `lost`. Defaults to `new`. |
| `notes` | string | No | Free-text notes about the lead. |
| `service` | string (max 255) | No | Service the lead is interested in. |
| `booking_amount` | number (≥ 0) | No | Amount paid at booking, if any. |
| `booking_date` | date | No | Date of the booking, if any. |
| `assigned_to` | integer | No | User ID of the salesperson to assign this lead to. Must be a valid user in your business. |

---

## Example request

```bash
curl -X POST "https://app.myspa.co.ke/api/leads" \
  -H "X-Api-Key: spa_your_key_here" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Jane Doe",
    "email": "jane@example.com",
    "phone": "+254700000000",
    "source": "website",
    "service": "Facial",
    "booking_amount": 2500
  }'
```

---

## Responses

### `201 Created` — Success

The lead was created successfully.

```json
{
  "message": "Lead created successfully.",
  "data": {
    "id": 482,
    "business_id": 17,
    "name": "Jane Doe",
    "email": "jane@example.com",
    "phone": "+254700000000",
    "source": "website",
    "status": "new",
    "notes": null,
    "service": "Facial",
    "booking_amount": "2500.00",
    "booking_date": null,
    "assigned_to": null,
    "created_at": "2026-07-02T09:14:32.000000Z",
    "updated_at": "2026-07-02T09:14:32.000000Z"
  }
}
```

### `401 Unauthorized` — Missing or invalid API key

```json
{
  "message": "Invalid or missing API key."
}
```

**Common causes:** the `X-Api-Key` / `Authorization` header is missing, misspelled, or contains an incorrect key.

### `403 Forbidden` — Business account inactive

```json
{
  "message": "This business account is inactive."
}
```

**Common causes:** the business's subscription or account has been deactivated. Contact support to reactivate.

### `422 Unprocessable Entity` — Validation failed

Returned when required fields are missing/invalid, or the email already exists for this business.

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "email": [
      "The email has already been taken."
    ],
    "source": [
      "The source field is required."
    ]
  }
}
```

### `429 Too Many Requests` — Rate limit exceeded

```json
{
  "message": "Too Many Attempts."
}
```

You are limited to **60 requests per minute** per API key. Space out requests or implement retry-with-backoff if you expect to exceed this.

---

## How to get your API key

No coding experience needed — here's how to find your key in the app:

1. **Log in** to your account.
2. Go to **CRM → Leads** in the left-hand navigation.
3. Click the **"API Access"** button in the top-right corner, next to **"Add New Lead."**
4. A dialog will open showing:
   - Your unique **endpoint URL**
   - Your **business API key** (hidden by default — click the 👁 eye icon to reveal it, or the copy icon to copy it directly)
   - A ready-to-use **cURL example** you can hand to a developer
   - Notes on required fields and rate limits

That's it — copy your key and endpoint into your form/integration and you're ready to start sending leads.

> 🔒 Treat this key like a password. Anyone with it can add leads to your account. If you ever suspect it's been exposed, contact support to have it regenerated.
