# Architecture Document  
## Restaurant Management System — Customer Ordering Frontend

| Field | Value |
|-------|--------|
| **Document type** | Software Architecture Description (SAD) |
| **Standard alignment** | ISO/IEC/IEEE 42010 viewpoints · C4 model · REST API style |
| **Audience** | Engineering, architecture, QA, DevOps, product |
| **Scope** | Public customer web client (this repository) |
| **Market** | European multi-location restaurant operations |
| **Language** | English |

---

## 1. Purpose

This document describes the architectural decisions, structural views, and operational characteristics of the customer-facing ordering frontend for the Restaurant Management System (RMS).

The system supports:

- Menu discovery (categories, items, search, promotional surfaces)
- Product configuration (sizes, custom options / extras)
- Dynamic pricing on the client, reconciled at checkout with server fees and promotions
- Cart → checkout → invoice → order tracking / history
- Guest and registered customer journeys
- Privacy consent appropriate for EU deployments

---

## 2. System Context (Level 1)

```text
┌──────────────────────────────────────────────────────────────────────────┐
│                        Restaurant Management Platform                     │
│                                                                          │
│  ┌────────────────────┐     HTTPS/REST      ┌─────────────────────────┐  │
│  │ Customer Web SPA   │ ──────────────────► │ RMS Backend API         │  │
│  │ (this repository)  │ ◄────────────────── │ Catalog · Auth · Orders │  │
│  └─────────┬──────────┘                     │ Addresses · Payments    │  │
│            │ localStorage                    └───────────┬─────────────┘  │
│            ▼                                             │                │
│  ┌────────────────────┐                     ┌───────────▼─────────────┐  │
│  │ Browser Storage    │                     │ Persistence / Gateways  │  │
│  │ cart · session     │                     │ DB · Payment providers  │  │
│  │ guest orders       │                     └─────────────────────────┘  │
│  └────────────────────┘                                                   │
└──────────────────────────────────────────────────────────────────────────┘
         ▲
         │ Browser
┌────────┴────────┐
│ Customer        │  Guest or registered end-user
│ (EU market UX)  │
└─────────────────┘
```

### External actors

| Actor | Interaction |
|-------|-------------|
| **Customer (guest)** | Browse, configure, checkout without full account; local order snapshot |
| **Customer (registered)** | Auth session, address book, server-side order history, profile |
| **RMS Backend API** | Source of truth for catalog, fees, checkout, invoices |
| **Payment methods (via API)** | Selected at checkout; gateway execution owned by backend |
| **CDN / Nginx** | Static asset delivery for production builds |

---

## 3. Architectural Style

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Client type | SPA (Single-Page Application) | Fast navigation across catalog → cart → checkout |
| UI library | React 18 | Component model, ecosystem maturity |
| Routing | Declarative route table (`router.js`) + React Router | Clear ownership of screens |
| Data access | Thin API layer + custom hooks | Separation of UI from transport |
| Cart state | Client-side `localStorage` | Offline-tolerant browse/cart; sync at verify/checkout |
| Auth | Bearer token in `localStorage` | Simple session for B2C web; interceptor enforces 401 |
| Styling | Tailwind + CSS modules / styled-components | Rapid UI iteration with selective component styling |
| Deployment | Multi-stage Docker → Nginx | Immutable static artifacts, portable EU hosting |

---

## 4. Logical Architecture (Level 2)

```text
┌─────────────────────────────────────────────────────────────────┐
│ Presentation Layer                                              │
│  pages/*          Route screens (Home, Product, Cart, Checkout…)│
│  components/*     Feature UI (header, cart modals, forms…)      │
└────────────────────────────┬────────────────────────────────────┘
                             │ props / hooks / events
┌────────────────────────────▼────────────────────────────────────┐
│ Application Layer                                               │
│  Domain hooks & services (api/*.js)                             │
│  · useGetProducts · useGetproductDetailsData                    │
│  · useGetcheckoutData · useGetAddressData · useGetOrderHistory  │
│  · cart mutations · submitOrder · auth submit*                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│ Infrastructure Layer                                            │
│  utils/axios.js   HTTP client, auth headers, error interceptors │
│  api/index.js     Endpoint registry (BASE_URL + resource paths) │
│  localStorage     Cart, tokens, guest flags, consent, fees      │
└────────────────────────────┬────────────────────────────────────┘
                             │ HTTPS
┌────────────────────────────▼────────────────────────────────────┐
│ Remote System                                                   │
│  RMS REST API   /api/*                                          │
└─────────────────────────────────────────────────────────────────┘
```

