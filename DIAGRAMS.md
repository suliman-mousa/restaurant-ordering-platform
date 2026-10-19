
# Architecture Diagrams
## Restaurant Management System — Customer Ordering Frontend

| Field | Value |
|---|---|
| Document Type | Software Architecture Diagram Pack |
| Notation | Mermaid — C4-inspired context, flowcharts, sequence diagrams, and mind map |
| Architecture Scope | Customer-facing web frontend and its interactions with the RMS backend |
| Standards Reference | C4 Model concepts; ISO/IEC/IEEE 42010-inspired architecture views |
| Companion Documents | [README.md](./README.md) · [ARCHITECTURE.md](./ARCHITECTURE.md) |

> These diagrams describe the customer ordering frontend and its documented interactions with the RMS platform. Backend services, databases, payment integrations, and operational systems are external dependencies unless explicitly identified as part of this repository.
>
> Diagrams should be kept synchronized with the implementation. Architectural recommendations and intended improvements must not be interpreted as evidence that the corresponding behavior has already been implemented or tested.

---

## 1. System Context — C4 Level 1

This view identifies the customer, the frontend application, and the external systems involved in the ordering journey.

### 1.1 C4 Context View

```mermaid
C4Context
    title System Context - RMS Customer Ordering

    Person(customer, "Customer", "Guest or registered diner")
    System(spa, "Customer Ordering SPA", "React web application for browsing, cart management, checkout, and orders")
    System_Ext(api, "RMS Backend API", "Catalog, authentication, addresses, checkout, orders, and invoice data")
    System_Ext(pay, "Payment Providers", "External payment processing services")
    System_Ext(ops, "Operations and Admin Platform", "Restaurant, catalog, and location management")

    Rel(customer, spa, "Browses, configures, and submits orders", "HTTPS")
    Rel(spa, api, "Requests catalog and ordering operations", "HTTPS / REST JSON")
    Rel(api, pay, "Processes payment operations when applicable", "Provider APIs")
    Rel(ops, api, "Manages operational data", "HTTPS")
```

### 1.2 Portable Context View

Use this alternative if the target Mermaid renderer does not support the C4 diagram syntax.

```mermaid
flowchart LR
    C["Customer<br/>Guest or registered"] -->|"HTTPS"| SPA["Customer Ordering SPA<br/>React web client"]
    SPA -->|"HTTPS / REST JSON"| API["RMS Backend API"]
    API --> SVC["Backend Domain Services"]
    SVC --> DB[("Platform Database")]
    API -->|"When applicable"| PAY["Payment Providers"]
    OPS["Operations / Admin Platform"] -->|"HTTPS"| API
```

**Architectural boundary:** The browser-facing frontend initiates requests to the backend. Business-critical validation, authoritative order processing, and payment-related decisions belong to the backend and its integrated services.

---

## 2. Container Diagram — C4 Level 2

This view separates the browser runtime, static asset delivery, and backend platform responsibilities.

```mermaid
flowchart TB
    subgraph Browser["Customer Browser"]
        SPA["React SPA<br/>Pages, components, and client-side state"]
        LS[("Browser localStorage<br/>Cart, session-related data, guest data, consent")]
        SPA <--> LS
    end

    subgraph Delivery["Frontend Delivery"]
        NGX["Nginx<br/>Static asset delivery"]
    end

    subgraph Platform["RMS Backend Platform"]
        API["RMS Backend API<br/>REST endpoints"]
        SVC["Backend Domain Services<br/>Catalog, cart verification, checkout, orders"]
        DB[("Platform Database")]
        API --> SVC
        SVC --> DB
    end

    CUSTOMER["Customer"] -->|"HTTPS"| NGX
    NGX -->|"Serves built frontend assets"| SPA
    SPA -->|"HTTPS / REST JSON"| API
```

**Responsibilities and boundaries**

