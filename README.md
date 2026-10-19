# Restaurant Management System — Customer Ordering Frontend

Enterprise-grade customer ordering interface for a multi-location Restaurant Management Platform targeting the European market. This repository delivers the public-facing web client that supports discovery, customization, checkout, invoicing, and order lifecycle visibility for dine-in, pickup, and delivery operations.

---

## 1. Product Overview

The platform enables restaurants to operate digital ordering channels with:

| Domain | Capabilities |
|--------|--------------|
| **Catalog** | Categories, subcategories, products, search, offers, best dishes |
| **Customization** | Size selection, extras / add-ons, allergen information |
| **Pricing** | Dynamic line pricing (base + extras × quantity), delivery fees, promotions |
| **Commerce** | Cart, cart verification, checkout, payment method selection |
| **Fulfillment** | Delivery / pickup, address book, address history |
| **Accounts** | Guest & registered customers, profile, password recovery |
| **Orders** | Order submission, invoice data, authenticated history, guest local history |
| **Compliance** | Cookie consent (EN/DE) aligned with European privacy expectations |

This frontend integrates with a backend REST API that owns catalog, pricing, customers, addresses, payments, and order persistence.

---

## 2. Scope of This Repository

| In scope | Out of scope |
|----------|--------------|
| Customer SPA (React) | Kitchen / POS / admin back-office UI |
| Ordering UX & client-side cart | Backend business rules & persistence |
| Auth session handling (token) | Payment gateway PCI internals |
| Dockerized static delivery | Multi-tenant admin configuration |

---

## 3. Technology Stack

| Layer | Technology |
|-------|------------|
| Runtime | React 18 |
| Routing | React Router v6 |
| HTTP | Axios (centralized client + interceptors) |
| Forms / Validation | React Hook Form, Yup |
| Styling | Tailwind CSS, styled-components, component CSS |
| Motion / UX | Framer Motion, React Spring, AOS, Notiflix |
| Build | Create React App (`react-scripts`) |
| Delivery | Multi-stage Docker → Nginx |
| CI/CD | GitLab Auto DevOps |

---

## 4. Getting Started

### Prerequisites

- Node.js 16+ (aligned with Docker build image)
- npm 8+
- Access to the Restaurant Management API

### Install

```bash
npm install
```

### Configure

Point the API base URL in:

- `src/api/index.js` — endpoint registry
- `src/utils/axios.js` — HTTP client defaults

> **Security note:** Do not commit production API keys or secrets. Prefer environment-injected configuration for staging and production.

### Run (development)

```bash
npm start
```

Application: `http://localhost:3000`

### Test

```bash
npm test
```

### Production build

```bash
npm run build
```

### Docker

```bash
docker compose up --build
```

Serves the production build via Nginx on port `3000` (mapped to container port `80`).

---

## 5. Application Routes

| Path | Purpose | Auth |
|------|---------|------|
| `/` | Home — categories, offers, featured dishes | Public |
| `/products` | Category / subcategory product listing | Public |
| `/product/:id` | Product details, sizes, extras | Public |
| `/search-results` | Catalog search results | Public |
| `/cart` | Cart review & line-item editing | Public |
| `/create-order` | Checkout — address, payment, delivery | Guest or registered |
| `/MyOrders` | Authenticated order history | Token |
| `/guestOrder` | Guest order history (local) | Guest |
| `/profile` | Customer profile | Token |
| `/login`, `/signup` | Authentication | Public |
| `/forgotpassword`, `/reset-password` | Password recovery | Public |

---

## 6. Core Business Flows

### 6.1 Browse → Customize → Cart

1. Customer browses categories / subcategories or search.
2. Opens product details (sizes, extras, allergens).
3. Client computes **dynamic unit price** = base (or size) + selected extras.
4. Line is stored in `localStorage` cart with a composite `uniqueKey` (product + size + extras).

### 6.2 Checkout → Order → Invoice

1. Cart is reviewed; optional cart verification against the API.
2. Customer selects delivery or pickup, payment method, and address.
3. Delivery fee and optional promotion code are applied.
4. Order payload is submitted (`checkout-mobile`).
5. Response includes invoice metadata; guest orders may be persisted locally for `/guestOrder`.

### 6.3 Accounts

- **Registered:** Bearer token + customer id; address history; order history API.
- **Guest:** Lightweight customer record; local address / order snapshots.

---

## 7. Project Structure

```text
src/
├── api/                 # Endpoint registry & domain hooks/services
├── components/          # Presentational & feature UI
│   ├── cart/
│   ├── checkout/
│   ├── layout/header/
│   ├── products/
│   ├── productDetails/
│   └── cookieConsent/
├── pages/               # Route-level screens
├── utils/               # Axios client, schemas
├── App.js               # Route host + cookie consent
└── router.js            # Declarative route table
```

---

## 8. API Surface (Frontend Consumption)

Primary resource groups consumed by this client:

- **Auth / Customers** — login, signup, guest, profile, password reset
- **Catalog** — categories, subcategories, products, search, offers, best dishes
- **Commerce** — verify cart, checkout, payment methods, delivery fees
- **Addresses** — create, update, history, cities
- **Orders** — authenticated order history
- **Content** — about, contact infos

Full endpoint mapping: `src/api/index.js`.

---

## 9. Quality & Operations

- **Lint / unit:** CRA ESLint + Jest / React Testing Library
- **CI:** GitLab Auto DevOps stages (build → test → deploy → security scanning)
- **Runtime errors:** Axios interceptors handle `401` (redirect to login) and network failures
- **Privacy:** Cookie consent modal with English / German copy for EU-facing deployments

---

## 10. Related Documentation

| Document | Description |
|----------|-------------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | System context, layers, modules, data ownership |
| [DIAGRAMS.md](./DIAGRAMS.md) | C4-style and sequence diagrams (Mermaid) |

---

## 11. Team & Market Context

Built as part of a larger engineering organization delivering a full operational restaurant platform for European operators — covering customer channels, administrative control, and day-to-day multi-location operations. This repository is the **customer ordering web client** within that platform.

---

## License

Proprietary — all rights reserved. Unauthorized distribution or reuse is prohibited.
