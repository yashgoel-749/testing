# Car Dealership Inventory System — Senior Software Engineer Interview Master Guide

This comprehensive guide prepares you for senior technical interviews regarding the **Car Dealership Inventory System**. It includes exhaustive breakdowns of every architectural layer, design choice, edge case, security mechanism, scalability constraint, and AI workflow.

---

## 1. PROJECT OVERVIEW

### 2-Minute Elevator Pitch
"The Car Dealership Inventory System is a full-stack, type-safe web application designed to streamline vehicle inventory management and retail transactions for car dealerships. 

On the backend, I built a RESTful API using Node.js, Express, and TypeScript, backed by PostgreSQL hosted on Neon and managed via Prisma ORM. The API uses JSON Web Tokens (JWT) for stateless authentication and implements Role-Based Access Control (RBAC) to enforce administrative privileges for inventory creation, updates, restocking, and deletion.

On the frontend, I built a Single Page Application (SPA) using React 18, Vite, and TypeScript, styled with Tailwind CSS. Global state—including user authentication tokens and cached vehicle listings—is managed cleanly via Redux Toolkit. The UI dynamically adapts based on user roles: regular buyers can search, filter, and purchase vehicles, while administrators get an inventory control panel.

The entire project was developed following strict Test-Driven Development (TDD) principles. I wrote Supertest integration test suites for the backend and Jest/React Testing Library component tests for the frontend before writing feature code. This ensured high test coverage, zero regression during refactoring, and clear boundary contracts."

---

### 30-Second Pitch
"I built a full-stack, TDD-driven Car Dealership Inventory System using React 18, Redux Toolkit, Node.js, Express, Prisma, and PostgreSQL. It features JWT-based role-based access control (Admin vs. Buyer), real-time search/filtering, atomic vehicle purchasing logic, and a fully typed architecture from database to UI components."

---

### Architecture & Request Lifecycle

#### Overall System Architecture
```
┌────────────────────────────────────────────────────────────────────────┐
│                          React 18 SPA Client                           │
│  [React Router v7] ──► [Redux Toolkit Store] ──► [Axios Interceptor]   │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │ HTTPS (JSON + Bearer JWT Header)
┌──────────────────────────────────▼─────────────────────────────────────┐
│                          Express HTTP Server                           │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Middlewares: cors -> express.json() -> auth -> validate(Zod)     │  │
│  └───────────────────────────────┬──────────────────────────────────┘  │
│                                  │                                     │
│  ┌───────────────────────────────▼──────────────────────────────────┐  │
│  │ Controllers (HTTP Request parsing & Response formatting)          │  │
│  └───────────────────────────────┬──────────────────────────────────┘  │
│                                  │                                     │
│  ┌───────────────────────────────▼──────────────────────────────────┐  │
│  │ Services (Business Logic, Auth, Domain Validations)               │  │
│  └───────────────────────────────┬──────────────────────────────────┘  │
│                                  │                                     │
│  ┌───────────────────────────────▼──────────────────────────────────┐  │
│  │ Repositories (Prisma Client Data Access Layer)                   │  │
│  └───────────────────────────────┬──────────────────────────────────┘  │
└──────────────────────────────────┼─────────────────────────────────────┘
                                   │ Prisma Client (SQL Query Protocol)
┌──────────────────────────────────▼─────────────────────────────────────┐
│                       Neon PostgreSQL Database                         │
│                  [User Table]       [Vehicle Table]                    │
└────────────────────────────────────────────────────────────────────────┘
```

#### Step-by-Step Request Lifecycle (e.g., `POST /api/vehicles/:id/purchase`)