- **React SPA:** Renders the user interface, manages client-side interactions, and initiates API requests.
- **Browser localStorage:** Persists selected client-side data. It is not an authoritative source for order, payment, or server-side account state.
- **Nginx:** Serves the built frontend assets in the depicted deployment topology.
- **RMS Backend API:** Exposes the platform operations consumed by the frontend.
- **Backend services and database:** Own server-side business processing and persistent platform records.

The exact relationship between Nginx, the SPA build, and the deployed API must match the actual container and reverse-proxy configuration.

---

## 3. Component Diagram — Frontend Internals

This view illustrates the logical separation between presentation, application-facing API modules, and shared infrastructure.

```mermaid
flowchart TB
    subgraph Presentation["Presentation Layer"]
        ROUTER["router.js"]
        PAGES["pages/*"]
        COMPONENTS["components/*"]
        ROUTER --> PAGES
        PAGES --> COMPONENTS
    end

    subgraph Application["Application API Modules"]
        CART["api/cart.js"]
        CHECKOUT["api/checkout.js"]
        PRODUCTS["api/products.js<br/>api/productDetails.js"]
        ORDERS["api/myorder.js"]
        AUTH["api/user.js"]
        CATEGORIES["api/categories.js"]
    end

    subgraph Infrastructure["Infrastructure"]
        AXIOS["utils/axios.js"]
        API_INDEX["api/index.js"]
        STORAGE[("Browser localStorage")]
    end

    PAGES --> CART
    PAGES --> CHECKOUT
    PAGES --> PRODUCTS
    PAGES --> ORDERS
    PAGES --> AUTH
    PAGES --> CATEGORIES

    CART --> STORAGE
    CART --> AXIOS
    CHECKOUT --> AXIOS
    PRODUCTS --> AXIOS
    ORDERS --> AXIOS
    AUTH --> AXIOS
    CATEGORIES --> AXIOS

    AXIOS --> API_INDEX
    API_INDEX -->|"HTTPS / REST JSON"| BACKEND["RMS Backend API"]
```

**Interpretation**

- Presentation components handle rendering and user interaction.
- Application-facing modules organize operations by functional area.
- Shared infrastructure centralizes API communication and relevant client-side persistence.
- The diagram is a logical dependency view. Confirm actual imports and call paths against the repository before treating every arrow as a verified code dependency.

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
            Featured Dishes
        Product Configuration
            Sizes
            Extras and Options
            Allergens
            Dynamic Price Preview
        Commerce
            Cart
            Cart Verification
            Checkout
            Payment Methods
            Delivery Fees
            Promotion Codes
        Fulfillment
            Delivery and Pickup
            Addresses
            Cities
        Account
            Login and Signup
            Guest Customer
            Profile
            Password Recovery
        Orders
            Submit Order
            Invoice Data
            Registered Order History
            Guest Order History
        Privacy and Consent
            Cookie Consent
            English and German Localization
```

This map groups the documented frontend functionality by business domain. It does not imply that each domain is implemented as an independently deployable module.

---

## 5. End-to-End Ordering Flow

```mermaid
flowchart TD
    A["Home / Categories"] --> B["Product Listing"]
    B --> C["Product Details"]
    C --> D{"Product configuration required?"}

    D -->|"Yes"| E["Select size and extras"]
    D -->|"No"| F["Prepare cart item"]
    E --> G["Calculate client-side price estimate"]
    G --> F

    F --> H["Add or update cart"]
    H --> I["Persist cart locally"]
    I --> J["Review and edit cart"]
    J --> K["Enter checkout"]

    K --> L["Select fulfillment method"]
    K --> M["Select payment method"]
    K --> N["Provide delivery details when required"]

    L --> O["Submit checkout request"]
    M --> O
    N --> O

    O --> P{"Backend accepts request?"}
    P -->|"No"| Q["Display error and allow correction"]
    Q --> J

    P -->|"Yes"| R["Receive order response"]
    R --> S{"Registered customer?"}

    S -->|"Yes"| T["Access order history through backend"]
    S -->|"Guest"| U["Persist guest order snapshot locally, if supported"]

    R --> V["Display order confirmation and returned invoice data"]
