# Architecture Diagrams  
## Restaurant Management System — Customer Ordering Frontend

| Field | Value |
|-------|--------|
| **Notation** | Mermaid (C4-inspired · UML sequence · flowcharts) |
| **Standard alignment** | C4 Model · ISO/IEC/IEEE 42010 views |
| **Companion docs** | [README.md](./README.md) · [ARCHITECTURE.md](./ARCHITECTURE.md) |

> Render these diagrams in any Mermaid-compatible viewer (GitLab, GitHub, VS Code Mermaid extension, Notion, etc.).

---

## 1. System Context (C4 Level 1)

```mermaid
C4Context
  title System Context — RMS Customer Ordering

  Person(customer, "Customer", "Guest or registered diner (European market)")
  System(spa, "Customer Ordering SPA", "React web client: menu, cart, checkout, orders")
  System_Ext(api, "RMS Backend API", "Catalog, auth, addresses, payments, orders, invoices")
  System_Ext(pay, "Payment Providers", "Settled via backend payment methods")
  System_Ext(ops, "Ops / Admin Platform", "Multi-location operations (out of this repo)")

  Rel(customer, spa, "Browses, orders, tracks", "HTTPS")
  Rel(spa, api, "REST JSON", "HTTPS + API key / Bearer")
  Rel(api, pay, "Charge / authorize", "Provider APIs")
  Rel(ops, api, "Configures catalog & locations", "HTTPS")
```

### Context (portable flowchart)

```mermaid
flowchart LR
  C[Customer<br/>Guest / Registered] -->|HTTPS| SPA[Customer Ordering SPA<br/>React + Nginx]
  SPA -->|REST / JSON| API[RMS Backend API]
  API --> DB[(Platform Data Store)]
  API --> PAY[Payment Gateways]
  ADMIN[Admin / Ops Systems] --> API
```

---

## 2. Container Diagram (C4 Level 2)

```mermaid
flowchart TB
  subgraph Browser["Customer Browser"]
    SPA["React SPA<br/>pages · components · hooks"]
    LS["localStorage<br/>cart · token · guest · consent"]
    SPA <--> LS
  end

  subgraph Edge["Delivery"]
    NGX["Nginx<br/>static build assets"]
  end

  subgraph Platform["RMS Platform"]
    API["REST API<br/>/api/*"]
    SVC["Domain Services<br/>Catalog · Cart Verify · Checkout · Orders"]
    DATA[(Database)]
    API --> SVC --> DATA
  end

  C[Customer] --> NGX --> SPA
  SPA -->|Axios HTTPS| API
```

---

## 3. Component Diagram — Frontend Internals

```mermaid
flowchart TB
  subgraph Presentation
    R[router.js]
    P[pages/*]
    UI[components/*]
    R --> P --> UI
  end

  subgraph Application
    CART[api/cart.js]
    CHK[api/checkout.js]
    PRD[api/products.js · productDetails.js]
    ORD[api/myorder.js]
    AUTH[api/user.js]
    CAT[api/categories.js]
  end

  subgraph Infrastructure
    AX[utils/axios.js]
    REG[api/index.js]
    STORE[(localStorage)]
  end

  P --> CART & CHK & PRD & ORD & AUTH & CAT
  CART --> STORE
  CHK --> AX
  PRD --> AX
  ORD --> AX
  AUTH --> AX
  CAT --> AX
  AX --> REG
  AX -->|HTTPS| API[RMS Backend API]
```

---

## 4. Module Map by Business Domain

```mermaid
mindmap
  root((RMS Customer SPA))
    Catalog
      Categories
      Subcategories
      Products
      Search
      Offers
      Best Dishes
    Configuration
      Sizes
      Extras / Options
      Allergens
      Dynamic Price Preview
    Commerce
      Cart
      Verify Cart
      Checkout
      Payment Methods
      Delivery Fees
      Promotion Codes
    Fulfillment
      Delivery / Pickup
      Addresses
      Cities
    Account
      Login / Signup
      Guest Customer
      Profile
      Password Recovery
    Orders
      Submit Order
      Invoice
      My Orders
      Guest Local Orders
    Compliance
      Cookie Consent EN/DE
```

---

## 5. End-to-End Ordering Flow

```mermaid
flowchart TD
  A[Home / Categories] --> B[Product List]
  B --> C[Product Details]
  C --> D{Configure<br/>size + extras?}
  D -->|Yes| E[Compute dynamic unit & line price]
  D -->|No| E
  E --> F[Add to Cart<br/>localStorage]
  F --> G[Cart Review / Edit]
  G --> H[Checkout]
  H --> I[Select address]
  H --> J[Select payment]
  H --> K[Delivery or Pickup]
  I & J & K --> L[Submit Order]
  L --> M{Authenticated?}
  M -->|Yes| N[Order History API]
  M -->|Guest| O[Local Order Snapshot]
  L --> P[Invoice metadata]
```

---

## 6. Sequence — Browse, Customize, Add to Cart

```mermaid
sequenceDiagram
  autonumber
  actor U as Customer
  participant UI as Product Details UI
  participant Hook as useGetproductDetailsData
  participant API as RMS API
  participant Cart as localStorage Cart

  U->>UI: Open /product/:id
  UI->>Hook: load product
  Hook->>API: GET /mobileProducts/{id}
  API-->>Hook: product + sizes + extras + allergens
  Hook-->>UI: render configurator
  U->>UI: Select size, toggle extras, set qty
  UI->>UI: unitPrice = size/base + Σ extras
  U->>UI: Add to cart
  UI->>Cart: upsert line by uniqueKey
  Cart-->>UI: persisted
  UI-->>U: success notification → /cart
```