1. **Client Trigger:** User clicks "Purchase Vehicle" in React card component (`VehicleCard.tsx`).
2. **Redux Dispatch:** Component dispatches `purchaseVehicleThunk({ id, quantity })`.
3. **HTTP Client:** Axios instance attaches `Authorization: Bearer <jwt_token>` from `sessionStorage` and sends `POST /api/vehicles/5/purchase`.
4. **Express Routing:** Request hits `vehicle.routes.ts`.
5. **Auth Middleware (`authenticate`):** Extracts JWT from `Authorization` header, verifies signature using `jwt.verify(token, JWT_SECRET)`, attaches decoded user payload (`req.user = { id, email, role }`) to request object. If invalid/missing, short-circuits with `401 Unauthorized`.
6. **Validation Middleware (`validate(purchaseSchema)`):** Validates path param `id` and body `{ quantity }` using Zod schema. If invalid, short-circuits with `400 Bad Request`.
7. **Controller (`purchaseVehicle`):** Wrapped in `catchAsync` helper. Extracts `id` and `quantity`, invokes `vehicleService.purchaseVehicle(id, quantity)`.
8. **Service Layer (`purchaseVehicle`):** Validates business preconditions (e.g., quantity > 0), delegates database execution to `vehicleRepository.purchase(id, quantity)`.
9. **Repository Layer (`purchase`):** Executes a Prisma interactive transaction (`prisma.$transaction`).
   - Fetches vehicle inside transaction (`tx.vehicle.findUnique`).
   - Checks if vehicle exists (if not, throws `AppError(404)`).
   - Checks if `stock >= quantity` (if not, throws `AppError(400)`).
   - Computes `newStock = stock - quantity`. Updates vehicle with `stock: newStock` and sets `status: 'SOLD'` if `newStock === 0`.
10. **Global Error Handler (if exception thrown):** Captured by `catchAsync`, passed to `errorHandler` middleware, formatted into `{ success: false, message: "..." }` with corresponding HTTP status.
11. **HTTP Response:** Controller returns `200 OK` with updated vehicle object.
12. **Redux Store & UI Update:** Redux slice receives fullfilled action, updates state, triggers React re-render.

---

### Why Built This Way?
- **Controller-Service-Repository Pattern:** Decouples Express HTTP infrastructure from business rules and database queries. Enables isolated unit testing of business logic without hitting the DB.
- **Prisma Interactive Transactions:** Guarantees atomicity when checking and updating stock levels.
- **Redux Toolkit over Context API:** Provides predictable, centralized state management, clean handling of async thunk lifecycles (pending, fulfilled, rejected), and easy testing via pure reducer functions.
- **Zod Schema Validation:** Guarantees strict runtime boundary typing at both HTTP entry points (backend) and form submissions (frontend).

---

### Biggest Learning & Future Improvements (If Given 1 Extra Week)

#### Biggest Learning:
"Developing TDD-first completely transformed how I structure system boundaries. When writing integration tests before code, you are forced to design clean contracts, predictable error responses, and deterministic database isolation early on, rather than retrofitting tests onto coupled code."

#### What to Improve with 1 Extra Week:
1. **Audit & Order Tracking Schema:** Introduce a dedicated `Order` / `PurchaseHistory` entity linked via foreign keys to `User` and `Vehicle`. Currently, purchasing only decrements `stock` on `Vehicle` without recording transaction logs or buyer user IDs.
2. **True Pessimistic / Optimistic Database Locking:** Replace the standard read-then-write transaction with atomic SQL updates (`UPDATE vehicle SET stock = stock - q WHERE id = id AND stock >= q`) or Postgres row-level locks (`SELECT FOR UPDATE`) to prevent race conditions under high concurrency.
3. **Refresh Token Flow & HTTP-Only Cookies:** Shift JWT storage from `sessionStorage` (vulnerable to XSS) to HTTP-Only `SameSite=Strict` cookies, with short-lived access tokens and refresh tokens in database sessions.
4. **Pagination & Server-Side Filtering:** Implement offset/cursor pagination for vehicle listings on the backend instead of loading all vehicles into memory.

---

## 2. DESIGN DECISIONS