```

**Important distinction:** A client-side price estimate is not the final authority for order totals. The backend must validate applicable prices, availability, promotions, fees, and order data according to the platform's business rules.

The precise order of validation, persistence, invoice generation, and payment processing must be documented from the actual backend contract.

---

## 6. Sequence — Browse, Customize, Add to Cart

```mermaid
sequenceDiagram
    autonumber
    actor U as Customer
    participant UI as Product Details UI
    participant Hook as Product Details Hook
    participant API as RMS Backend API
    participant Cart as Browser Cart Storage

    U->>UI: Open product details
    UI->>Hook: Request product data
    Hook->>API: GET /mobileProducts/{id}
    API-->>Hook: Product details and available configuration
    Hook-->>UI: Provide data for rendering

    UI-->>U: Display product and configuration options
    U->>UI: Select size, extras, and quantity
    UI->>UI: Calculate estimated unit and line totals

    U->>UI: Add item to cart
    UI->>Cart: Insert or update cart line
    Cart-->>UI: Complete local persistence operation
    UI-->>U: Display cart update feedback
```

**Notes**

- The product endpoint and response fields shown here must match the API implementation.
- The client may calculate a price preview for usability, but the backend remains authoritative for checkout totals.
- Cart-line merging depends on the application's actual identity rules for products, sizes, extras, and other configurable attributes.

---

## 7. Sequence — Checkout and Invoice

```mermaid
sequenceDiagram
    autonumber
    actor U as Customer
    participant CO as Checkout Page
    participant AX as Axios Client
    participant API as RMS Backend API
    participant LS as Browser localStorage

    U->>CO: Open checkout
    CO->>LS: Read cart and relevant guest data

    CO->>AX: Request available payment methods
    AX->>API: GET /payment-methods
    API-->>AX: Payment methods response
    AX-->>CO: Available payment methods

    alt Registered customer
        CO->>AX: Request saved addresses
        AX->>API: Authenticated address request
        API-->>AX: Address response
        AX-->>CO: Available saved addresses
    else Guest customer
        CO->>LS: Read previously saved guest address, if available
        LS-->>CO: Guest address data or empty result
    end

    U->>CO: Select fulfillment, payment, and order details
    U->>CO: Confirm checkout

    CO->>AX: Submit checkout request
    AX->>API: POST checkout request

    Note over API: Validate request and calculate authoritative totals

    alt Checkout accepted
        API-->>AX: Order response and available invoice data
        AX-->>CO: Successful checkout response

        CO->>LS: Update or clear cart
        opt Guest order persistence is supported
            CO->>LS: Save guest order snapshot
        end

        CO-->>U: Display order confirmation
    else Checkout rejected
        API-->>AX: Error response
        AX-->>CO: Checkout failure details
        CO-->>U: Display error and allow correction
    else Outcome is unknown due to a network failure
        AX--xCO: No usable response received
        CO-->>U: Explain that order status needs verification
    end
