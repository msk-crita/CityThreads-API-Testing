# End-to-End Test Guide

Manual API walkthrough for City Threads Backend. Use Insomnia, curl, or Swagger after login.

Project Markdown is only [README.md](README.md) (setup and architecture) and this file (how to test APIs). Do not add other `.md` docs; put new material in one of these two.

**Base URL:** `http://localhost:8080`  
**Auth:** `Authorization: Bearer <access_token>` on protected routes  
**Envelope:** success and error bodies are `ApiResponse`: `success`, `data`, `notification.exceptionMessage`, `statusCode`. Validation errors are `400`. The message is in `notification.exceptionMessage`.  
**Clients:** `client_type` is an `enum_values` code (`type_category = client_type`). Seeded values: `mobile` (description `body`, refresh in JSON) and `browser` (description `cookie`, refresh cookie + CSRF on writes). Sending `MOBILE` or `BROWSER` still matches.

When you add or change an API, update this file in the same change. See [Keeping this guide current](#keeping-this-guide-current).

### Contents

0. [Prerequisites](#0-prerequisites)  
1. [Auth](#1-auth-required-before-admin-apis)  
2. [Users admin](#2-users-admin)  
3. [Products](#3-product-masters--sku-happy-path)  
4. [Warehouses](#4-warehouses--areas--racks--bins)  
4b. [Contacts](#4b-contacts)  
4c. [Inventory](#4c-inventory)  
4e. [Order to Dispatch](#4e-order-to-dispatch)  
5. [Permission matrix — products](#5-permission-matrix-products)  
6. [Permission matrix — warehouses](#6-permission-matrix-warehouses)  
6b. [Permission matrix — contacts](#6b-permission-matrix-contacts)  
6c. [Permission matrix — inventory](#6c-permission-matrix-inventory)  
6e. [Permission matrix — Order to Dispatch](#6e-permission-matrix--order-to-dispatch)  
7. [Not implemented yet](#7-not-implemented-yet-skip--expect-absence)  
8. [Automated tests](#8-automated-tests-complement-not-a-substitute)

---

## 0. Prerequisites

1. PostgreSQL database created; `secrets-local.json` filled from `secrets-example.json`.
2. App starts cleanly: `./mvnw spring-boot:run` (Flyway V1–V28 applied).
3. First admin: set `bootstrap.enabled` true with username/email/password, start once, then set it back to false.
4. SMTP works for MFA OTP (or read OTP from mail/logs if using a local catcher).

---

## 1. Auth (required before admin APIs)

### 1.1 Login

```http
POST /api/auth/login
Content-Type: application/json

{
  "login": "<username-or-email>",
  "password": "<password>",
  "client_type": "mobile",
  "device_label": "e2e"
}
```

| Expect | Notes |
|--------|--------|
| `status: MFA_REQUIRED` | First login / MFA on → capture `challenge_token` |
| `status: AUTHENTICATED` | Capture `access_token` (and `refresh_token` when the client type's description is `body`). `role_code`, `platform`, and `permissions` are included. |

Use `mobile` for curl and Swagger. `browser` stores the refresh token in a cookie. Login, MFA, password reset, and invitation accept do not require a CSRF header, even if an old refresh cookie is still in the browser. A request that already sends `Authorization: Bearer` does not need the CSRF header. Cookie-only writes (`POST /api/auth/refresh`, `POST /api/auth/logout`, `POST /api/auth/logout-all`, `DELETE /api/auth/sessions/{id}`) still require the CSRF header when that cookie is sent.

`permissions` lists only granted actions. A module action is `true`. A code that names a resource inside the module is nested. `READ` is returned as `view`. `MFA_REQUIRED` does not include permissions.

```json
"permissions": {
  "users": { "create": true, "view": true, "update": true },
  "warehouses": { "view": true, "location": { "create": true } },
  "products": { "view": true, "product_info": { "view": true, "create": true } }
}
```

### 1.2 MFA verify (when required)

```http
POST /api/auth/mfa/verify
Content-Type: application/json

{
  "challenge_token": "<from-login>",
  "otp": "<6-digit-code>",
  "client_type": "mobile",
  "device_label": "e2e"
}
```

Expect `AUTHENTICATED` + `access_token`.

### 1.3 Profile

```http
GET /api/auth/me
Authorization: Bearer <access_token>
```

Expect `200` and the signed-in user only. Fields are `id`, `username`, `first_name`, `last_name`, `email`, `phone`, `role_id`, `role_name`, `role_code`, `job_title`, `status`, `created_at`, `last_signed_in`, `responsibility`, and `sessions`. Permissions are returned on login, not on this profile. `responsibility.facility` is the primary assigned warehouse, or the lowest warehouse id when none is primary. With a warehouse assignment, `total_areas` and `total_bins` count every staging zone and storage bin in that warehouse, and `total_lots` counts distinct lots with quantity there. With only location assignments, those counts stay inside the assigned locations of that facility. No assignment leaves `facility` null and the counts at `0`. `sessions` is every unrevoked, unexpired refresh session for this user, newest first. Each item has `id`, `status` (`active`), `client_type`, `device_label`, `ip_address`, `current`, `signed_in_at`, `expires_at`, and `expires_in`. `current` is true for the session that sent the refresh cookie. A bearer-only client has `current` false on every row. An empty list means no live refresh session.

`phone` and `job_title` are optional on `POST` and `PATCH /api/admin/users`. The profile page does not update them.

### 1.4 Refresh / logout (optional)

```http
POST /api/auth/refresh
Content-Type: application/json

{ "refresh_token": "<refresh_token>" }
```

```http
POST /api/auth/logout
Authorization: Bearer <access_token>
```

Expect `200` and `notification.exceptionMessage` `Successfully signed out.` This revokes only the refresh session that sent the cookie or `refresh_token`. A cookie-only browser call must send the CSRF header. A bearer token does not.

Sign out one device from `sessions` on `GET /api/auth/me`. Use that row's `id`:

```http
DELETE /api/auth/sessions/{id}
Authorization: Bearer <access_token>
```

Expect `200` and `notification.exceptionMessage` `Session revoked.` Another user's id returns `403`. When the id is the current browser session, the refresh cookie is cleared. A cookie-only call must send the CSRF header. A bearer token does not.

Sign out every device:

```http
POST /api/auth/logout-all
Authorization: Bearer <access_token>
```

Expect `200` and `notification.exceptionMessage` `Signed out of every device.` Every refresh session for this user is revoked and the refresh cookie is cleared. A cookie-only call must send the CSRF header. A bearer token does not. An access token already issued on another device stays valid until it expires; that device cannot refresh it.

### 1.5 Broader auth scenarios

- [ ] Bootstrap first `SUPER_ADMIN` from secrets, then disable bootstrap
- [ ] Wrong password → 401, no hint whether login or password failed
- [ ] MFA resend respects cooldown; wrong OTP eventually locks the challenge
- [ ] Refresh rotates; replaying the old refresh revokes the family
- [ ] Logout invalidates refresh; further refresh fails
- [ ] `GET /api/auth/sessions` lists own sessions; `DELETE /api/auth/sessions/{id}` revokes one; `POST /api/auth/logout-all` revokes every device
- [ ] Invite user (`POST /api/admin/users/invitations`), accept (`POST /api/auth/invitations/accept`), login
- [ ] Self-service password reset request + confirm; expired/replayed token fails
- [ ] Unmapped `/api/**` with valid token → 403
- [ ] No token on protected route → 401
- [ ] Health is public: `GET /api/v1/health` and `/api/v1/health/info`
- [ ] `ADMIN` cannot change route mappings or grant `SUPER_ADMIN`
- [ ] `SUPER_ADMIN` can create a permission, map a route, assign it, and see access change from deny to allow

---

## 2. Users admin

Requires `USER_READ` / `USER_CREATE` / `USER_UPDATE` / `USER_PASSWORD_RESET` (and `ROLE_READ` for roles) as mapped in the DB.

```http
GET /api/admin/users?search=&page=0&size=20
Authorization: Bearer <token>

GET /api/admin/users/roles
```

```http
POST /api/admin/users
Authorization: Bearer <token>
Content-Type: application/json

{
  "username": "e2e.user",
  "email": "e2e.user@example.com",
  "first_name": "E2E",
  "last_name": "User",
  "role_id": <executive_or_admin_role_id>,
  "password": "A-strong-password-12+",
  "status": "active",
  "phone": "+91 98250 41120",
  "job_title": "Warehouse Director"
}
```

Expect `201` `ApiResponse`. `data` is the created user.

```http
PATCH /api/admin/users/{id}
Content-Type: application/json

{ "first_name": "Updated", "job_title": "Picker" }
```

```http
POST /api/admin/users/{id}/password-reset
Content-Type: application/json

{ "password": "Another-strong-password-12+" }
```

### Users checklist

- [ ] Duplicate username or email → 400
- [ ] Password shorter than policy minimum → 400
- [ ] Cannot deactivate/demote the last protected admin account
- [ ] List search matches username/email/name

---

## 3. Product masters → SKU (happy path)

Use the same Bearer token for all steps. Save IDs from each create response.

### 3.1 Product info (one of each type)

Types come from `enum_values` (`product_info_type`): `ply`, `article`, `tkt`, `range`, `tex`.  
API field is still `type` (code); DB stores `type_id`.

```http
GET /api/admin/product-info/types
Authorization: Bearer <token>
```

Expect `200`. `data` is every `product_info_type` row, including inactive, ordered by `sort_order`. Each item has `id`, `code`, `display_label`, `sort_order`, and `is_active`. Use `code` as the `type` value on create and list.

```http
POST /api/admin/product-info
Authorization: Bearer <token>
Content-Type: application/json

{ "code": "2PLY", "name": "2 Ply", "type": "ply", "status": "active" }
```

Repeat for:

| type | example code | example name |
|------|--------------|--------------|
| `article` | `ART-100` | Article 100 |
| `tkt` | `TKT-40` | TKT 40 |
| `range` | `RNG-A` | Range A |
| `tex` | `TEX-27` | TEX 27 |

Expect `201` `ApiResponse`. `data` is the created product-info row.

```http
GET /api/admin/product-info?type=ply
GET /api/admin/product-info/{id}
PATCH /api/admin/product-info/{id}
Content-Type: application/json

{ "name": "2 Ply Updated" }
```

### 3.2 Shade

```http
POST /api/admin/shades
Authorization: Bearer <token>
Content-Type: application/json

{ "code": "BK01", "name": "Black", "status": "active" }
```

```http
GET /api/admin/shades
PATCH /api/admin/shades/{id}
```

### 3.3 Unit

```http
POST /api/admin/units
Authorization: Bearer <token>
Content-Type: application/json

{ "code": "CONE", "name": "Cone", "decimal_places": 0, "status": "active" }
```

```http
GET /api/admin/units
PATCH /api/admin/units/{id}
```

### 3.4 Create product

Use IDs from 3.1–3.3. `ticket` column maps to product_info type `tkt`.

```http
POST /api/admin/products
Authorization: Bearer <token>
Content-Type: application/json

{
  "sku": "SKU-E2E-001",
  "name": "E2E Thread SKU",
  "HSN_code": "5401",
  "range": <range_id>,
  "article": <article_id>,
  "ply": <ply_id>,
  "tex": <tex_id>,
  "ticket": <tkt_id>,
  "length": 5000,
  "shade_id": <shade_id>,
  "stock_unit_id": <unit_id>,
  "rate_per_unit": 12.50,
  "reorder_level": 10,
  "gst": 18,
  "reorder_quantity": 50,
  "requires_lot_tracking": true,
  "status": "active"
}
```

Expect `201` `ApiResponse`. `data` is the created product (`id`, `sku`, and the other saved fields).

### 3.5 List / get / update / status

```http
GET /api/admin/products?page=1&page_size=20
GET /api/admin/products?search=SKU-E2E&status=active
GET /api/admin/products/{id}
```

List items include `quantity`, `reserved_quantity`, and `available_quantity`. Those are sums across `inventory_balances` for the product, and `0` when the product has no stock. Detail `inventory` lists each balance (`lot_number`, `location_code`, `warehouse_code`, quantity, reserved, available). Detail `movements` is the latest 50 ledger lines for the product (`type` is the transaction type code, `reference` is `stock_adjustment` or `stock_count` plus the document number when that is the source, `from_location` and `to_location` are location codes). Until a balance or a posted movement exists, inventory is `[]` and movements is `[]`. The dedicated ledger is still `GET /api/admin/inventory/transactions?product_id=`.

```http
PUT /api/admin/products/{id}
Content-Type: application/json

{ "name": "E2E Thread SKU Updated", "rate_per_unit": 13.00 }
```

```http
PATCH /api/admin/products/{id}/status
Content-Type: application/json

{ "status": "inactive" }
```

Then set back to `active` if needed. Allowed product statuses: `active`, `inactive`, `discontinued`.

### 3.6 Product barcodes

One primary barcode per product. `ean` must be 8 or 13 digits. `upc` must be 8 or 12 digits. `internal`, `qr`, `supplier`, and `other` are any unique non-blank string up to 100 characters. Type defaults to `internal`.

```http
POST /api/admin/products/{id}/barcodes
Authorization: Bearer <token>
Content-Type: application/json

{
  "barcode": "8901234567890",
  "barcode_type": "ean",
  "is_primary": true
}
```

Expect `201` `ApiResponse`. `data` is the created barcode.

```http
GET /api/admin/products/{id}/barcodes
PATCH /api/admin/products/{id}/barcodes/{barcodeId}
DELETE /api/admin/products/{id}/barcodes/{barcodeId}
```

`GET /api/admin/products/{id}` includes the same barcode list. A second barcode with `is_primary: true` clears the previous primary. Duplicate barcode (any case) → 400. Delete is a hard delete and returns `ApiResponse` with a null `data`.

### 3.7 Product suppliers

`supplier_id` must be an active contact whose type is `supplier`. Several links on one product may be primary. `status` uses `common_status` and defaults to `active`.

```http
POST /api/admin/products/{id}/suppliers
Authorization: Bearer <token>
Content-Type: application/json

{
  "supplier_id": <supplier_contact_id>,
  "supplier_sku": "SR-3PLY-03183",
  "purchase_unit_id": <unit_id>,
  "purchase_rate": 10.50,
  "lead_time_days": 15,
  "is_primary": true,
  "status": "active"
}
```

Expect `201` `ApiResponse`. `data` is the supplier link, including `purchase_unit` `{id, code, name}`, `lead_time_days` `15`, and `status` `active`.

```http
GET /api/admin/products/{id}/suppliers
PATCH /api/admin/products/{id}/suppliers/{linkId}
DELETE /api/admin/products/{id}/suppliers/{linkId}
```

Soft-deactivate with `PATCH` `{ "status": "inactive" }`. Hard `DELETE` is `SUPER_ADMIN` only (`403` otherwise). The same supplier twice on one product → 400. A customer id, or an inactive supplier, → 400. Product detail includes this list. The supplier contact's `product_count` counts every link, including inactive ones.

### 3.8 Unit conversions

`product_id` null is a global conversion and applies when the list is filtered by a product as well as when it is omitted. Factor must be greater than 0 and at most 6 decimal places. From and to units must be different active units. The same product (or the same global pair) cannot be stored twice. Validation errors are HTTP 400 `ApiResponse`. Omitting a field on PATCH leaves it unchanged; sending `product_id` null does not clear an existing product link.

```http
POST /api/admin/unit-conversions
Authorization: Bearer <token>
Content-Type: application/json

{
  "product_id": null,
  "from_unit_id": <unit_id>,
  "to_unit_id": <other_unit_id>,
  "conversion_factor": 1000
}
```

Expect `201` `ApiResponse` with the saved row.

```http
GET /api/admin/unit-conversions
GET /api/admin/unit-conversions?product_id={id}
PATCH /api/admin/unit-conversions/{id}
DELETE /api/admin/unit-conversions/{id}
```

PATCH and DELETE expect `200`. DELETE hard-deletes the row. Duplicate from/to for the same product (or both global) → 400.

### Masters / product checklist

- [ ] Duplicate `product_info` code **same type** → 400
- [ ] Same code **different type** → allowed
- [ ] Invalid `type` (not in `product_info_type`) → 400
- [ ] Product with wrong attribute type (e.g. ply id in `range`) → 400 on that field
- [ ] Product with inactive master → 400
- [ ] Duplicate SKU → 400
- [ ] List pagination is **1-based** (`page`, `page_size`)
- [ ] After product uses a master, that master’s `in_use` > 0
- [ ] EAN that is not 8 or 13 digits, or UPC that is not 8 or 12 digits → 400 `barcode`
- [ ] Second primary barcode clears the first
- [ ] Two primary supplier links on one product are both allowed
- [ ] Product list quantities are `0` before any balance exists, and match `inventory_balances` after one does
- [ ] Product detail `inventory` and `movements` follow the balances and the latest ledger lines
- [ ] Duplicate unit conversion (same product or both global, same from/to) → 400 `from_unit_id`

---

## 4. Warehouses → areas → racks → bins

Use the same Bearer token. Save warehouse / area / rack / bin IDs from responses.

**Rules to remember while testing**

- Set `manager_id` (active user) before creating any location under the warehouse.
- `capacity_unit` is a `units.code` (`sq.ft`, `sq.m`, `pallets`, `bins`, `cbm`).
- Location codes unique **per warehouse**; warehouse codes unique globally.
- Soft-deactivate via status; hard `DELETE` only as `SUPER_ADMIN`.

### 4.1 Create / list / update warehouse

```http
POST /api/admin/warehouses
Authorization: Bearer <token>
Content-Type: application/json

{
  "code": "CT-WH-01",
  "name": "Main Warehouse",
  "address": "Plot 18",
  "city": "Surat",
  "state": "Gujarat",
  "pincode": "395003",
  "manager_id": <active_user_id>,
  "total_capacity": 11000,
  "capacity_unit": "sq.ft",
  "is_primary": true,
  "status": "active"
}
```

Expect `201` `ApiResponse`. `data` is the created warehouse. The list, tree, summary, capacity, and put-away reads use the same envelope, with the previous body in `data`.

```http
GET /api/admin/warehouses
Authorization: Bearer <token>

PATCH /api/admin/warehouses
Authorization: Bearer <token>
Content-Type: application/json

{ "id": <warehouse_id>, "name": "Main Warehouse Updated", "total_capacity": 12000 }
```

Expect PATCH `200` with `warehouse` + `capacity` summary. Soft-deactivate: `{ "id": ..., "status": "inactive" }`.

### 4.2 Area → rack → bins

```http
POST /api/admin/warehouses/areas
Authorization: Bearer <token>
Content-Type: application/json

{
  "warehouse_id": <warehouse_id>,
  "code": "AREA-A",
  "name": "Finished goods",
  "purpose": "Picking and dispatch",
  "capacity": 4000,
  "status": "active"
}
```

```http
POST /api/admin/warehouses/racks
Authorization: Bearer <token>
Content-Type: application/json

{
  "area_id": <area_id>,
  "code": "A01",
  "name": "Rack A01",
  "capacity": 1400,
  "status": "active"
}
```

```http
POST /api/admin/warehouses/bins/batch
Authorization: Bearer <token>
Content-Type: application/json

{
  "rack_id": <rack_id>,
  "shelf_id": "S1",
  "bin_count": 3,
  "capacity": 50,
  "stock_capacity": 100,
  "status": "active"
}
```

Expect batch `201` `ApiResponse`. `data` has `shelfId` `S1`, `binIds`, and `codes` such as `A01-S1-B1`. Area, rack, and bin creates put `{ "id": ... }` in `data`.

Optional single bin:

```http
POST /api/admin/warehouses/bins
Authorization: Bearer <token>
Content-Type: application/json

{
  "rack_id": <rack_id>,
  "name": "S1 B4",
  "code": "A01-S1-B4",
  "capacity": 50,
  "stock_capacity": 100,
  "shelf": "S1",
  "bin": "B4",
  "status": "active"
}
```

Updates / soft status:

```http
PATCH /api/admin/warehouses/areas/{id}
{ "name": "Finished goods updated" }

PATCH /api/admin/warehouses/racks/{id}
{ "status": "inactive" }

PATCH /api/admin/warehouses/bins/{id}
{ "status": "blocked" }
```

### 4.3 Tree / KPIs / put-away

```http
GET /api/admin/warehouses/tree?warehouse=<warehouse_id>
GET /api/admin/warehouses/summary?warehouse=<warehouse_id>
GET /api/admin/warehouses/capacity?warehouse=<warehouse_id>
GET /api/admin/warehouses/putaway-bins?warehouse=<warehouse_id>
```

Bin `state` is derived. `quantity` is the sum of `inventory_balances.quantity` at that bin (`0` when none). Empty active bins are `available`. A bin at or above 99% of `stock_capacity`, or with quantity and no stock capacity, is `occupied`. `product_name` and `stock_code` on the tree come from the largest positive balance in the bin (`stock_code` is the lot number, or the SKU when the lot is null).

### 4.4 Hard delete (SUPER_ADMIN only)

```http
DELETE /api/admin/warehouses/bins/{id}
DELETE /api/admin/warehouses/racks/{id}
DELETE /api/admin/warehouses/areas/{id}
```

- Non–`SUPER_ADMIN` → `403`
- Children still present → `400`
- A location that still has an `inventory_balances` row, or is referenced by a ledger, adjustment, or count line → `400`
- Soft-delete for everyone else: PATCH status (`inactive` / `blocked`)

### Warehouse checklist

- [ ] Create warehouse without manager, then create area → 400 `manager_id`
- [ ] Duplicate warehouse code → 400
- [ ] Duplicate location code in same warehouse → 400; same code in another warehouse → allowed
- [ ] Area capacity exceeding remaining warehouse capacity → 400
- [ ] Rack/bin capacity exceeding parent remaining → 400
- [ ] Soft-deactivate area/rack (`inactive`) or bin (`blocked`)
- [ ] Soft-deactivate warehouse (`status: inactive`)
- [ ] Hard `DELETE` as non–`SUPER_ADMIN` → 403
- [ ] Hard `DELETE` as `SUPER_ADMIN` with children → 400; empty leaf → 200
- [ ] Tree shows area → racks → bins; batch codes match `{rack}-{shelf}-B{n}`
- [ ] Put-away bins include `allocatable` false when bin/parent/warehouse not active

---

## 4b. Contacts

`type` is required on list, next-code, and summary. Values: `customer` or `supplier`.

### 4b.1 Next code and create

```http
GET /api/admin/contacts/next-code?type=customer
Authorization: Bearer <token>
```

Expect `200` and `data.code` `CUS-0001` (or the next free number). Suppliers use `SUP-NNNN`.

```http
POST /api/admin/contacts
Authorization: Bearer <token>
Content-Type: application/json

{
  "type": "customer",
  "code": "CUS-0001",
  "name": "Anantha Textiles",
  "contact_person": "S. Lakshmi",
  "email": "accounts@ananthatextiles.in",
  "phone": "+91 94430 82260",
  "city": "Coimbatore",
  "state": "Tamil Nadu",
  "gst_number": "33AAFCR8226P1Z4",
  "credit_limit": 500000,
  "status": "active"
}
```

Expect `201` `ApiResponse`. `data` is the created contact. Omit `country` and the stored value is `India`. A supplier stores `credit_limit` as `0`.

### 4b.2 List, get, summary, update

```http
GET /api/admin/contacts?type=customer&search=anantha&status=active&sort=name
GET /api/admin/contacts/{id}
GET /api/admin/contacts/summary?type=customer
GET /api/admin/contacts/next-code?type=customer
```

These reads return `ApiResponse`. The contact array, one contact, summary, or next code is in `data`.

Summary for customers includes `total`, `active`, `inactive`, and `credit`. Suppliers omit `credit`.

```http
PATCH /api/admin/contacts/{id}
Authorization: Bearer <token>
Content-Type: application/json

{ "status": "inactive", "contact_person": "S. Lakshmi" }
```

Customer list rows include `orders: 0`, `revenue: 0`, and `last_order: null` until sales orders exist. Supplier `product_count` is the number of product-supplier links for that contact, including inactive links.

### 4b.3 Hard delete

```http
DELETE /api/admin/contacts/{id}
```

Non–`SUPER_ADMIN` → `403` `ApiResponse`. `SUPER_ADMIN` → `200` with a null `data` when nothing references the contact. Soft-deactivate for everyone else with `PATCH` `status: inactive`.

### Contacts checklist

- [ ] Missing or invalid `type` → error on `type`
- [ ] Duplicate code (any case) → 400 `code`
- [ ] Code shorter than 2 characters or with spaces → 400 `code`
- [ ] Bad email → 400 `email`
- [ ] GSTIN not 15 characters / wrong pattern → 400 `gst_number`; saved value is uppercase
- [ ] Negative `credit_limit` on a customer → 400
- [ ] Supplier create with a credit limit still stores `0`
- [ ] Blank country stores `India`
- [ ] `PATCH` cannot change type (type is not an update field)
- [ ] Summary counts match active/inactive rows
- [ ] Hard delete as admin → 403; as `SUPER_ADMIN` on an unused contact → 200

---

## 4d. Enum values

`ADMIN` and `SUPER_ADMIN` can list and get rows (`ENUM_VALUE_READ`). `POST` and `PATCH` stay `SUPER_ADMIN` (`ENUM_VALUE_CREATE`, `ENUM_VALUE_UPDATE`). `type_category` and `code` are stored lowercase. A code is unique inside its category. Patch can change `code`, `display_label`, `description`, `sort_order`, and `is_active`. The category stays as it was created. An empty `description` clears it. Omit a field to leave it unchanged.

```http
GET /api/admin/enum-values
GET /api/admin/enum-values?type_category=product_info_type
GET /api/admin/enum-values?type_category=client_type&active=true
GET /api/admin/enum-values/{id}
Authorization: Bearer <token>
```

`data` is the list, or one row for get-by-id. Each row has `id`, `type_category`, `code`, `display_label`, `description`, `sort_order`, `is_active`, `created_at`, and `updated_at`. Omit `type_category` to return every row. `active=true` keeps only active rows.

```http
POST /api/admin/enum-values
Authorization: Bearer <token>
Content-Type: application/json

{
  "type_category": "product_info_type",
  "code": "denier",
  "display_label": "Denier",
  "description": "Yarn thickness",
  "sort_order": 6,
  "is_active": true
}
```

Expect `201`. Duplicate code in that category → `400` on `code`.

```http
PATCH /api/admin/enum-values/{id}
Authorization: Bearer <token>
Content-Type: application/json

{ "display_label": "Denier (D)", "is_active": false }
```

---

## 4c. Inventory

Lists return `ApiResponse` with `data.items` and `data.pagination` (`page`, `page_size`, `total`). Validation errors are `400` with `success: false` and the message on `notification.exceptionMessage`.

Balances are not created here. An adjustment or a count line must already have a balance for that product, lot, and location. Create the first quantity with a goods receipt (§4e). Until a balance exists, create and post return `400`.

Shared list filters: `location_id`, `product_id`, `range`, `article`, `ply`, `tex`, `ticket`, `from`, `to` (`YYYY-MM-DD` or a date-time, on `created_at`).

### 4c.1 Stock

```http
GET /api/admin/inventory/stock?page=1&page_size=20&status=active&sort=lot
GET /api/admin/inventory/stock/summary
GET /api/admin/inventory/stock/{id}
```

Permission: `STOCK_READ`. `status` is the lot status: `active`, `depleted`, `quarantine`, `expired`, `blocked`. `sort` is `lot`, `product`, `quantity`, or `expiry_date`. `available_quantity` is quantity minus reserved quantity. `received_date` and `received_by` on the detail are null.

### 4c.2 Transactions (read-only)

```http
GET /api/admin/inventory/transactions?type=adjustment_out&status=posted
GET /api/admin/inventory/transactions/summary
GET /api/admin/inventory/transactions/{id}
```

Permission: `TRANSACTION_READ`. `type` is a `txn_type` code. `status` is `draft`, `posted`, or `cancelled`. `performed_by` is `{id, name}`. `created_at` is the recorded time. `reference` is built from `reference_type` and the source document number (`stock_adjustment ADJ-000001`, `stock_count CC-000001`). Rows appear when an adjustment or a count is posted.

### 4c.3 Adjustments

All lines on one adjustment use the same `direction` (`in` or `out`). `in` adds to an existing balance. `out` removes from it and cannot go below reserved quantity. `before_quantity` and `after_quantity` are filled on post.

```http
POST /api/admin/inventory/adjustments
Authorization: Bearer <token>
Content-Type: application/json

{
  "warehouse_id": 1,
  "reason": "Physical count correction",
  "lines": [
    {
      "product_id": 1,
      "lot_id": 1,
      "location_id": 1,
      "quantity": 1,
      "direction": "out",
      "reason": "Missing cone"
    }
  ]
}
```

Expect `201` and `data.status` `draft`, `data.adjustment_no` like `ADJ-000001`.

```http
PATCH /api/admin/inventory/adjustments/{id}
POST /api/admin/inventory/adjustments/{id}/submit
POST /api/admin/inventory/adjustments/{id}/approve
POST /api/admin/inventory/adjustments/{id}/post
```

Reject with `POST .../reject` from `submitted`. Cancel with `POST .../cancel` from `draft`, `submitted`, or `approved`. Post writes one `stock_transaction` (`adjustment_in` or `adjustment_out`, status `posted`) and sets `posted_transaction`. `ADMIN` and `SUPER_ADMIN` can both approve and post.

```http
GET /api/admin/inventory/adjustments?status=submitted&warehouse_id=1
GET /api/admin/inventory/adjustments/summary
GET /api/admin/inventory/adjustments/{id}
```

### 4c.4 Stock counts

A count does not point at an adjustment. Posting it writes its own ledger rows with `reference_type` `stock_count`.

```http
POST /api/admin/inventory/stock-counts
Authorization: Bearer <token>
Content-Type: application/json

{ "warehouse_id": 1, "note": "Monthly verification" }
```

Expect `201`, `data.status` `draft`, `data.count_number` like `CC-000001`.

```http
POST /api/admin/inventory/stock-counts/{id}/lines
Content-Type: application/json

{
  "product_id": 1,
  "lot_id": 1,
  "location_id": 1,
  "counted_quantity": 10
}
```

`system_quantity` is copied from the balance. `variance` is counted minus system. Line `status` is `counted` once a quantity is entered, otherwise `pending`.

```http
POST /api/admin/inventory/stock-counts/{id}/start
POST /api/admin/inventory/stock-counts/{id}/submit
POST /api/admin/inventory/stock-counts/{id}/approve
POST /api/admin/inventory/stock-counts/{id}/post
```

Lines can be added, patched, or deleted while the count is `draft` or `in_progress`. Submit is only from `in_progress`, and it refreshes `system_quantity`. Post fails if the balance changed after submit. A positive variance posts `adjustment_in`. A negative variance posts `adjustment_out`. Both can exist; `posted_transactions` lists them, and `posted_transaction` is the outbound one when both exist. A count with no variance still becomes `posted` and has no transaction.

```http
GET /api/admin/inventory/stock-counts?status=submitted
GET /api/admin/inventory/stock-counts/summary
GET /api/admin/inventory/stock-counts/{id}
```

### Inventory checklist

- [ ] Stock list with an unknown lot status → `400`
- [ ] Adjustment for a product/lot/location with no balance → `400` on `product_id`
- [ ] Mixed `in` and `out` lines on one adjustment → `400` on `direction`
- [ ] `out` that would drop quantity below reserved stock → `400` on post
- [ ] Draft adjustment can be edited; submitted cannot
- [ ] Non-approved adjustment post → `400`
- [ ] Posted adjustment appears under transactions with `reference_type` `stock_adjustment`
- [ ] Count line in another warehouse → `400` on `location_id`
- [ ] Count submit before start → `400`
- [ ] Posted count is visible from `GET /transactions?type=adjustment_in` or `adjustment_out`, not from the adjustment list

---

## 5. Permission matrix (products)

| Method | Route | Permission |
|--------|-------|------------|
| GET | `/api/admin/product-info` | `PRODUCT_INFO_READ` |
| GET | `/api/admin/product-info/types` | `PRODUCT_INFO_READ` |
| GET | `/api/admin/product-info/{id}` | `PRODUCT_INFO_READ` |
| POST | `/api/admin/product-info` | `PRODUCT_INFO_CREATE` |
| PATCH | `/api/admin/product-info/{id}` | `PRODUCT_INFO_UPDATE` |
| GET | `/api/admin/shades` | `SHADE_READ` |
| POST | `/api/admin/shades` | `SHADE_CREATE` |
| PATCH | `/api/admin/shades/{id}` | `SHADE_UPDATE` |
| GET | `/api/admin/units` | `UNIT_READ` |
| POST | `/api/admin/units` | `UNIT_CREATE` |
| PATCH | `/api/admin/units/{id}` | `UNIT_UPDATE` |
| GET | `/api/admin/products` | `PRODUCT_READ` |
| GET | `/api/admin/products/{id}` | `PRODUCT_READ` |
| POST | `/api/admin/products` | `PRODUCT_CREATE` |
| PUT | `/api/admin/products/{id}` | `PRODUCT_UPDATE` |
| PATCH | `/api/admin/products/{id}/status` | `PRODUCT_UPDATE` |
| GET | `/api/admin/products/{id}/barcodes` | `BARCODE_READ` |
| POST | `/api/admin/products/{id}/barcodes` | `BARCODE_CREATE` |
| PATCH | `/api/admin/products/{id}/barcodes/{barcodeId}` | `BARCODE_UPDATE` |
| DELETE | `/api/admin/products/{id}/barcodes/{barcodeId}` | `BARCODE_DELETE` |
| GET | `/api/admin/products/{id}/suppliers` | `PRODUCT_SUPPLIER_READ` |
| POST | `/api/admin/products/{id}/suppliers` | `PRODUCT_SUPPLIER_CREATE` |
| PATCH | `/api/admin/products/{id}/suppliers/{linkId}` | `PRODUCT_SUPPLIER_UPDATE` |
| DELETE | `/api/admin/products/{id}/suppliers/{linkId}` | `PRODUCT_SUPPLIER_DELETE` (+ `SUPER_ADMIN` in service) |
| GET | `/api/admin/unit-conversions` | `UNIT_CONVERSION_READ` |
| POST | `/api/admin/unit-conversions` | `UNIT_CONVERSION_CREATE` |
| PATCH | `/api/admin/unit-conversions/{id}` | `UNIT_CONVERSION_UPDATE` |
| DELETE | `/api/admin/unit-conversions/{id}` | `UNIT_CONVERSION_DELETE` |

`SUPER_ADMIN` and `ADMIN` are seeded with the master and SKU permissions in V10. Barcode and product-supplier permissions are seeded in V16. Unit-conversion write permissions are seeded in V19.

---

## 6. Permission matrix (warehouses)

| Method | Route | Permission |
|--------|-------|------------|
| GET | `/api/admin/warehouses` | `WAREHOUSE_READ` |
| POST | `/api/admin/warehouses` | `WAREHOUSE_CREATE` |
| PATCH | `/api/admin/warehouses` | `WAREHOUSE_UPDATE` |
| GET | `/api/admin/warehouses/tree` | `WAREHOUSE_READ` |
| GET | `/api/admin/warehouses/summary` | `WAREHOUSE_READ` |
| GET | `/api/admin/warehouses/capacity` | `WAREHOUSE_READ` |
| GET | `/api/admin/warehouses/putaway-bins` | `WAREHOUSE_READ` |
| POST | `/api/admin/warehouses/areas` | `LOCATION_CREATE` |
| PATCH | `/api/admin/warehouses/areas/{id}` | `LOCATION_UPDATE` |
| DELETE | `/api/admin/warehouses/areas/{id}` | `LOCATION_DELETE` (+ `SUPER_ADMIN` in service) |
| POST | `/api/admin/warehouses/racks` | `LOCATION_CREATE` |
| PATCH | `/api/admin/warehouses/racks/{id}` | `LOCATION_UPDATE` |
| DELETE | `/api/admin/warehouses/racks/{id}` | `LOCATION_DELETE` (+ `SUPER_ADMIN` in service) |
| POST | `/api/admin/warehouses/bins` | `LOCATION_CREATE` |
| POST | `/api/admin/warehouses/bins/batch` | `LOCATION_CREATE` |
| PATCH | `/api/admin/warehouses/bins/{id}` | `LOCATION_UPDATE` |
| DELETE | `/api/admin/warehouses/bins/{id}` | `LOCATION_DELETE` (+ `SUPER_ADMIN` in service) |

Seeded in V12 for `SUPER_ADMIN` and `ADMIN`.

---

## 6b. Permission matrix (contacts)

| Method | Route | Permission |
|--------|-------|------------|
| GET | `/api/admin/contacts` | `CONTACT_READ` |
| GET | `/api/admin/contacts/next-code` | `CONTACT_READ` |
| GET | `/api/admin/contacts/summary` | `CONTACT_READ` |
| GET | `/api/admin/contacts/{id}` | `CONTACT_READ` |
| POST | `/api/admin/contacts` | `CONTACT_CREATE` |
| PATCH | `/api/admin/contacts/{id}` | `CONTACT_UPDATE` |
| DELETE | `/api/admin/contacts/{id}` | `CONTACT_DELETE` (+ `SUPER_ADMIN` in service) |

Seeded in V14 for `SUPER_ADMIN` and `ADMIN`.

---

## 6c. Permission matrix (inventory)

| Method | Route | Permission |
|--------|-------|------------|
| GET | `/api/admin/inventory/stock` | `STOCK_READ` |
| GET | `/api/admin/inventory/stock/summary` | `STOCK_READ` |
| GET | `/api/admin/inventory/stock/{id}` | `STOCK_READ` |
| GET | `/api/admin/inventory/transactions` | `TRANSACTION_READ` |
| GET | `/api/admin/inventory/transactions/summary` | `TRANSACTION_READ` |
| GET | `/api/admin/inventory/transactions/{id}` | `TRANSACTION_READ` |
| GET | `/api/admin/inventory/adjustments` | `ADJUSTMENT_READ` |
| GET | `/api/admin/inventory/adjustments/summary` | `ADJUSTMENT_READ` |
| GET | `/api/admin/inventory/adjustments/{id}` | `ADJUSTMENT_READ` |
| POST | `/api/admin/inventory/adjustments` | `ADJUSTMENT_CREATE` |
| PATCH | `/api/admin/inventory/adjustments/{id}` | `ADJUSTMENT_UPDATE` |
| POST | `/api/admin/inventory/adjustments/{id}/submit` | `ADJUSTMENT_UPDATE` |
| POST | `/api/admin/inventory/adjustments/{id}/cancel` | `ADJUSTMENT_UPDATE` |
| POST | `/api/admin/inventory/adjustments/{id}/approve` | `ADJUSTMENT_APPROVE` |
| POST | `/api/admin/inventory/adjustments/{id}/reject` | `ADJUSTMENT_APPROVE` |
| POST | `/api/admin/inventory/adjustments/{id}/post` | `ADJUSTMENT_POST` |
| GET | `/api/admin/inventory/stock-counts` | `STOCK_COUNT_READ` |
| GET | `/api/admin/inventory/stock-counts/summary` | `STOCK_COUNT_READ` |
| GET | `/api/admin/inventory/stock-counts/{id}` | `STOCK_COUNT_READ` |
| POST | `/api/admin/inventory/stock-counts` | `STOCK_COUNT_CREATE` |
| PATCH | `/api/admin/inventory/stock-counts/{id}` | `STOCK_COUNT_UPDATE` |
| POST | `/api/admin/inventory/stock-counts/{id}/lines` | `STOCK_COUNT_UPDATE` |
| PATCH | `/api/admin/inventory/stock-counts/{id}/lines/{lineId}` | `STOCK_COUNT_UPDATE` |
| DELETE | `/api/admin/inventory/stock-counts/{id}/lines/{lineId}` | `STOCK_COUNT_UPDATE` |
| POST | `/api/admin/inventory/stock-counts/{id}/start` | `STOCK_COUNT_UPDATE` |
| POST | `/api/admin/inventory/stock-counts/{id}/submit` | `STOCK_COUNT_UPDATE` |
| POST | `/api/admin/inventory/stock-counts/{id}/cancel` | `STOCK_COUNT_UPDATE` |
| POST | `/api/admin/inventory/stock-counts/{id}/approve` | `STOCK_COUNT_APPROVE` |
| POST | `/api/admin/inventory/stock-counts/{id}/reject` | `STOCK_COUNT_APPROVE` |
| POST | `/api/admin/inventory/stock-counts/{id}/post` | `STOCK_COUNT_POST` |

Seeded in V18 for `SUPER_ADMIN` and `ADMIN`. Both roles can approve and post.

## 6d. Permission matrix (enum values)

| Method | Route | Permission |
|--------|-------|------------|
| GET | `/api/admin/enum-values` | `ENUM_VALUE_READ` |
| GET | `/api/admin/enum-values/{id}` | `ENUM_VALUE_READ` |
| POST | `/api/admin/enum-values` | `ENUM_VALUE_CREATE` |
| PATCH | `/api/admin/enum-values/{id}` | `ENUM_VALUE_UPDATE` |

`ENUM_VALUE_READ` is on `SUPER_ADMIN` (V24) and `ADMIN` (V25). Create and update stay `SUPER_ADMIN`.

## 6e. Permission matrix — Order to Dispatch

| Method | Route | Permission |
|--------|-------|------------|
| GET | `/api/admin/purchase-orders` | `PURCHASE_ORDER_READ` |
| GET | `/api/admin/purchase-orders/totals` | `PURCHASE_ORDER_READ` |
| GET | `/api/admin/purchase-orders/receivable` | `PURCHASE_ORDER_READ` |
| GET | `/api/admin/purchase-orders/{id}` | `PURCHASE_ORDER_READ` |
| POST | `/api/admin/purchase-orders` | `PURCHASE_ORDER_CREATE` |
| PATCH | `/api/admin/purchase-orders/{id}` | `PURCHASE_ORDER_UPDATE` |
| GET | `/api/admin/goods-receipts` | `GOODS_RECEIPT_READ` |
| GET | `/api/admin/goods-receipts/totals` | `GOODS_RECEIPT_READ` |
| GET | `/api/admin/goods-receipts/{id}` | `GOODS_RECEIPT_READ` |
| POST | `/api/admin/goods-receipts` | `GOODS_RECEIPT_CREATE` |
| GET | `/api/admin/sales-orders` | `SALES_ORDER_READ` |
| GET | `/api/admin/sales-orders/totals` | `SALES_ORDER_READ` |
| GET | `/api/admin/sales-orders/reference` | `SALES_ORDER_READ` |
| GET | `/api/admin/sales-orders/{id}` | `SALES_ORDER_READ` |
| POST | `/api/admin/sales-orders` | `SALES_ORDER_CREATE` |
| PATCH | `/api/admin/sales-orders/{id}` | `SALES_ORDER_UPDATE` |
| GET | `/api/admin/pick-lists` | `PICK_LIST_READ` |
| GET | `/api/admin/pick-lists/totals` | `PICK_LIST_READ` |
| GET | `/api/admin/pick-lists/{id}` | `PICK_LIST_READ` |
| PATCH | `/api/admin/pick-lists/{id}` | `PICK_LIST_UPDATE` |
| GET | `/api/admin/dispatches` | `DISPATCH_READ` |
| GET | `/api/admin/dispatches/queue` | `DISPATCH_READ` |
| GET | `/api/admin/dispatches/totals` | `DISPATCH_READ` |
| GET | `/api/admin/dispatches/candidate/{salesOrderId}` | `DISPATCH_READ` |
| GET | `/api/admin/dispatches/{id}` | `DISPATCH_READ` |
| POST | `/api/admin/dispatches` | `DISPATCH_CREATE` |
| PATCH | `/api/admin/dispatches/{id}` | `DISPATCH_UPDATE` |
| GET | `/api/admin/order-products` | `ORDER_PRODUCT_READ` |

Seeded in V28 for `SUPER_ADMIN` and `ADMIN`.

---

## 4e. Order to Dispatch

Happy path: purchase order → goods receipt (creates lots) → sales order with reserve → pick → dispatch. Every response is `ApiResponse`. Request and response fields are snake_case.

### 4e.1 Purchase order

```http
POST /api/admin/purchase-orders
Authorization: Bearer <token>
Content-Type: application/json

{
  "supplier_id": 1,
  "order_date": "2026-11-12",
  "expected_date": "2026-11-20",
  "submit": true,
  "lines": [{ "product_id": 1, "quantity": 100, "unit_price": 250 }]
}
```

Expect `201`. `data` has `id`, `po_number` (`PO-######`), and `status` (`submitted` when `submit` is true, else `draft`).

```http
PATCH /api/admin/purchase-orders/{id}
{ "action": "approve" }
```

Then list receivable POs and create a GRN:

```http
GET /api/admin/purchase-orders/receivable
GET /api/admin/purchase-orders/totals
GET /api/admin/order-products?q=ply&supplier=1
```

### 4e.2 Goods receipt (first stock)

```http
POST /api/admin/goods-receipts
Authorization: Bearer <token>
Content-Type: application/json

{
  "purchase_order_id": 1,
  "warehouse_id": 1,
  "note": "Lorry 1",
  "lines": [
    { "purchase_order_line_id": 1, "quantity": 50, "bin_id": 30, "batch_no": "SUP-LOT-1" }
  ]
}
```

Expect `201` with `grn_number` (`GRN-######`). Side effects: mint `CT-STK-######` lots, insert balances, post ledger `receipt`, update PO line `received_quantity` and PO status `partially_received` or `received`. Bin must be an empty active `storage_bin` in that warehouse. Reuse `GET /api/admin/warehouses/putaway-bins`.

### 4e.3 Sales order

```http
GET /api/admin/sales-orders/reference
POST /api/admin/sales-orders
Authorization: Bearer <token>
Content-Type: application/json

{
  "customer_id": 1,
  "warehouse_id": 1,
  "order_date": "2026-11-12",
  "reserve": true,
  "assigned_to": 2,
  "lines": [{ "product_id": 1, "quantity": 20, "unit_price": 280 }]
}
```

Expect `201`. `order_number` is `CT/YY-####`. With `reserve: true`, free stock is held FIFO, `inventory_reservations` are created, and a pick list is issued. With `reserve: false`, status is `confirmed` and no pick list is created.

```http
PATCH /api/admin/sales-orders/{id}
{ "action": "issue_pick_list", "assigned_to": 2 }
```

Cancel with `{ "action": "cancel" }` is blocked once any line has `dispatched_quantity > 0`.

### 4e.4 Pick list

```http
GET /api/admin/pick-lists
GET /api/admin/pick-lists/{id}
PATCH /api/admin/pick-lists/{id}
Authorization: Bearer <token>
Content-Type: application/json

{ "action": "assign", "assigned_to": 2 }
```

Then `start`, `record` (with `lines[{id, picked_quantity}]`), and `complete`. Complete may create a continuation pick for remaining reserved qty. Cancel does not release reservations.

### 4e.5 Dispatch

```http
GET /api/admin/dispatches/queue
GET /api/admin/dispatches/candidate/{salesOrderId}
POST /api/admin/dispatches
Authorization: Bearer <token>
Content-Type: application/json

{
  "sales_order_id": 1,
  "vehicle_number": "GJ-05-AB-1234",
  "lines": [{ "sales_order_line_id": 1, "lot_id": 1, "quantity": 20 }]
}
```

Expect `201` draft (`DSP-######`). Confirm:

```http
PATCH /api/admin/dispatches/{id}
{ "action": "dispatch", "vehicle_number": "GJ-05-AB-1234" }
```

Confirm draws stock, settles reservations, posts ledger `dispatch`, and refreshes the sales order status. Cancel only works while draft: `{ "action": "cancel" }`.

### Order-to-Dispatch checklist

- [ ] Duplicate product on one PO or SO → 400
- [ ] Cancel PO with received qty → 400
- [ ] GRN over outstanding qty → 400; same `bin_id` twice → 400
- [ ] Reserve more than free stock → 400
- [ ] Pick complete with 0 picked → 400
- [ ] Dispatch more than dispatchable → 400
- [ ] Confirm dispatch without vehicle when draft has none → 400

---

## 7. Not implemented yet (skip / expect absence)

Do **not** treat these as bugs until built. When implemented, move them into sections above (and document behavior in [README.md](README.md) if it is architectural).

| Area | Status |
|------|--------|
| Opening stock / GRN without a purchase order | Deferred. Goods receipts against a PO create the first lot quantity |
| Other location types in Management UI (receiving_bay, etc.) | Deferred |
| Contact list metrics (`orders`, `revenue`, `last_order`) | Still placeholders; sales orders exist but contact enrichment is not wired |
| `GET /api/admin/contacts/{id}/orders` | Deferred |
| Floor counter sale (`dispatchSale`) | Out of Order-to-Dispatch scope |

---

## 8. Automated tests (complement, not a substitute)

```bash
# Unit + slice tests (no Docker)
./mvnw test

# Full verify including Testcontainers IT (Docker required)
./mvnw verify
```

Manual E2E in this guide covers auth cookies/MFA mail and full admin product flows that may not be fully covered by IT yet.

---

## Keeping this guide current

**Rule:** Any PR or local change that adds/changes an HTTP API, permission, route mapping, request/response field, or status enum must update this file in the same change.

### Checklist for new work

1. Add or update a numbered section (or subsection) with method, path, sample body, and expected status.
2. Add rows to [§5](#5-permission-matrix-products) / [§6](#6-permission-matrix-warehouses) or a new matrix.
3. Add negative cases to the relevant checklist.
4. If the feature was listed under [§7 Not implemented](#7-not-implemented-yet-skip--expect-absence), remove it from that table.
5. Update [README.md](README.md) for setup, architecture, or module summaries (not a third Markdown file).
6. Keep curl/JSON examples copy-pasteable; prefer real field names (`HSN_code`, `shade_id`, `manager_id`, `warehouse`).

### Suggested section order for new modules

1. Auth prerequisites (if new public auth routes)
2. Module happy path (dependency order: masters → dependents)
3. Permission matrix
4. Negative checklist
5. Deferred items