| Choice Made | Alternative Considered | Why This Choice? | Key Tradeoff |
|---|---|---|---|
| **Express.js** | NestJS | Lightweight, zero boilerplate, total control over routing and middleware architecture. Ideal for medium-sized microservices. | Lacks NestJS's out-of-the-box opinionated dependency injection container and module structure. |
| **React 18 + Vite** | Next.js / Angular | Vite provides instantaneous HMR and fast builds. Client-Side Rendering (CSR) fits an authenticated dashboard app where SEO on user dashboards isn't required. | No Server-Side Rendering (SSR) out of the box; client must download JS bundle before initial render. |
| **PostgreSQL (Neon)** | MongoDB | Car inventory and user accounts are highly structured relational data requiring strict table schemas, foreign key integrity, and ACID transactions. | Schema migrations require explicit management (Prisma Migrations) compared to schemaless MongoDB. |
| **JWT Authentication** | Cookie Sessions (Express-Session + Redis) | Stateless authentication eliminates server-side session lookup overhead and scales horizontally across multiple backend instances without session affinity. | Cannot revoke active JWTs instantly before expiration unless a token blacklist (e.g., Redis) is maintained. |
| **bcryptjs** | Argon2 / PBKDF2 | Industry standard for password hashing with configurable work factor (salt rounds), built-in salt generation, and zero native binary build dependencies. | Slightly slower in JS compared to native C bindings (`bcrypt`), but avoids native compilation issues across OS platforms. |
| **Tailwind CSS** | CSS Modules / Styled Components | Utility-first styling enables rapid UI iteration, zero unused CSS in production builds, consistent design tokens (colors, spacing), and zero runtime CSS-in-JS performance penalty. | Class names can clutter JSX templates if not extracted into reusable components. |
| **Controller-Service-Repository** | Monolithic Route Handlers | Separation of Concerns (SoC). Controllers handle HTTP/express params, Services handle business domain logic, Repositories execute database queries. | Adds file boilerplate for small CRUD operations. |
| **REST API** | GraphQL | Simple, standard HTTP methods (GET, POST, PUT, DELETE), easy caching at network edge, explicit status codes (200, 400, 401, 404, 500), fast setup. | Over-fetching or under-fetching data if endpoints return fixed payloads. |
| **Prisma ORM** | TypeORM / Raw SQL (pg) | Auto-generated type-safe database client synced with `schema.prisma`, declarative migrations, built-in connection pooling support with Neon. | Abstraction layer can generate sub-optimal SQL queries for complex multi-join reporting queries compared to raw SQL. |

---

## 3. REAL ENGINEERING CHALLENGES & DEBUGGING

### 1. Database Connection Spikes During Integration Testing (Jest)
- **Challenge:** Running `jest --watch` or parallel integration tests opened dozens of concurrent Prisma DB client instances, exceeding Neon PostgreSQL serverless connection limits and causing `FATAL: too many connections` errors.
- **Root Cause:** Instantiating `new PrismaClient()` inside repository or test files created fresh connection pools per test suite execution.
- **Solution:** Implemented the Prisma Singleton pattern in `src/config/db.ts`, attaching the Prisma Client instance to `globalThis` in non-production environments to reuse a single instance across test runs.

### 2. React Redux State Hydration vs. SessionStorage Corruption
- **Challenge:** When users refreshed the browser page, Redux state cleared. Restoring auth state from `sessionStorage` resulted in app crashes if stored JSON became corrupted or expired.
- **Solution:** Implemented robust JSON parsing inside `authSlice.ts` initialization with fallback try-catch blocks to safely clear invalid storage strings and reset state to unauthenticated defaults. Added comprehensive unit tests targeting corrupted storage handling.

### 3. Zod Input Coercion for HTTP Query Parameters
- **Challenge:** Query parameters in Express (`req.query`) are received as raw strings (e.g., `minPrice="5000"`). Zod numeric validation (`z.number()`) failed validation with `Expected number, received string`.
- **Solution:** Used `z.coerce.number()` in Zod search schemas to automatically transform string query parameters into numbers before running validation checks.

### 4. What AI Generated Incorrectly & How It Was Fixed
- **Issue:** AI (Antigravity/ChatGPT) initially generated Prisma transaction logic that used `prisma.vehicle.update` outside the interactive transaction callback `tx`.
- **Impact:** The code was not actually atomic and suffered from race conditions.
- **Fix:** Refactored the repository method to explicitly use the transactional client parameter `tx.vehicle.update` and `tx.vehicle.findUnique` inside `prisma.$transaction(async (tx) => { ... })`.

---

## 4. LIMITATIONS & TECHNICAL DEBT

