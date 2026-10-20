# SWYNEX Project Architecture — Public Distribution Service

## 1. Product idea
**Public Distribution Service (PDS)** is a web application for managing ration-card families, available ration products, monthly purchase quotas, and transaction approvals. Family users can verify their ration card, browse eligible items, submit a purchase, and review transaction status. Administrators manage families and inventory, monitor transactions, and update delivery status.

## 2. Goals and users
- **Family user:** verify card, view eligible products and quota, create a purchase, view receipts/history.
- **Administrator:** sign in, manage family records and products, approve/cancel transactions, update delivery status, and view reports.

## 3. Technology stack
- **Frontend:** React 18, React Router-style component structure, Axios, CSS.
- **Backend:** Node.js, Express.
- **Database:** MySQL using `mysql2`.
- **Authentication:** JSON Web Tokens (JWT); password hashing with bcrypt.
- **API prefix:** `/api`.

## 4. High-level architecture
```text
React Frontend (port 3000)
        |
        | HTTP / JSON (Axios)
        v
Express REST API (port 5000)
  ├── Auth routes/controllers + auth/admin middleware
  ├── Family routes/controllers
  ├── Product routes/controllers
  └── Transaction routes/controllers
        |
        v
MySQL database
  ├── admins
  ├── families
  ├── products
  └── transactions
```

## 5. Data models

### Admin
| Field | Type | Notes |
|---|---|---|
| id | INT, PK | Auto-increment |
| name | VARCHAR | Administrator display name |
| email | VARCHAR, unique | Login identity |
| password | VARCHAR | Store bcrypt hash only |
| role | ENUM/VARCHAR | e.g. `admin` |
| created_at | TIMESTAMP | Creation time |

### Family
| Field | Type | Notes |
|---|---|---|
| id | INT, PK | Auto-increment |
| card_number | VARCHAR, unique | Ration card identifier |
| head_name | VARCHAR | Head of family |
| head_aadhar | VARCHAR, sensitive | Avoid exposing in API/logs; consider storing only masked value |
| head_phone | VARCHAR | Contact |
| address | TEXT | Household address |
| ration_type | VARCHAR | Card/category type |
| monthly_quota | JSON | Product-specific monthly allowance |
| members | JSON | Household members; normalize into a `family_members` table in a future version |
| is_active | BOOLEAN | Soft-delete flag |
| created_at / updated_at | TIMESTAMP | Audit timestamps |

### Product
| Field | Type | Notes |
|---|---|---|
| id | INT, PK | Auto-increment |
| name | VARCHAR | Product name |
| price | DECIMAL(10,2) | Unit price |
| limit | DECIMAL/INT | Per-family purchase limit, define unit consistently |
| unit | VARCHAR | kg, litre, etc. |
| image | VARCHAR, nullable | Image URL/path |
| stock | DECIMAL/INT | Available stock |
| is_active | BOOLEAN | Soft-delete flag |
| created_at / updated_at | TIMESTAMP | Audit timestamps |

### Transaction
| Field | Type | Notes |
|---|---|---|
| id | INT, PK | Auto-increment |
| transaction_id | VARCHAR, unique | Public reference |
| card_number | VARCHAR, FK to families.card_number | Purchasing household |
| member_name | VARCHAR | Verified household member |
| items | JSON | Snapshot of product id/name/quantity/unit price |
| total | DECIMAL(10,2) | Server-calculated total |
| payment_method | VARCHAR | Selected payment method |
| biometric_type | VARCHAR, nullable | Verification method; store minimal metadata |
| approval_status | ENUM | pending, approved, cancelled |
| delivery_status | ENUM | pending, processing, delivered |
| approved_at / delivered_at | TIMESTAMP, nullable | Lifecycle timestamps |
| created_at | TIMESTAMP | Purchase time |

**Relationships:** One family can have many transactions. Transactions contain one or more product line items. Admins approve transactions. Products are referenced by transaction item snapshots.

## 6. REST API routes
All responses should use JSON. Protected routes require `Authorization: Bearer <token>`.