```

**Interpretation and safeguards**

- The backend response determines whether checkout was accepted.
- Invoice data is shown as part of the returned response only where supported by the actual API contract. This diagram does not assume that an invoice is necessarily finalized at the same moment as order creation.
- A network failure does not prove that the backend failed to create the order. Before retrying, the frontend should use a supported order-status lookup or idempotent submission mechanism where available.
- Cart clearing, guest order persistence, and navigation must follow the actual application behavior and should not discard data needed to recover from an uncertain outcome.
- Payment authorization, payment capture, and payment failure are not expanded into separate flows here because their exact lifecycle is not established by the frontend documentation alone.

---

## 8. Sequence — Authentication and Session Guard

```mermaid
sequenceDiagram
    autonumber
    actor U as Customer
    participant App as React SPA
    participant AX as Axios Client
    participant API as RMS Backend API
    participant LS as Browser localStorage

    U->>App: Submit login credentials
    App->>AX: Send login request
    AX->>API: POST /login

    alt Login successful
        API-->>AX: Authentication response
        AX-->>App: Token and user data, if returned
        App->>LS: Persist session-related values as implemented
        App-->>U: Show authenticated application state
    else Invalid credentials or rejected login
        API-->>AX: Authentication error
        AX-->>App: Login failure
        App-->>U: Display login error
    else Network or server failure
        AX--xApp: No usable response
        App-->>U: Display recoverable error
    end

    U->>App: Request protected resource
    App->>AX: Send request with Bearer token, if available
    AX->>API: Protected API request

    alt Request authorized
        API-->>AX: Resource response
        AX-->>App: Return resource data
    else Unauthorized or expired session
        API-->>AX: 401 Unauthorized
        AX-->>App: Authentication failure
        App-->>U: Redirect to login with safe return path
    else Forbidden
        API-->>AX: 403 Forbidden
        AX-->>App: Authorization failure
        App-->>U: Display access-denied state
    end
```

**Security considerations**

- A locally stored token is not proof that the session remains valid; the backend determines whether the request is authorized.
- Return paths must be validated to prevent unsafe external redirects.
- Token storage, expiration, refresh, revocation, and logout behavior must reflect the actual authentication implementation.
- Browser-accessible storage is exposed to JavaScript. If bearer tokens are stored in localStorage, cross-site scripting prevention is especially important.
- Do not assume that `401`, `403`, token refresh, or logout behavior is implemented exactly as illustrated without checking the interceptor and backend contract.

---

## 9. Dynamic Pricing Breakdown

```mermaid
flowchart TD
    subgraph Inputs["Pricing Inputs"]
        BASE["Product base price"]
        SIZE["Selected size price, if applicable"]
        EXTRAS["Selected extra prices"]
        QTY["Quantity"]
        DELIVERY["Delivery fee, when applicable"]
        PROMO["Promotion or discount"]
    end

    subgraph Preview["Client-Side Estimate"]
        SELECT["selectedBasePrice = sizePrice if applicable, otherwise productBasePrice"]
        EXTRA_TOTAL["extrasTotal = sum of selected extra prices"]
        UNIT["estimatedUnitPrice = selectedBasePrice + extrasTotal"]
        LINE["estimatedLineTotal = estimatedUnitPrice × quantity"]
        SUBTOTAL["estimatedCartSubtotal = sum of line totals"]
    end

    subgraph Authority["Backend Validation and Final Calculation"]
        VERIFY["Validate cart and applicable pricing rules"]
        TOTAL["Calculate authoritative order totals"]
        INVOICE["Return applicable invoice data"]
    end

    BASE --> SELECT
    SIZE --> SELECT
    EXTRAS --> EXTRA_TOTAL
    SELECT --> UNIT
    EXTRA_TOTAL --> UNIT
    UNIT --> LINE
    QTY --> LINE
    LINE --> SUBTOTAL

    SUBTOTAL --> VERIFY
    DELIVERY --> VERIFY
    PROMO --> VERIFY
    VERIFY --> TOTAL
    TOTAL --> INVOICE
```

**Pricing rules**

1. Select the applicable base price: the chosen size price when the product configuration requires it; otherwise, the product base price.
2. Add the prices of the selected extras.
3. Multiply the estimated unit price by the requested quantity to calculate the estimated line total.
4. Sum line totals to obtain the estimated cart subtotal.
5. Apply applicable delivery fees, promotions, taxes, rounding, and other business rules on the authoritative calculation path.

The exact treatment of taxes, discounts, rounding, and fee ordering must be specified by the backend's pricing contract. The diagram intentionally does not invent those rules.

---

## 10. Data Flow — Cart Line Identity

```mermaid
flowchart TD
    PRODUCT["Product ID"] --> KEY
    SIZE["Selected size ID or default-size representation"] --> KEY
    EXTRAS["Normalized selected extra IDs"] --> NORMALIZE["Normalize extra IDs"]
    NORMALIZE --> KEY

    KEY["Build deterministic cart-line key"]
    KEY --> EXISTS{"Matching cart line exists?"}

    EXISTS -->|"Yes"| MERGE["Apply documented quantity and line-update rules"]
    EXISTS -->|"No"| CREATE["Create new cart line"]

    MERGE --> SAVE[("Persist updated cart")]
    CREATE --> SAVE