| Domain | Current Limitation | Technical Debt / Weakness | Senior Engineering Fix |
|---|---|---|---|
| **Data Model** | No `Order` / `PurchaseHistory` table. | Purchases only update vehicle stock; buyer user ID and transaction timestamp are lost. | Add `Order` model in Prisma schema (`id`, `userId`, `vehicleId`, `quantity`, `totalPrice`, `createdAt`) with standard foreign key constraints. |
| **Concurrency** | Read-then-write stock check in transaction. | No row lock (`FOR UPDATE`). Two parallel transactions can read `stock=1` concurrently and both succeed. | Execute atomic SQL update: `UPDATE "Vehicle" SET stock = stock - $1, status = CASE WHEN stock - $1 = 0 THEN 'SOLD' ELSE status END WHERE id = $2 AND stock >= $1`. |
| **Authentication** | JWT stored in `sessionStorage`. | Susceptible to Cross-Site Scripting (XSS) if client script is compromised. | Move JWT to `HTTP-Only`, `SameSite=Strict`, `Secure` cookies. Implement refresh token rotation. |
| **Pagination** | `findAll()` fetches all records into memory. | Server memory usage and response latency will spike as vehicle table grows to 100k+ rows. | Implement offset/cursor pagination (`take`, `skip` or `cursor` in Prisma) returning `{ data: [...], meta: { page, totalPages, totalItems } }`. |
| **Security** | No rate limiting middleware on login/API endpoints. | Vulnerable to credential stuffing and brute-force password guessing. | Add `express-rate-limit` with Redis store (`rate-limit-redis`) to restrict IP requests (e.g., max 5 login attempts per minute). |
| **Search Engine** | SQL `LIKE` / `contains` text search. | Full table scans for text searches without specialized GIN/GiST indexes; poor search relevance. | Integrate PostgreSQL Full Text Search (`tsvector` / `tsquery`) or offload search index to Elasticsearch / Algolia. |
| **Logging** | Basic `console.error` logs. | No structured log formats, log levels, request correlation IDs, or centralized aggregator. | Replace console logs with `Winston` or `Pino` emitting structured JSON logs with correlation IDs (`x-request-id`). |

---

## 5. PRODUCTION READINESS & SCALABILITY (100,000 USERS)

### What Fails First Under High Load?
1. **Database Connection Pool Exhaustion:** Postgres serverless connections will saturate as concurrent request threads attempt to acquire Prisma connections.
2. **Backend Memory Overhead:** Endpoints like `GET /api/vehicles` returning unbounded JSON payloads will cause Node.js heap allocations to spike, leading to Event Loop lag and Out-Of-Memory (OOM) crashes.
3. **Database I/O Bottleneck:** Sequential table scans on unindexed columns (e.g., `make`, `model`, `price`) during heavy search operations will max out CPU and disk I/O.

### How to Scale to 100,000 Users

#### Architecture Migration Diagram
```
                              ┌────────────────────────┐
                              │  CloudFront CDN (Edge) │
                              └───────────┬────────────┘
                                          │
                              ┌───────────▼────────────┐
                              │ Application Load Balancer│
                              └───────────┬────────────┘
                                          │
            ┌─────────────────────────────┼─────────────────────────────┐
            │                             │                             │
  ┌─────────▼──────────┐        ┌─────────▼──────────┐        ┌─────────▼──────────┐
  │ Backend Instance 1 │        │ Backend Instance 2 │        │ Backend Instance N │
  └─────────┬──────────┘        └─────────┬──────────┘        └─────────┬──────────┘
            │                             │                             │
            └─────────────────────────────┼─────────────────────────────┘
                                          │
                      ┌───────────────────┴───────────────────┐
                      │                                       │
            ┌─────────▼──────────┐                  ┌─────────▼──────────┐
            │   Redis Cache      │                  │ AWS SQS Message    │
            │  (Listings/Auth)   │                  │     Queue          │
            └─────────┬──────────┘                  └─────────┬──────────┘
                      │                                       │
            ┌─────────▼──────────┐                  ┌─────────▼──────────┐
            │ Primary Postgres DB│                  │ Worker Service     │
            │  (Write Engine)    │                  │(Async Inventory/  │
            └─────────┬──────────┘                  │ Order Processing)  │
                      │ Read Replication            └────────────────────┘
            ┌─────────▼──────────┐
            │ Read Replica DB 1  │
            └────────────────────┘
```

#### Scaling Strategies:
1. **Caching Layer (Redis):** Cache high-read endpoints (e.g., vehicle list, search filters) in Redis with a 5-minute TTL. Invalidate cache tags when an admin creates/updates/deletes a vehicle.
2. **Database Read Replicas & Connection Pooling:** 
   - Separate Read and Write database traffic. Route search/read traffic to PostgreSQL Read Replicas and write/purchase traffic to the Primary DB instance.
   - Use PgBouncer or Neon Connection Pooling to manage thousands of incoming database clients.
3. **Asynchronous Queues (BullMQ / SQS):** Offload heavy tasks (email notifications, purchase receipt generation, inventory analytics) to background workers using message queues.
4. **Stateless Horizontal Scaling:** Deploy Node.js containers across AWS ECS / EKS behind an Application Load Balancer (ALB) with Auto-Scaling Policies based on CPU/RAM metrics.
5. **Monitoring & Observability:**
   - **Metrics:** Prometheus + Grafana for CPU, RAM, throughput, and HTTP error rate tracking.
   - **Tracing:** OpenTelemetry + Jaeger / Datadog for distributed request tracing.
   - **Error Tracking:** Sentry for frontend and backend runtime exception capture.