---


## 7. Sequence — Checkout & Invoice

```mermaid
sequenceDiagram
    autonumber
    actor U as Customer
    participant UI as React Frontend
    participant API as RMS Backend API
    participant DB as Backend Database

    U->>UI: Confirm checkout
    UI->>API: GET payment methods
    API-->>UI: Available payment methods

    alt Registered customer
        UI->>API: GET saved addresses
        API-->>UI: Saved addresses
    else Guest customer
        UI->>UI: Read guest address from form or local storage
    end

    U->>UI: Submit order details
    UI->>API: POST checkout request
    API->>API: Validate request and recalculate totals
    API->>DB: Create order and related records
    DB-->>API: Persisted order data
    API-->>UI: Order response and invoice data

    alt Order created successfully
        UI->>UI: Update cart state
        alt Guest customer
            UI->>UI: Save guest order snapshot locally
        else Registered customer
            UI->>UI: Prepare order history navigation
        end
        UI-->>U: Show order confirmation
    else Request rejected or failed
        API-->>UI: Error response
        UI-->>U: Show error and allow retry
    end
```


  U->>CO: Select address, payment, delivery/pickup, promo
  CO->>AX: POST checkout-mobile
  Note over AX,API: payload: customer_id, address_id,<br/>payment_method_id, order_items, promotion_code
  API-->>CO: order + invoice
  CO->>LS: clear/update cart; store guest order if needed
  CO-->>U: confirmation / navigate to orders
```

---

## 8. Sequence — Authentication & Session Guard

```mermaid
sequenceDiagram
  autonumber
  actor U as Customer
  participant App as SPA
  participant AX as Axios Interceptor
  participant API as RMS API

  U->>App: Login
  App->>API: POST /login
  API-->>App: token + user.customer.id + role
  App->>App: localStorage token, id; clear isGuest

  U->>App: Request protected resource
  App->>AX: GET with Bearer
  AX->>API: authorized call

  alt 401 Unauthorized
    API-->>AX: 401
    AX-->>App: redirect /login?next=...
  else Success
    API-->>App: payload
  end
```

---

## 9. Dynamic Pricing Breakdown

```mermaid
flowchart LR
  subgraph Inputs
    BP[Base / Size Price]
    EX[Selected Extras Prices]
    Q[Quantity]
    DF[Delivery Fee]
    PR[Promotion Code]
  end

  subgraph Client Preview
    UP[unitPrice = BP + Σ EX]
    LT[lineTotal = unitPrice × Q]
    ST[cartSubtotal = Σ lineTotal]
  end

  subgraph Server Authority
    VT[verify-cart / checkout]
    INV[Invoice totals]
  end

  BP --> UP
  EX --> UP
  UP --> LT
  Q --> LT
  LT --> ST
  ST --> VT
  DF --> VT
  PR --> VT
  VT --> INV
```

---

## 10. Data Flow — Cart Line Identity

```mermaid
flowchart TD
  P[Product ID] --> K
  S[Size ID or default] --> K
  E[Sorted Extra IDs or no_extras] --> K
  K["uniqueKey =<br/>product_size_extras"]
  K --> L{Line exists?}
  L -->|Yes| INC[Increment qty & total]
  L -->|No| NEW[Push new CartLineItem]
  INC --> LS[(localStorage.cart)]
  NEW --> LS
```

---

## 11. Deployment Topology

```mermaid
flowchart LR
  subgraph CI["GitLab CI / Auto DevOps"]
    B[build] --> T[test / SAST]
    T --> D[deploy]
  end

  subgraph Image["Container Image"]
    N16[node:16-alpine<br/>npm ci + npm run build]
    NGX[nginx:alpine<br/>/usr/share/nginx/html]
    N16 --> NGX
  end

  D --> Image
  NGX -->|:80 → host :3000| WEB[Customers]
  WEB -->|API calls| BE[RMS Backend :8000/api]
```

---

## 12. Guest vs Registered Decision

```mermaid
flowchart TD
  Start[Enter Checkout] --> Q{token present<br/>and not isGuest?}
  Q -->|Registered| A1[Load address history from API]
  Q -->|Guest| A2[Use / create guestAddress]
  A1 --> Pay[Select payment + delivery]
  A2 --> Pay
  Pay --> Sub[POST checkout-mobile]
  Sub --> Q2{Registered?}
  Q2 -->|Yes| H1[/MyOrders via API/]
  Q2 -->|No| H2[/guestOrder via localStorage/]
```

---

## 13. Route Topology

```mermaid
flowchart LR
  subgraph Public
    H["/"]
    PR["/products"]
    PD["/product/:id"]
    SR["/search-results"]
    CA["/cart"]
    CK["/create-order"]
    LI["/login"]
    SU["/signup"]
  end

  subgraph Session
    PF["/profile"]
    MO["/MyOrders"]
  end

  subgraph GuestPersist
    GO["/guestOrder"]
  end

  H --> PR --> PD --> CA --> CK
  CK --> MO
  CK --> GO
  LI --> PF
```

---

## 14. How to Maintain These Diagrams

| When you change… | Update diagram(s) |
|------------------|-------------------|
| New external system | §1 Context |
| New deployable | §2 Containers, §11 Deployment |
| New major module | §3 Components, §4 Mind map |
| Checkout / cart rules | §5–§7, §9–§10 |
| Auth model | §8, §12 |
| Routes | §13 |

Keep diagrams **as code** in this file so architecture reviews stay synchronized with the repository.

---

*End of diagram pack.*