### Layer responsibilities

| Layer | Owns | Must not own |
|-------|------|--------------|
| **Presentation** | Layout, interaction, accessibility, notifications | Raw HTTP URLs, persistence schema |
| **Application** | Use-cases, payload shaping, loading/error state | Visual markup |
| **Infrastructure** | Transport, headers, redirects, storage keys | Business pricing policy beyond client preview |
| **Backend (external)** | Authoritative prices, stock, invoices, auth | — |

---

## 5. Module View

| Module | Path | Responsibility |
|--------|------|----------------|
| Route host | `App.js`, `router.js` | Mount routes; global cookie consent |
| Catalog | `pages/Product`, `pages/productDetails`, `api/products`, `api/categories`, `api/productDetails` | Browse & detail |
| Search | `pages/SearchResults`, `api/useSearchProducts` | Name-based product search |
| Cart | `pages/cart`, `api/cart`, cart components | Line items, edit size/extras, totals |
| Checkout | `pages/checkout`, `api/checkout`, `AddressPopup` | Payment, address, delivery, submit |
| Orders | `pages/myorder`, `localorder.js`, `api/myorder` | History (API vs local) |
| Identity | `Login`, `SignUp`, password pages, `api/user` | Session establishment |
| Profile | `pages/profile`, `api/useGetProfile` | Customer account view |
| Content | Home sections, about, contact | Marketing & info surfaces |
| Compliance | `cookieConsent` | EN/DE consent capture |

---

## 6. Domain Model (Client Perspective)

```text
Category 1──* Subcategory 1──* Product
                                 │
                    ┌────────────┼────────────┐
                    ▼            ▼            ▼
                  Size        Extra      AllergenInfo
                    │            │
                    └─────┬──────┘
                          ▼
                   CartLineItem
                   (uniqueKey, qty, unitPrice, totalPrice)
                          │
                          ▼
                      Checkout
                   address + payment
                   + delivery_method
                   + promotion_code
                          │
                          ▼
                       Order
                    + Invoice
```

### Dynamic pricing (client preview)

```text
unitPrice   = sizePrice || product.basePrice
            + Σ(selectedExtra.price)

lineTotal   = unitPrice × quantity

cartSubtotal = Σ(lineTotal)
orderEstimate = cartSubtotal + deliveryFee − promotion (server-applied)
```

Authoritative totals and invoice amounts are confirmed by the backend at checkout.

### Cart identity

Lines are keyed by a composite `uniqueKey`:

```text
{productId}_{sizeId|default}_{sortedExtraIds|no_extras}
```

This allows the same base product to exist as multiple cart lines when configuration differs.

---

## 7. Key Use-Case Architectures

### 7.1 Authenticated order

1. Login → store `token`, `id`, `role`; clear guest flag  
2. Browse / configure → mutate cart in `localStorage`  
3. Checkout loads payment methods + address history (authenticated GET)  
4. Optional `verify-cart`  
5. `POST checkout-mobile` with `customer_id`, `address_id`, `payment_method_id`, `order_items`  
6. History via `GET orders/history` with Bearer token  

### 7.2 Guest order

1. Guest customer creation / flag `isGuest`  
2. Same cart model  
3. Address created and cached (`guestAddress`)  
4. Checkout submit  
5. Invoice/order snapshot stored locally for `/guestOrder`  

### 7.3 Session enforcement

Axios response interceptor:

- **401** → redirect to `/login` with `next` return path  
- **Network failure** → rejected promise with normalized message  

---

## 8. Data Ownership Matrix

| Data | Client | Backend |
|------|--------|---------|
| Product catalog | Cache in memory (hook state) | **Source of truth** |
| Cart lines | **Primary (localStorage)** | Verified at checkout |
| Auth token / customer id | Stored locally | Issues & validates |
| Addresses | Form UX + guest cache | **Persisted** |
| Delivery fees | Display cache | **Source of truth** |
| Promotion codes | Input only | Validation & discount |
| Invoices / order status | Display | **Source of truth** |
| Cookie consent | **localStorage** | Policy content (external) |

---

## 9. Cross-Cutting Concerns

### Security

- API key header on HTTP client (must be environment-managed in production)
- Bearer token for authenticated resources
- No payment card PAN handling in this SPA (method selection only)
- Avoid logging PII in production builds

### Privacy (EU)

- Cookie consent gate (English / German)
- Consent flag persisted client-side
- Minimize stored personal data for guests

### Performance

- SPA route transitions without full reload
- Image URLs resolved via shared `IMAGE_BASE_URL`
- Production assets served by Nginx with hashed CRA filenames

### Reliability

- Explicit loading states (`Loader`)
- User-facing notifications (Notiflix)
- Checkout submit guarded against double-submit (`isSubmitting`)

### Internationalization readiness

- Cookie copy already bilingual (EN/DE)
- UI strings primarily English; extend via i18n layer if multi-locale menu content is required

---

## 10. Deployment Architecture

```text
┌─────────────┐     ┌──────────────────┐     ┌─────────────────┐
│ GitLab CI   │────►│ Docker build     │────►│ Nginx container │
│ Auto DevOps │     │ node:16 → build  │     │ static /build   │
└─────────────┘     └──────────────────┘     └────────┬────────┘
                                                      │ :80
                                                      ▼
                                              Customer browsers
                                                      │
                                                      ▼
                                              RMS Backend API
```

| Environment concern | Approach |
|---------------------|----------|
| Build immutability | `npm run build` artifact copied into Nginx image |
| Portability | `Dockerfile` + `docker-compose.yml` |
| Pipeline | GitLab Auto DevOps (build, test, deploy, SAST, etc.) |

---

## 11. Quality Attributes (Target)

| Attribute | Architectural response |
|-----------|------------------------|
| **Usability** | Clear browse → customize → cart → checkout path |
| **Maintainability** | Route table, API registry, domain hooks |
| **Scalability (clients)** | Stateless SPA; scale Nginx horizontally |
| **Security** | Token auth, interceptor, secret externalization |
| **Observability** | Extend with RUM / error tracking at ops layer |
| **Compliance** | Cookie consent for EU market entry |

---

## 12. Constraints & Assumptions

1. Backend REST API is available and version-compatible with `src/api/index.js`.  
2. Cart is intentionally client-owned until checkout; multi-device cart sync is not in scope.  
3. Administrative / kitchen systems are separate deployables within the wider RMS.  
4. Multi-location catalog and fee rules are resolved server-side; this client renders what the API returns.  

---

## 13. Evolution Guidelines

| Change type | Preferred approach |
|-------------|-------------------|
| New endpoint | Register in `api/index.js`; add hook/service; keep pages thin |
| New screen | Add page + route entry in `router.js` |
| Pricing rules | Prefer server authority; client shows preview only |
| State complexity | Introduce dedicated state store only if cart/session outgrow `localStorage` |
| Secrets | Move base URL / keys to build-time env (`REACT_APP_*`) |

---

## 14. Related Artifacts

- [README.md](./README.md) — product overview & developer onboarding  
- [DIAGRAMS.md](./DIAGRAMS.md) — visual architecture & sequence diagrams  
- Source of truth for routes: `src/router.js`  
- Source of truth for API map: `src/api/index.js`  

---

*Document status: Baseline architecture for the customer ordering frontend. Update when deployment topology, auth model, or cart ownership strategy changes.*