---

## 6. EDGE CASE ANALYSIS & PRODUCTION HANDLING

| Edge Case Scenario | Current Code Behavior | Production-Grade Standard |
|---|---|---|
| **Simultaneous Vehicle Purchase (Race Condition)** | Prisma transaction reads `stock`, checks `stock >= q`, updates `stock`. Under high concurrency, without row locks, both reads can overlap, leading to negative stock or overbooking. | Use atomic SQL decrement with guard clause: `UPDATE vehicle SET stock = stock - 1 WHERE id = 1 AND stock >= 1`. If affected rows == 0, return `409 Conflict (Out of Stock)`. |
| **Admin Deletes Vehicle Mid-Purchase** | Transaction attempts `findUnique(id)` inside purchase. Returns `404 Not Found`. | Soft Deletion (`deletedAt` timestamp). Active checkout locks the row; soft-deleted records remain accessible to existing purchase transactions until committed. |
| **JWT Expires During Checkout** | Auth middleware rejects request with `401 Unauthorized` (`jwt.verify` fails). Client displays error toast. | Axios response interceptor intercepts 401, invokes `/api/auth/refresh` using HTTP-Only refresh cookie, gets new access token, and transparently retries checkout request. |
| **Database Connection Failure** | Server throws 500 internal error. Express global error handler returns generic error payload. | Implement Circuit Breaker pattern (e.g., Opossum library). Retry connection with exponential backoff and jitter. Return graceful static fallback response. |
| **Negative / Invalid Purchase Quantity** | Prevented by Zod validation middleware (`z.number().int().positive()`) returning `400 Bad Request`. | Multi-layer validation: Zod middleware on HTTP boundary + DB constraint `CHECK (stock >= 0)`. |
| **Duplicate User Email Registration** | Prisma throws code `P2002` (Unique Constraint Violation). Auth controller catches and returns `409 Conflict`. | Existing implementation handles correctly via catch block. In production, ensure user enumeration protection if privacy policies dictate. |
| **Invalid or Tampered JWT** | `jwt.verify()` throws `JsonWebTokenError` or `TokenExpiredError`. Auth middleware returns `401 Unauthorized`. | Standard behavior correctly implemented. Log security warning for tampered signatures to detect suspicious API scanning. |
| **Frontend Refresh After Login** | Component state resets. Auth slice initializes user state from `sessionStorage`. If valid token exists, user stays logged in. | Store refresh token in secure HTTP-Only cookie and call `/api/auth/me` on app initialization to restore state safely. |

---

## 7. SECURITY DEEP DIVE

### 1. Password Storage & Hashing
- **Method:** `bcryptjs` with salt rounds set to `10`.
- **Mechanism:** Blowfish-based adaptive key derivation function. Salt is generated per password to prevent rainbow table attacks. High work factor defends against brute-force offline GPU cracking.

### 2. SQL Injection Prevention
- **Mechanism:** Prisma ORM utilizes parameterized queries (prepared statements) for all database interactions. Input values are never concatenated into raw SQL strings.

### 3. XSS (Cross-Site Scripting)
- **Front-end Escaping:** React automatically escapes values rendered in JSX (`{vehicle.make}`), preventing inline script execution.
- **Production Improvement:** Set Content Security Policy (CSP) headers using `helmet` middleware on Express to restrict executable script sources.

### 4. CSRF (Cross-Site Request Forgery)
- **Current State:** Using JWT in custom HTTP headers (`Authorization: Bearer <token>`). Browsers do not automatically attach custom authorization headers to cross-origin requests, rendering standard CSRF attacks ineffective.
- **Production Cookie Transition:** If moving to HTTP-Only cookies, enforce `SameSite=Strict` and issue anti-CSRF double-submit tokens.

### 5. Input Sanitization & Validation
- **Backend:** Zod middleware validates request headers, params, and body before execution reaches controllers.
- **Frontend:** React Hook Form coupled with Zod resolvers validates inputs prior to network transmission.

---

## 8. TESTING STRATEGY (TDD)

```
        ▲
       / \         E2E Tests (Cypress / Playwright) - User Flows
      /   \
     /-----\       Integration Tests (Jest + Supertest) - API + DB Layer
    /       \      Component Tests (React Testing Library) - UI + Redux
   /---------\
  /           \    Unit Tests (Jest) - Validators, Services, Reducers
 /─────────────\
```