| Method | Route | Access | Purpose |
|---|---|---|---|
| POST | `/api/auth/admin/login` | Public | Admin login; returns token |
| POST | `/api/auth/user/verify` | Public* | Verify ration card |
| POST | `/api/auth/user/biometric` | Public* | Verify biometric flow |
| GET | `/api/families/:cardNumber` | Public* | Look up a family/card |
| GET | `/api/families` | Admin | List active families |
| POST | `/api/families` | Admin | Create family |
| PUT | `/api/families/:cardNumber` | Admin | Update family |
| DELETE | `/api/families/:cardNumber` | Admin | Soft-deactivate family |
| GET | `/api/products` | Public | List active products |
| GET | `/api/products/:id` | Public | Get product details |
| POST | `/api/products` | Admin | Create product |
| PUT | `/api/products/:id` | Admin | Update product |
| DELETE | `/api/products/:id` | Admin | Deactivate product |
| POST | `/api/transactions` | Verified user | Submit purchase |
| GET | `/api/transactions/card/:cardNumber` | Verified owner | View card history |
| GET | `/api/transactions` | Admin | List transactions |
| PUT | `/api/transactions/:id/approve` | Admin | Approve transaction |
| PUT | `/api/transactions/:id/cancel` | Admin | Cancel transaction |
| PUT | `/api/transactions/:id/delivery` | Admin | Update delivery status |

\* **Security note:** the existing route shape includes some unauthenticated card lookups and purchase/history endpoints. Before production, require verified sessions and enforce ownership server-side. Never rely on a client-provided card number as authorization. Protect Aadhaar/biometric-related data, avoid storing biometric templates, validate every request, rate-limit login/verification, and audit admin actions.

## 7. Frontend screens
1. **Welcome / Login:** choose family verification or administrator login.
2. **Family verification:** enter ration-card details and complete the supported verification flow.
3. **Family dashboard:** show masked household/card details, monthly quota, and current eligibility.
4. **Ration items:** browse available products, unit prices, remaining stock, and allowed quantities.
5. **Cart / Checkout:** review quantities and server-validated total, then submit the purchase.
6. **Transaction status / Receipt:** show reference number, approval status, delivery status, and printable receipt.
7. **Transaction history:** list purchases for the authenticated family.
8. **Admin dashboard:** key counts and transaction overview.
9. **Family management:** search, add, update, and deactivate family records.
10. **Product management:** create/edit/deactivate products and update stock.
11. **Transaction monitoring:** approve/cancel purchases and update delivery status.
12. **Analytics / Reports:** summary of transactions and inventory (role-restricted).

## 8. Core flows
### Family purchase
1. User verifies card and identity.
2. Frontend loads active products and eligible quota.
3. User selects products and quantities.
4. API rechecks stock, quota, and monthly purchase eligibility inside a database transaction.
5. API calculates total on the server and creates a pending transaction.
6. Admin approves/cancels; delivery status is updated as the order progresses.

### Admin management
1. Admin signs in and receives a JWT.
2. Frontend sends the token on protected API requests.
3. API validates token and role.
4. Changes are validated, persisted, and logged.

## 9. Non-functional requirements
- Use HTTPS in production and keep secrets in environment variables.
- Validate and sanitize inputs; use parameterized SQL.
- Add pagination/filtering to admin lists.
- Use consistent error responses and structured logs without personal data.
- Add unit/integration tests for auth, quota rules, stock updates, and transaction state transitions.
- Add database migrations and seed data for local development.
- Configure CORS with the deployed frontend origin, not `*`.
- Use database transactions/locking to prevent overselling or duplicate monthly purchases.
- Keep `.env`, credentials, personal data, logs, and `node_modules` out of Git.

## 10. Suggested starter repository structure
```text
SWYNEX-Project-Architecture/
├── README.md
├── ARCHITECTURE.md
├── .gitignore
└── app/
    ├── Backend/
    │   ├── config/ controllers/ middleware/ models/ routes/
    │   └── server.js
    └── Frontend/
        └── src/
```

## 11. Future improvements
- Normalize family members and transaction line items into relational tables.
- Add explicit quota-ledger records for reliable monthly balance calculation.
- Add API versioning, automated tests, CI checks, monitoring, and database backups.
- Replace demo biometric integration with a compliant, privacy-preserving verification provider if legally and operationally appropriate.