```

**Identity requirements**

- The key must distinguish configurations that represent different orderable items.
- Selected extra IDs should be normalized deterministically, for example by sorting them when order does not affect meaning.
- The default-size representation must be consistent so that equivalent configurations produce equivalent keys.
- Quantity merging must follow the application's defined behavior.
- If configuration includes other price-affecting attributes, they must be considered in the identity rules when necessary.

The actual `uniqueKey` format should be documented from the cart implementation rather than inferred from the diagram.

---

## 11. Deployment Topology

The following view describes the intended delivery pattern. Replace component labels or connections where the actual CI configuration, Dockerfiles, or hosting environment differ.

```mermaid
flowchart TB
    subgraph Source["Source Control"]
        REPO["Frontend Repository"]
    end

    subgraph CI["CI Pipeline"]
        INSTALL["Install locked dependencies"]
        BUILD["Build frontend assets"]
        TEST["Run configured tests and quality checks"]
        PACKAGE["Package deployment artifact or image"]

        INSTALL --> BUILD
        BUILD --> TEST
        TEST --> PACKAGE
    end

    subgraph Runtime["Frontend Runtime"]
        NGINX["Nginx<br/>Serve static frontend assets"]
    end

    CUSTOMER["Customer Browser"]
    BACKEND["RMS Backend API"]

    REPO -->|"Pipeline trigger"| INSTALL
    PACKAGE -->|"Deploy"| NGINX
    CUSTOMER -->|"HTTPS"| NGINX
    CUSTOMER -->|"HTTPS API requests"| BACKEND
```

**Deployment requirements**

- Use a supported Node.js LTS release compatible with the repository's dependencies and build tooling.
- Pin runtime and build image versions appropriately for reproducible builds.
- Keep dependency installation aligned with the committed lockfile.
- Configure the production frontend's API base URL and other public build-time settings explicitly.
- Never place private API credentials or server secrets in browser-delivered environment variables or frontend bundles.
- Confirm whether the actual pipeline uses GitLab CI, Auto DevOps, Docker, or another deployment mechanism before describing any of them as verified infrastructure.
- Document HTTPS termination, API routing, environment configuration, health checks, rollback, and artifact promotion where applicable.

The diagram intentionally omits specific host ports and internal backend ports because those values must be verified against the actual deployment configuration.

---

## 12. Guest vs. Registered Customer Decision

```mermaid
flowchart TD
    START["Enter checkout"] --> SESSION{"Application identifies registered session?"}

    SESSION -->|"Yes"| ADDRESSES["Load registered customer addresses"]
    SESSION -->|"No"| GUEST["Collect guest details and address as required"]

    ADDRESSES --> FULFILLMENT["Select delivery or pickup"]
    GUEST --> FULFILLMENT

    FULFILLMENT --> PAYMENT["Select payment method"]
    PAYMENT --> SUBMIT["Submit checkout request"]

    SUBMIT --> RESULT{"Backend confirms order creation?"}

    RESULT -->|"No"| ERROR["Display error and preserve recoverable data"]
    ERROR --> FULFILLMENT

    RESULT -->|"Yes"| CUSTOMER_TYPE{"Registered or guest?"}

    CUSTOMER_TYPE -->|"Registered"| HISTORY["Navigate to registered order history when appropriate"]
    CUSTOMER_TYPE -->|"Guest"| GUEST_ORDER["Show guest order details or local snapshot, if supported"]