### Why Test-Driven Development (TDD)?
1. **Clear Interface Boundaries:** Writing tests first forces you to design clean function signatures, REST routes, and state shapes before getting bogged down in implementation details.
2. **Regression-Free Refactoring:** Allowed refactoring monolithic auth and vehicle routes into Controller-Service-Repository layers with 100% confidence because existing Supertest suites immediately flagged breaking changes.
3. **Red-Green-Refactor Cycle:**
   - 🔴 **Red:** Write a failing test asserting desired endpoint behavior or UI state.
   - 🟢 **Green:** Implement the minimal code needed to make the test pass.
   - 🔵 **Refactor:** Clean up duplicated logic, extract services, enforce type safety while keeping tests green.

### What is Mocked vs. Not Mocked?

#### Backend Integration Tests (`backend/tests/*.test.ts`):
- **NOT Mocked:** Express routes, controllers, services, repositories, and Prisma ORM hitting a real Neon PostgreSQL database instance.
- **Why?** Real database queries verify SQL constraints, unique keys, and Prisma transaction behaviors accurately.

#### Frontend Component & Slice Tests (`frontend/src/**/*.test.tsx`):
- **Mocked:** Axios HTTP calls (mocked via Jest functions/MSW) and browser `sessionStorage`.
- **NOT Mocked:** Redux store reducers, React component rendering, DOM user event simulations (`@testing-library/user-event`).

---

## 9. AI USAGE & METHODOLOGY

### AI Tools Utilized:
1. **Antigravity (IDE Agent):** Scaffolding directory structures, boilerplate generation, refactoring monolithic files into layered architecture.
2. **Claude:** Architectural reasoning, database concurrency design, debugging complex TypeScript/Prisma type mismatches.
3. **ChatGPT:** Drafting initial failing test cases for Supertest, explaining configuration errors (Jest/Vite).

### Strategic AI Prompting Workflow:
- **Incremental Prompting:** Prompts were executed strictly one requirement at a time (e.g., Prompt 4: Registration Tests → Prompt 5: Registration Code → Prompt 6: Layer Refactor).
- **Verification Protocol:** Every snippet generated by AI was audited, run against local TypeScript compiler (`tsc`), and verified via `npm test`. No AI code was committed without passing automated test suites.

### Common AI Errors & How They Were Resolved:
- **Prisma Transaction Scope Error:** AI omitted the `tx` parameter in transaction callbacks. Detected by manual code review and fixed by explicitly passing `tx` down to repository methods.
- **Jest Async Hanging Handles:** AI generated Supertest execution blocks without closing database connections. Resolved by adding `afterAll(async () => { await prisma.$disconnect(); })` hooks across test suites.

---

## 10. SENIOR MOCK INTERVIEW QUESTION BANK

Below is the exact question bank senior interviewers will pull from:

1. **Architecture:** "Walk me through how a request flows from the React UI to PostgreSQL in your app."
2. **Concurrency:** "What happens if two buyers click 'Purchase' on a vehicle with stock = 1 at the exact same millisecond?"
3. **Security:** "Why did you store JWTs in sessionStorage instead of HTTP-Only cookies? What are the risks?"
4. **Database Design:** "I noticed you don't have an Order table. Why? How would you add one without breaking existing APIs?"
5. **Testing:** "Explain how TDD helped you refactor your backend from monolithic routes to Controller-Service-Repository."
6. **Scalability:** "If this system receives 50,000 requests/minute, what component fails first and how do you fix it?"
7. **Performance:** "How would you optimize `GET /api/vehicles?make=Toyota&maxPrice=30000` when the table has 2,000,000 rows?"
8. **AI Usage:** "Did AI write code that contained bugs or performance flaws? Give me a concrete example of how you fixed it."

---

## 11. CONFIDENCE SCORE & REVISION CHECKLIST

### Current Preparedness Score: **92 / 100**

### Key Topics to Revise Before Interview:
- [ ] Practice explaining PostgreSQL Transaction Isolation Levels (`ReadCommitted` vs `Serializable`).
- [ ] Practice articulating the difference between Optimistic vs. Pessimistic Locking.
- [ ] Review HTTP-Only `SameSite` cookie security mechanics vs JWT `Authorization` header.
- [ ] Be ready to demonstrate full ownership of AI prompts documented in `Prompts.md`.