```

**Decision rules**

- Session classification must follow the application's actual authentication state and guest-mode logic.
- Token presence alone should not be treated as conclusive evidence of a valid authenticated session.
- Registered order history should use the backend's authorized order data.
- Locally persisted guest order data is a convenience, not a guaranteed or authoritative order record.
- Navigation should happen only after the checkout response establishes the appropriate outcome.

---

## 13. Route Topology

This view groups the documented routes by their primary purpose. Route names and access controls must be verified against the router configuration.

```mermaid
flowchart TB
    subgraph Public["Public Routes"]
        HOME["/"]
        PRODUCTS["/products"]
        DETAILS["/product/:id"]
        SEARCH["/search-results"]
        CART["/cart"]
        CHECKOUT["/create-order"]
        LOGIN["/login"]
        SIGNUP["/signup"]
    end

    subgraph Account["Account and Registered Customer Routes"]
        PROFILE["/profile"]
        ORDERS["/MyOrders"]
    end

    subgraph Guest["Guest Order Route"]
        GUEST_ORDER["/guestOrder"]
    end

    HOME --> PRODUCTS
    PRODUCTS --> DETAILS
    DETAILS --> CART
    SEARCH --> DETAILS
    CART --> CHECKOUT

    LOGIN --> PROFILE
    SIGNUP --> PROFILE

    CHECKOUT --> RESULT{"Checkout result"}
    RESULT -->|"Accepted, registered customer"| ORDERS
    RESULT -->|"Accepted, guest customer"| GUEST_ORDER
```

**Route interpretation**

- Route grouping does not itself establish whether a route is public, protected, or conditionally accessible.
- Login redirects and protected-route guards should preserve only validated internal return paths.
- The checkout result determines the appropriate post-checkout destination; the diagram does not imply that both order destinations are visited.
- Confirm route casing, route parameters, fallback routes, and redirect behavior against the actual application router.

---

## 14. Diagram Maintenance and Change Control

Update the relevant diagrams whenever implementation changes affect the system's structure, responsibilities, or runtime behavior.

| Change | Diagrams to Review |
|---|---|
| New external system or integration | §1 System Context |
| New runtime container or deployment boundary | §2 Container Diagram, §11 Deployment |
| New frontend module or dependency | §3 Component Diagram, §4 Module Map |
| Changes to the customer ordering journey | §5 End-to-End Ordering Flow |
| Product configuration or cart behavior | §4 Module Map, §6 Browse Sequence, §10 Cart Identity |
| Checkout, invoice, or payment contract | §5 End-to-End Flow, §7 Checkout Sequence, §9 Pricing |
| Authentication or session handling | §8 Authentication Sequence, §12 Guest vs. Registered |
| Pricing, promotions, taxes, or fees | §5 End-to-End Flow, §7 Checkout Sequence, §9 Pricing |
| Route additions or access-control changes | §8 Authentication Sequence, §12 Guest vs. Registered, §13 Route Topology |
| CI, Docker, hosting, or runtime changes | §2 Container Diagram, §11 Deployment |

### Review Checklist

Before merging changes to this document:

- [ ] Validate Mermaid syntax using the target GitHub renderer.
- [ ] Confirm route names and endpoint paths against the source code.
- [ ] Confirm sequence participants and message directions.
- [ ] Ensure frontend calculations are distinguished from backend-authoritative decisions.
- [ ] Verify error, retry, and uncertain-outcome behavior against the implementation.
- [ ] Confirm deployment diagrams match the actual CI and runtime configuration.
- [ ] Remove obsolete components, endpoints, and flows.
- [ ] Ensure all companion documentation remains consistent.

### Architecture Documentation Note

This document uses C4-inspired context and container views, component-level dependency diagrams, and UML-style sequence diagrams. Its references to C4 and ISO/IEC/IEEE 42010 describe the documentation approach; they do not, by themselves, establish formal standards compliance.

---

*End of Architecture Diagram Pack.*
