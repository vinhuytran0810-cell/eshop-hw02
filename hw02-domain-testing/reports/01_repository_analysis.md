# Task 01 - Repository Analysis

| Item | Value |
| --- | --- |
| Assignment | HW02 - Domain Testing on EShop |
| Assigned features | FR-04 Personal Profile Management (Web); FR-10 Order State Machine (Web); FR-15 Product CRUD (Web Admin); FR-12 Access Control (Mobile testing scope) |
| Repository root | `D:/KiemThu/eshop-sut` |
| Inspected Git HEAD | `85af3ba875c88283615e22cb108f13e2fccaf0e9` |
| Analysis date | 2026-10-13 (Asia/Saigon) |
| Authoritative requirements | Root `README.md`, SRS version 2.0, dated 2026-05-14 (line 288) |
| Analysis method | Read-only source and documentation inspection; no application execution or API requests |
| Evidence status | Static observations and potential discrepancies only; no confirmed bugs and no pass/fail results |

The root README is the only authority for expected business behavior in this report. `api_specification.md` and `setup_guide.md` are secondary documentation, not replacement requirements. Source code identifies the System Under Test (SUT), its actual request wiring, and risks to investigate. Comments that describe intentional defects are not execution evidence. All source paths below are relative to the repository root unless explicitly absolute; line references identify the inspected revision.

## 1. Repository Architecture

### 1.1 Components and entry points

| Component | Verified files / entry points | Structure and responsibility | Address / configuration observation |
| --- | --- | --- | --- |
| Backend API | `backend/package.json`; `backend/server.js:1-12,570-572` | Node.js, Express, `body-parser`, CORS, `jsonwebtoken`, SQLite. All routes and their anonymous controller callbacks are in `server.js`; there are no separate route/controller directories in the inspected tree. | `PORT = 3000`; frontend requests use `/api`. No server was started. |
| Persistence | `backend/database.js:1-13,50-81,117-119`; `backend/database.sqlite` | Direct `sqlite3` queries, not ORM model classes. `initDatabase()` defines tables and seeds data; the exported `db` is imported by the server. | Database path is resolved relative to `backend/database.js`, not the shell working directory. Existing database contents were not queried. |
| Customer Web | `frontend-web/package.json`; `frontend-web/src/main.jsx`; `frontend-web/src/App.jsx`, functions `App()` and `Header()` | React, Vite, Tailwind; React Router pages and shared `AuthProvider`/`CartProvider`. | SRS default is `http://localhost:5173`; `frontend-web/vite.config.js` does not pin a port. API URLs are hardcoded to `http://localhost:3000`. |
| Web Admin | `frontend-admin/package.json`; `frontend-admin/src/main.jsx`; `frontend-admin/src/App.jsx`, `App()` | React, Vite, Tailwind; login, tabs, CRUD handlers, and order controls are in one component. Tab selection uses `activeTab`, not separate page routes. | `frontend-admin/vite.config.js` pins port 5174 with `strictPort: true`; API base at `App.jsx:4` is `http://localhost:3000/api`. |
| Mobile | `frontend-mobile/package.json`; `frontend-mobile/index.js`, `registerRootComponent(App)`; `frontend-mobile/App.js`, `App()`; `frontend-mobile/app.json` | React Native + Expo. In-memory state and `view` conditionals select screens; requests use native `fetch`. No separate screen modules or navigation library are present in this app. | `frontend-mobile/App.js:16` hardcodes `http://192.168.10.13:3000/api`. Its reachability from a device/emulator is unverified. |
| Specifications / setup | Root `README.md`; `api_specification.md`; `setup_guide.md`; `run_servers.sh` | SRS, secondary API reference, setup instructions, and a shell startup helper. | `run_servers.sh` contains machine-specific macOS paths and `killall node`; it was not run. |
| Existing helper | `backend/test_profile.js` | A script signs a token with a hardcoded secret and sends a profile GET request. It contains no assertion framework. | Not executed; its filename/output text does not establish a passed test. |

### 1.2 Verified data structures and shared middleware

| Structure / function | Source | Relevant fields or behavior | Testing implication |
| --- | --- | --- | --- |
| `users` table | `backend/database.js:50-61`, inside `initDatabase()` | `id`, `name`, `email`, `password`, `role` (default `user`), `login_attempts`, `locked_until`, `reset_token`, `shipping_address`, `phone` | Stores profile, actor identity, and role. There are no name/phone length or role-enumeration constraints in the table definition. |
| `products` table | `backend/database.js:64-71` | `id`, `name`, `price INTEGER`, `description`, `imageUrl`, `category_id` | No `NOT NULL`, name-length `CHECK`, positive-price `CHECK`, or category foreign-key declaration is present. `INTEGER` is an implementation observation, not an SRS requirement that prices must be integers. |
| `categories` table | `backend/database.js:23-26` | `id`, `name` | Supplies existing category choices for the Admin product form. |
| `orders` table | `backend/database.js:74-81` | `id`, `user_id`, `total_amount`, `status` (default `pending`), `shipping_address`, `created_at` | No database state-enumeration or transition constraint is present; behavior depends on handlers. No order seed statements are present in `initDatabase()`. |
| `coupons`, `coupon_usage` tables | `backend/database.js:29-46` | Coupon definitions and usage references | Relevant to FR-12 endpoint coverage; coupon business validation itself is outside this assignment scope. |
| `authenticateToken(req,res,next)` | `backend/server.js:100-110` | Extracts the second space-separated Authorization header element; verifies JWT; places decoded claims in `req.user`. Missing token returns 401; verification error returns 403. | Observed HTTP codes, not SRS-prescribed codes. This middleware does not check `role`. |
| Anonymous login callback | `backend/server.js:32-66` | Looks up user and signs JWT claims `{ id: user.id, role: user.role }` at line 51, without an explicit expiry option. | Mobile and Web share token issuance. Expiry duration and immediate role-change revocation are not defined by the SRS. |
| `userCarts` | `backend/server.js:14,284-309` | Server in-memory per-user cart structure; checkout writes an order with `pending`. | Checkout is an order-fixture dependency; cart correctness is not being analyzed as an assigned feature. |

Importing `backend/database.js` calls `initDatabase()` unconditionally. It drops and recreates all six tables (`database.js:15-20`) and reseeds users/categories/products/coupons. Starting `server.js` imports this module at line 4. Consequently, restarting the backend can erase test fixtures and evidence. This was identified by inspection; no initialization was executed.

## 2. Feature-to-Source Mapping

### 2.1 Frontend components and Mobile screens

| Feature | Verified UI surface | Relevant functions / code | Request and persistence path |
| --- | --- | --- | --- |
| FR-04 Web profile | Web route `/profile` in `frontend-web/src/App.jsx:55`; `frontend-web/src/pages/Profile.jsx:5`, `Profile()` | `handleUpdate()` at line 40; profile form at lines 111-165; disabled email at lines 119-123; unauthenticated message at line 108 | `PUT /api/users/me` -> inline callback at `backend/server.js:118` -> `users`, filtered by token user ID. |
| FR-04 identity loading | `frontend-web/src/context/AuthContext.jsx`, `AuthProvider()` at line 6 | Token effect at lines 10-24 reads `/users/me`; `login()` line 26; `logout()` line 37; `useAuth()` line 50 | `POST /api/login`, `GET /api/users/me`; token in Web `localStorage` key `token`. |
| FR-10 Web user cancellation | Order history within `Profile()`, not a separate Orders page | `fetchOrders()` line 14; `cancelOrder()` line 66; `statusStyle()` line 82; `statusLabel()` line 97; button condition at line 200 | `GET /api/orders/my-orders`, `PUT /api/orders/:id/cancel` -> `orders`. |
| FR-10 Admin actor / fixture progression | `frontend-admin/src/App.jsx`, `App()`, `activeTab === 'orders'` at line 777 | `fetchData()` line 41; `updateOrderStatus()` line 85; state-specific controls at lines 814-869 | `GET /api/admin/orders`, `PUT /api/admin/orders/:id/status` -> `orders`. Admin surface is needed to exercise the full Web state machine. |
| FR-10 order creation dependency | Web `/checkout` in `frontend-web/src/App.jsx:58`; `frontend-web/src/pages/Checkout.jsx`, `Checkout()` / `handleCheckout()` | Submits checkout and shows completion UI | `POST /api/checkout` -> callback at `backend/server.js:297`; inserts `pending` and returns `orderId`. |
| FR-15 Admin product CRUD | `frontend-admin/src/App.jsx`, `activeTab === 'products'` at line 337 | `fetchData()` line 41; `productForm` lines 95-102; `handleProductSubmit()` line 105; `deleteProduct()` line 133; form lines 483-566; table/edit/delete controls lines 567-608 | `GET/POST /api/products`; `PUT/DELETE /api/products/:id`; `GET /api/categories`. Tables: `products`, `categories`. |
| FR-12 shared enforcement | Every applicable backend route, independent of client platform | `backend/server.js`, `authenticateToken()`; route inventory in Section 4 | Token validity and token role must be enforced at the backend; hiding a client control is insufficient. |
| FR-12 existing Admin UI comparison | `frontend-admin/src/App.jsx` | `handleLogin()` lines 61-74 checks returned `user.role`; stored `adminToken` lines 7,69-70; render gate checks token presence at line 188; token effect invokes `fetchData()` | The UI role check is in Web Admin only. No dedicated Admin authorization middleware was found. |
| FR-12 Mobile session entry | `frontend-mobile/App.js`, `App()` | `renderLogin()` line 759; `handleLogin()` lines 186-207; token/user state lines 34-35; `logout()` lines 161-169; `renderHeader()` lines 422-440 | Login obtains the same backend JWT. The app holds it in React state; no persistent token-storage call is present. |
| FR-12 Mobile session consumers / context | `frontend-mobile/App.js` | `fetchOrders()` line 171; `handleUpdateProfile()` line 286; `cancelOrder()` line 312; `renderProfile()` line 893 | Sends Bearer token to customer endpoints. These calls verify session wiring but do not themselves exercise the FR-12 Admin endpoint restriction. |

### 2.2 Verified Mobile surface and FR-12 reachability

The Mobile view switch at `frontend-mobile/App.js:978-985` contains `cart`, `checkout`, `login`, `register`, `forgotPassword`, `profile`, `productDetail`, and the default Home screen. Their render functions are `renderCart`, `renderCheckout`, `renderLogin`, `renderRegister`, `renderForgotPassword`, `renderProfile`, `renderProductDetail`, and `renderHome`. `renderScreen()` provides the common wrapper at line 442.

Inspection of the complete file and its request calls found no Admin screen, no Admin navigation destination, no role-based routing, no `/api/admin/*` call, and no product/category/coupon CRUD mutation call. An Admin account can reach the shared login handler because it has no client role restriction, but the resulting native UI remains the customer app in the inspected source. Native runtime behavior is still unverified.

FR-12 therefore cannot be fully exercised through existing Mobile UI controls alone. A proposed Mobile-origin session/API strategy is documented in Section 7; it is not presented as an existing application feature.

## 3. SRS Business Rules

The identifiers below are report traceability identifiers, not additional SRS requirements. Rules are translated from the Vietnamese root README. Mapping to source does not imply compliance.

### 3.1 FR-04 - Personal Profile Management

| Rule ID | Authoritative expected behavior | SRS reference | Relevant implementation mapping |
| --- | --- | --- | --- |
| P04-01 | A logged-in user can update full name, phone, and default shipping address. | `README.md:64` | `frontend-web/src/pages/Profile.jsx`, `Profile()` / `handleUpdate()` (5,40-64,126-154); `AuthProvider()` in `frontend-web/src/context/AuthContext.jsx:6-24`; `backend/server.js:112-135`, GET/PUT profile callbacks; `backend/database.js:50-61`, `users`. |
| P04-02 | A valid phone begins with `0` and contains 10-11 digits. | `README.md:65` | `frontend-web/src/pages/Profile.jsx:43-46`, `handleUpdate()` validation; `backend/server.js:118-134`, PUT callback; `users.phone` at `backend/database.js:60`. |
| P04-03 | Email cannot be changed through the UI. | `README.md:66` | Disabled email field in `frontend-web/src/pages/Profile.jsx:119-123`; update payload at lines 49-62; PUT callback at `backend/server.js:119-129`. This rule does not explicitly define a separate email-changing API. |
| P04-04 | Users can update only their own profile. | `README.md:67` | `authenticateToken()` at `backend/server.js:100-110`; PUT callback uses `WHERE id = ?` with `req.user.id` at lines 128-129; GET uses the same identity at lines 112-115. |
| P04-05 | Users cannot change their own `role`. | `README.md:67`; corroborated by reference security rule `SEC-06` at line 283 | PUT callback destructures `role` and conditionally adds it to SQL at `backend/server.js:119-127`; `users.role` at `backend/database.js:55`; login token claims at `backend/server.js:51`. The ordinary Web form has no role control. |

FR-04 does not state maximum name/address lengths, whitespace normalization, or whether phone/address may be omitted or cleared. These are open questions, not permission to adopt the UI's `required` attribute or regex as a requirement.

### 3.2 FR-10 - Order State Machine

| Rule ID | Authoritative expected behavior | SRS reference | Relevant implementation mapping |
| --- | --- | --- | --- |
| O10-01 | The five defined states are `pending`, `confirmed`, `shipping`, `delivered`, `canceled`. Transitions must follow the diagram. | `README.md:143-156` | `orders.status`, `backend/database.js:78`; checkout callback `backend/server.js:297-308`; cancel callback lines 321-342; Admin status callback lines 525-568; Web `statusLabel()` and Admin `statusLabel()`. |
| O10-02 | `delivered` and `canceled` are final: no transition to another state is allowed. | `README.md:160` | Cancel guard `backend/server.js:329-330`; Admin transition predicates lines 534-556; Admin final-state controls `frontend-admin/src/App.jsx:854-869`; Web cancellation control `frontend-web/src/pages/Profile.jsx:200`. |
| O10-03 | A user cannot self-cancel a `shipping` order; only Admin may act at that stage. | `README.md:161` | Web `cancelOrder()` and visibility condition (`Profile.jsx:66,200`); cancel callback `backend/server.js:321-341`; Admin status callback lines 525-568. The exact interpretation of Admin cancellation during shipping is ambiguous; see Section 6. |
| O10-04 | Every invalid transition must return an error with an appropriate message. | `README.md:162` | Cancel callback `backend/server.js:326,329-330`; Admin callback lines 532,553-556; Web `cancelOrder()` error display (`Profile.jsx:77-78`); Admin `updateOrderStatus()` error display (`App.jsx:89-91`). Exact HTTP code/text is not prescribed. |

| Diagram transition | Authorized actor in the SRS diagram | Implementation location |
| --- | --- | --- |
| `pending -> confirmed` | Admin | `backend/server.js:537-541`, Admin PUT status callback; `frontend-admin/src/App.jsx:814-823`, `updateOrderStatus()`. |
| `confirmed -> shipping` | Admin | `backend/server.js:542-546`; `frontend-admin/src/App.jsx:834-843`. |
| `shipping -> delivered` | Admin | `backend/server.js:547-548`; `frontend-admin/src/App.jsx:854-860`. |
| `pending -> canceled` | User or Admin | User cancel callback `backend/server.js:321-341`; Admin status callback lines 537-541; Web `Profile.jsx:66-80,200-207`; Admin `App.jsx:824-831`. |
| `confirmed -> canceled` | User or Admin | Same user callback; Admin status callback `backend/server.js:542-546`; Web same control; Admin `App.jsx:844-851`. |

Related SRS dependencies: FR-11 (`README.md:166-168`) limits order viewing to the owner and specifies displayed order fields, Vietnamese status labels, and distinct colors. FR-18 (`README.md:220-222`) provides Admin-wide order access and explicitly reuses FR-10. FR-20 (`README.md:235-236`) provides Mobile order history and restricts Mobile cancellation to `pending`/`confirmed`. These dependencies support observability and setup; they do not make every aspect of order history/Admin/Mobile an assigned feature.

### 3.3 FR-15 - Product Management CRUD

| Rule ID | Authoritative expected behavior | SRS reference | Relevant implementation mapping |
| --- | --- | --- | --- |
| P15-01 | Admin can create, view, edit, and delete products. | `README.md:193` | `frontend-admin/src/App.jsx`, `fetchData()` (41), `handleProductSubmit()` (105), `deleteProduct()` (133), product tab (337), edit/delete controls (592-603); backend GET list/detail and POST/PUT/DELETE callbacks (`server.js:141,159,167,179,191`). |
| P15-02 | Product name is required and at most 255 characters. | `README.md:195` | Name input `frontend-admin/src/App.jsx:491-499`; submit callback lines 105-131; backend create/update callbacks lines 167-189; `products.name`, `backend/database.js:66`. |
| P15-03 | Price is required and must be a positive number (`> 0`). | `README.md:196` | Price input `frontend-admin/src/App.jsx:500-508`; create/update callbacks `backend/server.js:167-189`; `products.price`, `backend/database.js:67`. |
| P15-04 | Category is required and must be chosen from the existing list. | `README.md:197` | `fetchData()` category load at `frontend-admin/src/App.jsx:49-50`; category select at lines 528-543; default category state at lines 101,126; `GET /api/categories` callback `backend/server.js:243`; product writes lines 167-189; `products.category_id` and `categories` schema. |
| P15-05 | Editing one product changes only that product; all others remain unchanged. | `README.md:198` | `handleProductSubmit()` at `frontend-admin/src/App.jsx:108-115`; backend update callback at `backend/server.js:179-189` filters by path ID. UI state and persisted data must be observed separately. |

FR-15 does not specify maximum price, smallest positive price/precision, description/image URL validation, product-name uniqueness, delete confirmation, or behavior for a nonexistent product ID. None is added as a business rule here.

### 3.4 FR-12 - Access Control, with Mobile testing context

| Rule ID | Authoritative expected behavior | SRS reference | Relevant implementation mapping |
| --- | --- | --- | --- |
| A12-01 | The Admin subsystem is restricted to accounts with `role = 'admin'`. | `README.md:176` | Web Admin `handleLogin()` at `frontend-admin/src/App.jsx:61-74` and render gate at line 188; backend login/token issuance at `server.js:32-66`; `users.role` schema. No native Admin surface is implemented. |
| A12-02 | All `/api/admin/*` APIs require a valid JWT and token `role = 'admin'`. | `README.md:177-179` | `authenticateToken()` at `backend/server.js:100-110`; all seven implemented Admin routes listed in Section 4.3. No role predicate is present in these handlers. |
| A12-03 | Data-mutating product/category/coupon APIs require a valid JWT and token `role = 'admin'`. | `README.md:177-179` | Product POST/PUT/DELETE at `backend/server.js:167,179,191`; category POST/PUT/DELETE at lines 249,257,269; implemented coupon writes under `/api/admin/coupons` at lines 457,483. |

Reference security rule SEC-03 (`README.md:280`) corroborates that token existence alone is insufficient. FR-02 (`README.md`, token paragraph) specifies Bearer-token client storage/transmission. FR-20 lists customer Mobile capabilities but does not define an Admin Mobile module, role-specific Mobile navigation, or an API-testing screen. The Mobile assignment does not change FR-12's server-side authorization rule.

### 3.5 Cross-cutting UI requirements relevant to later review

| SRS section | Applicable requirement / scope note | Relevant source |
| --- | --- | --- |
| FR-21, `README.md:244-248` | Vietnamese text, action/danger colors, currency formatting, one HTML `h1` per Web page, ordered tab focus. HTML rules cannot be automatically translated into native widgets. | Web `Profile()`; Admin product/order sections; Mobile `renderLogin()` / `renderProfile()`. |
| FR-22, `README.md:252-255` | Required fields marked `*`; HTML email/password types; errors above submit. Native control equivalents require interpretation for Mobile. | Web profile form; Admin product form and login; Mobile login/profile renderers. |
| FR-23 / FR-24, `README.md:260-270` | Selected navigation highlighted; logout label; appropriate empty state; descriptive product image alt text on Web. | Web `Header()` / `Profile()`; Admin product table; Mobile `renderHeader()` / `renderProfile()`. |
| SEC-04 (reference), `README.md:281` | Escape displayed user input; do not directly render it as HTML. FR-18 explicitly requires safe displayed shipping addresses. | `frontend-web/src/App.jsx:27`, `Header()`; `frontend-admin/src/App.jsx:799-804`, order rendering. |

These requirements are secondary review dimensions; they do not supply missing numeric limits for the assigned features.

## 4. Related API Endpoints

All routes below were verified in `backend/server.js`. The controller for each is the inline anonymous `(req,res)` callback at the stated registration line. No separate named controller file exists. `JWT only` means `authenticateToken` is attached but no Admin-role check is present. Success/error response descriptions below are source observations, not executed results or authoritative response contracts.

### 4.1 Profile, identity, and orders

| Method / endpoint | Handler line | Observed protection / inputs | Observed output and purpose |
| --- | --- | --- | --- |
| `POST /api/login` | 32 | No JWT; JSON `email`, `password` | JWT with `id`/`role`, `user`, message; supplies all three client sessions. |
| `GET /api/users/me` | 112 | JWT; target is `req.user.id` | `SELECT *` user object; Web profile loading. |
| `PUT /api/users/me` | 118 | JWT; JSON `name`, `shipping_address`, `phone`, optional truthy `role` read by implementation | Writes token user's row; message or database error. Web sends snake_case address; Mobile currently sends camelCase `shippingAddress`. |
| `POST /api/checkout` | 297 | JWT; handler reads `total_amount`, `shipping_address` | Inserts own `pending` order; returns `orderId` and message. Fixture dependency only. |
| `GET /api/orders/my-orders` | 311 | JWT; filters `user_id = req.user.id` | Own orders array ordered by descending ID; Web/Mobile history. |
| `PUT /api/orders/:id/cancel` | 321 | JWT; path ID and token user ID identify owned order; no required body fields | Rejects missing order / `delivered` / `canceled`; otherwise writes `canceled`. Web/Mobile cancellation. |
| `GET /api/orders/:id` | 344 | No JWT or owner check in handler; path ID | Returns order or 404 error; related observation endpoint. No Web/Mobile caller was found in the inspected assigned-feature source. |
| `GET /api/admin/orders` | 510 | JWT only | All orders joined with `user_name`; Admin progression/observation. |
| `PUT /api/admin/orders/:id/status` | 525 | JWT only; path ID, JSON `status` | Reads current state; transition predicate; writes new state or returns error. |

### 4.2 Product CRUD and category selection

| Method / endpoint | Handler line | Observed protection / inputs | Observed output and caller |
| --- | --- | --- | --- |
| `GET /api/products` | 141 | No JWT; optional `search` query | Product array; Admin `fetchData()`, Web/Mobile product browsing. |
| `GET /api/products/:id` | 159 | No JWT; path ID | Product object; absent record returns `{}` with 200 in source; even-ID product price converted to string. Used by customer product-detail screens, not Admin form population. |
| `POST /api/products` | 167 | No JWT or role guard; `name`, `price`, `description`, `imageUrl`, `category_id` | Insert with message and new ID; Admin `handleProductSubmit()`. |
| `PUT /api/products/:id` | 179 | No JWT or role guard; same body fields and path ID | Update filtered by ID; message; Admin `handleProductSubmit()`. Does not inspect affected-row count. |
| `DELETE /api/products/:id` | 191 | No JWT or role guard; path ID | Delete filtered by ID; message; Admin `deleteProduct()`. Does not inspect affected-row count. |
| `GET /api/categories` | 243 | No JWT | Category array; Admin `fetchData()` populates product selector. |

### 4.3 Complete implemented FR-12 protected-route inventory

SRS expected protection for every row below is **valid JWT AND token role `admin`**, regardless of caller platform. Public product/category reads are not made Admin-only by FR-12.

| Method / endpoint | Handler line | Observed protection | FR-12 relevance |
| --- | --- | --- | --- |
| `POST /api/products` | 167 | None | Product mutation. |
| `PUT /api/products/:id` | 179 | None | Product mutation. |
| `DELETE /api/products/:id` | 191 | None | Product mutation. |
| `POST /api/categories` | 249 | JWT only | Category mutation. |
| `PUT /api/categories/:id` | 257 | JWT only | Category mutation. |
| `DELETE /api/categories/:id` | 269 | JWT only | Category mutation. |
| `POST /api/admin/import-products` | 199 | JWT only | Admin-prefix API; import correctness remains outside FR-15 CRUD scope. |
| `POST /api/admin/coupons` | 457 | JWT only | Admin-prefix API and coupon mutation. |
| `DELETE /api/admin/coupons/:id` | 483 | JWT only | Admin-prefix API and coupon mutation. |
| `GET /api/admin/users` | 494 | JWT only | Admin-prefix read API. |
| `DELETE /api/admin/users/:id` | 504 | JWT only | Admin-prefix mutation API. |
| `GET /api/admin/orders` | 510 | JWT only | Admin-prefix read API. |
| `PUT /api/admin/orders/:id/status` | 525 | JWT only | Admin-prefix mutation API. |

The source implements `GET /api/coupons` at line 356 with JWT only; it is not a mutation and is not an `/api/admin/*` route, so FR-12 alone does not establish that this read must be Admin-only. No `POST/PUT/DELETE /api/coupons` or `PUT /api/admin/coupons/:id` registration exists. Do not fabricate these endpoints to satisfy the SRS's route-family notation. `POST /api/apply-coupon` and `POST /api/coupon-usage` are separate customer workflows; do not equate them with Admin coupon-definition CRUD without clarified scope.

## 5. Testing Scope

### 5.1 Scope by feature

| Feature | Primary scope | Supporting surface | Explicit exclusions / limits |
| --- | --- | --- | --- |
| FR-04 Web | Profile editing, specified phone rule, read-only email UI, own-user targeting, role immutability; compare UI and persisted/read-back values. | Web login/AuthContext; backend profile routes; related header rendering. | Registration/reset-password algorithms; invented name/address limits. Mobile profile findings are contextual, not substituted for assigned Web coverage. |
| FR-10 Web | User cancellation, actor-sensitive transition enforcement, final-state preservation, invalid-transition feedback. | Web profile history; Admin order tab/API for authorized transitions and state setup; checkout for order creation. | Financial correctness, coupon rules, dashboard totals; complete FR-18 coverage. Ownership/history is a related dependency under FR-11. |
| FR-15 Web Admin | Create/read/update/delete; name, price, category constraints; edit isolation in UI and persistence. | Admin login; category reads; public product reads for independent verification. | CSV import business rules (FR-16); category/coupon CRUD business validation; invented image/description restrictions. |
| FR-12 Mobile context | Shared authorization requirements, using identities established through native Mobile login where observable; denial must prevent data access/mutation. | Mobile session lifecycle; documented capture of Mobile-issued session evidence; external API tooling for existing protected endpoints, subject to human scope review. | No native Admin screen exists to drive. Web Admin UI tests or generic API tests cannot be described as native Mobile UI execution. |

No server, Web UI, Admin UI, or Mobile app was launched for Task 01. No endpoint request or test helper was executed. No test cases, EC representatives, boundary datasets, automated scripts, bug Issues, or test results were generated.

### 5.2 Variables and observations to analyze in the next task

This table identifies analysis dimensions only. It does not enumerate equivalence classes or choose test data.

| Feature | Candidate inputs / context | Outputs to observe later | Ordered dimensions grounded in the SRS |
| --- | --- | --- | --- |
| FR-04 | Token/session identity; full name; phone string; default address; submitted extra fields such as `role`; UI email editability | Update acceptance/rejection; displayed/read-back profile; target identity; unchanged role and UI email | Phone length: inclusive 10-11 digits; leading zero/digit composition is categorical. No specified name/address numeric limits. |
| FR-10 | Current state; requested action/target state; actor role; token; order ID/ownership as context | Accepted/rejected transition; persisted next state; unchanged state on rejection; error feedback | States/roles/actions are categorical. Diagram sequence is not a numeric scale. No timing/count bounds specified by FR-10. |
| FR-15 | Operation; target ID; name; price; selected category; description/image as unconstrained ancillary inputs; token role | CRUD result; persisted target; unchanged unrelated records; displayed list/form; rejection feedback | Name maximum 255 characters; required nonempty name; price strictly above zero. No specified price maximum or numeric precision. |
| FR-12 | Endpoint/method; token presence/validity; role claim; Mobile session context | Allowed/denied operation; response category; absence of unauthorized data/mutation | Token validity and role are categorical. No SRS minimum/maximum token length or expiry duration. |

Outputs must also be considered during later EC analysis, but the SRS does not define an exhaustive JSON schema or prescribe exact HTTP status codes for these features. Source response shapes can guide observation without becoming the oracle.

## 6. Observed Risks and Ambiguities

### 6.1 Potential discrepancies from static inspection

Every item below is a hypothesis for later execution, not a confirmed bug. Source statements indicate what the implementation appears to do, not a recorded runtime outcome.

| Risk ID | SRS expectation | Implementation observation with exact source | Validation needed later |
| --- | --- | --- | --- |
| R-01 | P04-02: phone starts with zero, length 10-11 digits. | Web `handleUpdate()` uses `/^[1-9][0-9]{8,9}$/` and a 9-10 digit message (`frontend-web/src/pages/Profile.jsx:43-46`). Profile PUT callback has no phone validation (`backend/server.js:118-134`). | Separate UI rejection/acceptance from direct API validation and persistence. Do not use the source regex as expected behavior. |
| R-02 | P04-05: users cannot change their role. | Profile PUT accepts truthy `role` and writes it (`backend/server.js:119-127`). Login signs the database role (`server.js:51`). | Observe persisted role and subsequent token issuance separately from the role in an already-issued token. No escalation was attempted. |
| R-03 | O10-03: user cannot cancel shipping. | Web cancellation button excludes only delivered/canceled (`Profile.jsx:200`); backend cancel guard excludes the same two states (`server.js:329-330`). | Establish actual user role, current persisted state, UI action/request, response and final state. |
| R-04 | O10-02: canceled is terminal. | Admin callback explicitly allows canceled to delivered (`backend/server.js:550-551`); Admin UI exposes that action (`frontend-admin/src/App.jsx:862-868`). | Exercise through Admin surface and independently observe persistence/error behavior. |
| R-05 | A12-02: all Admin APIs require Admin role in JWT. | `authenticateToken()` verifies signature but not role (`server.js:100-110`); all seven Admin-prefix handlers attach this middleware without a role predicate. | Observe read denial and mutation prevention for non-Admin identities across each existing protected endpoint. |
| R-06 | A12-03: product writes require valid Admin JWT. | Product POST/PUT/DELETE attach no auth middleware (`backend/server.js:167,179,191`); global middleware is only CORS/JSON parsing (`server.js:11-12`). | Evaluate backend authorization independently of the Web Admin login gate. |
| R-07 | A12-03: category writes require Admin role. | Category POST/PUT/DELETE use JWT verification only (`backend/server.js:249,257,269`). | Verify role-sensitive denial and lack of unintended mutations. |
| R-08 | P15-02/P15-03: required bounded name and positive required price. | Admin name field only has `required`, no `maxLength` (`App.jsx:491-499`); price lacks `required` and `min` (`App.jsx:500-508`). Product write callbacks and schema do not enforce these SRS constraints (`server.js:167-189`; `database.js:64-71`). | Observe native browser constraints, request conversion, API handling, and persistence independently. |
| R-09 | P15-04: category must exist and be required. | Selector uses fetched categories but defaults to ID 1 (`frontend-admin/src/App.jsx:101,126,528-543`); backend writes category IDs without existence lookup; schema has no FK (`server.js:167-189`; `database.js:70`). | Validate selection behavior, submitted ID, and membership in the actual existing category list. |
| R-10 | P15-05: other products remain unchanged after editing one. | Admin edit success maps every displayed product to the edited name (`frontend-admin/src/App.jsx:110-114`). SQL update filters one ID (`backend/server.js:182-183`). | Distinguish transient UI list corruption from database mutation; compare immediately after edit and after fresh retrieval/reload. Do not claim a mass database update. |
| R-11 | FR-04 editable profile should be accurately observable after update. | Web `handleUpdate()` alerts success without refreshing `user` or GET profile (`Profile.jsx:49-64`); header uses context name (`frontend-web/src/App.jsx:27`). | Compare current form, header, read-back and reload; determine which displayed values become stale. |
| R-12 | SEC-04 reference requires safe user-input rendering; FR-18 requires safe shipping-address display. | Header injects `user.name` as HTML (`frontend-web/src/App.jsx:27`); Admin order row injects address as HTML (`frontend-admin/src/App.jsx:799-804`). | Treat as related security-review risk; do not claim script execution without controlled rendering evidence. |
| R-13 | FR-11 limits viewing to own orders; SEC-02 references authenticated protected APIs. | `GET /api/orders/:id` has no token/owner guard (`backend/server.js:344-348`), unlike `/my-orders`. | Related ownership risk; no caller in assigned UI was found and no unauthorized read was executed. |
| R-14 | FR-20 Mobile includes profile capability (supporting context). | Mobile update sends `shippingAddress`, while backend reads `shipping_address` (`frontend-mobile/App.js:302`; `backend/server.js:119`). Mobile locally updates snake_case state at line 306. | Potential client/server address mismatch; contextual only, outside assigned Web FR-04 unless scope expands. |
| R-15 | FR-21/FR-22 define Web headings/form conventions. | Profile renders `h2` headings and email `type='text'`, required name lacks `*` (`Profile.jsx:113,120,126-134,167`); Admin product form uses placeholders and `h3` (`App.jsx:487-543`). | Inspect rendered page/form against applicable GUI rules later; static findings are not GUI test results. |

### 6.2 Requirements ambiguities and environment risks

| ID | Uncertainty / evidence | Proposed handling for human review |
| --- | --- | --- |
| Q-01 | FR-12 is under Web Admin in `README.md:172-179`; FR-20 has only customer Mobile capabilities. Mobile source has no Admin view or protected Admin request. | Confirm whether Mobile-origin login plus external API authorization checks satisfies Pool D, or whether assessment requires native UI-only execution and an alternative feature assignment. Keep native UI evidence and API replay evidence separate. |
| Q-02 | FR-10 diagram has only `shipping -> delivered`, but prose says only Admin may act when shipping (`README.md:147-161`). It does not explicitly draw an Admin shipping-to-canceled edge. | Use diagram-listed edges as the baseline. Mark Admin cancellation during shipping unresolved, rather than declaring an additional allowed edge. FR-20 explicitly restricts customer Mobile cancellation to pending/confirmed. |
| Q-03 | FR-04 defines a valid phone but does not explicitly state whether phone/address must be present; the UI makes name required. | Clarify empty/omitted-field policy and partial-update semantics. Do not derive requiredness from client HTML or backend assignment behavior. |
| Q-04 | FR-15 price must be positive; SRS states neither integer-only currency nor precision nor upper bound. Database declares `INTEGER`, and HTML number inputs may apply browser-specific constraints. | Do not invent minimum 1 or a maximum price. Clarify permitted precision/step before selecting adjacent BVA values. Zero is the exclusive lower cutoff, not a valid minimum in an unrestricted positive real domain. |
| Q-05 | SRS says name maximum 255 characters but does not define whitespace trimming, Unicode length units, or whitespace-only semantics. | Record the chosen counting convention for review; do not silently infer it from JavaScript length or SQLite storage. |
| Q-06 | Nonexistent IDs, same-state submissions, exact status codes, exact error wording, and missing-category concurrency behavior are not fully specified. | Separate definite SRS assertions from exploratory robustness observations. No confirmed outcome should depend on invented HTTP/message contracts. Same-state semantics merit explicit clarification because the final-state prose says no *other* state. |
| Q-07 | Mobile API base is hardcoded to `192.168.10.13` (`App.js:16`); no runtime/environment API override is implemented. | Verify whether the backend host can be reached at the configured address before Mobile execution. Keep source unchanged under current constraints; report an environment blocker if this cannot be arranged. Expo Web alone is not native Mobile evidence. |
| Q-08 | Database initialization destroys existing rows on each import (`backend/database.js:13-20,117`); no seeded orders. | Plan controlled disposable runtime data and evidence capture before restarts. Do not import this module merely to inspect the schema. |
| Q-09 | Setup guide says Admin password `admin123`, whereas SRS and seed use `Admin123!` (`README.md` overview; `database.js:92`). | Use SRS credentials as documented expectation; verify actual runtime login later. Secondary setup instructions are not the oracle. |
| Q-10 | Tokens have no explicit expiry option in login; middleware trusts decoded claims and does not reload user role (`server.js:51,100-110`). | Distinguish current database identity from issued token claims. Do not impose token lifetime, revocation, or immediate role-sync rules absent an SRS clarification. |
| Q-11 | Login lockout implementation differs from FR-02 (+2 attempts, 180000 ms at `server.js:54-57`) and may interfere with obtaining test sessions. | Treat as a fixture/setup risk outside primary assigned-feature assertions. Avoid conflating login failures with FR-12 authorization evidence. |
| Q-12 | Some SQL callbacks return success without handling update errors/affected-row count (e.g. `server.js:336-337,562-563`); current database contents and dependencies are unverified. | Future evidence must include read-back, not just a success alert/message. Environment startup and fixture verification remain pending. |

## 7. Proposed Testing Strategy

### 7.1 Course methodology to apply after Task 01

| Stage | Planned activity | Required discipline |
| --- | --- | --- |
| Domain Testing 1 | Identify input and output variables for each feature using Section 5.2 as an initial inventory. | Include actor/session/state context and observable persistence, not just visible fields. |
| Domain Testing 2 | Identify valid and invalid equivalence classes for every input and output against mapped SRS rules. | Record unspecified behavior separately; implementation-only constraints cannot define validity. |
| Domain Testing 3 | Select the best representative for each class. | Document why it represents the class; human-review AI suggestions. Not performed in this task. |
| Domain Testing 4 | Identify boundary values for ordered variables. | Only use justified order/limits; roles and statuses remain categorical. |
| EC design | Cover every identified class; combine valid classes where possible; test invalid classes independently. | Keep all other inputs/context valid while isolating each invalid class. Avoid accidental earlier validation failures. |
| 2-Point BVA | At each meaningful boundary, select the boundary and its closest adjacent value in the neighboring partition, respecting inclusive/exclusive endpoints. | Define permitted precision/adjacency first; hold other inputs at valid nominal values. No values are selected now. |
| 3-Point BVA | Select the boundary and its closest adjacent values on both sides. | Apply separately from 2-Point BVA and trace coverage; keep other inputs nominal. No datasets are generated now. |
| Human review and evidence | Review AI analysis, execute approved designs later, compare SRS expected behavior with actual behavior and persistence. | Log AI prompts/output/corrections. Confirm bugs only with reproducible execution evidence. |

The numerical-boundary portion of BVA is not automatically applicable to every feature. FR-10 and FR-12 currently have no SRS-defined ordered input range; document BVA as not applicable to those categorical dimensions unless a clarified requirement provides one. Do not assign arbitrary integer ranks to states/roles or adopt implementation JWT length as a business boundary.

### 7.2 Feature-specific approach

| Feature | Planned layers / focus | SRS-grounded boundary approach | Evidence and isolation |
| --- | --- | --- | --- |
| FR-04 Web | Use actual `/profile` UI for editing/email behavior; use existing profile API to distinguish frontend validation from backend enforcement. Assess own identity and prohibited role modification independently of ordinary form fields. | Apply 2-Point and 3-Point BVA to phone length bounds, with valid prefix/digit content. Analyze prefix/content categorically. Name/address limits remain unspecified. | Record UI input/feedback, network request and response, token identity, and GET/read-back after reload. Maintain valid nominal name/address/session while isolating phone behavior; use distinct users for identity isolation. |
| FR-10 Web | Use Web Profile cancellation plus Web Admin order controls for full actor-specific state progression. Model current state, action/target, and actor jointly; transition coverage supplements domain/EC coverage rather than replacing it. | No numeric BVA over state names. Preserve final-state and forbidden-transition analysis as categorical/state coverage; do not invent time limits. | Establish starting state through permitted existing workflow; capture before/after order read-back and error feedback. Keep token/ownership/order existence valid when isolating transition behavior. Admin transitions cannot be driven solely by customer Web UI. |
| FR-15 Web Admin | Drive product tab CRUD; inspect request data and independently retrieve products after changes. Evaluate backend validation independently of browser restrictions. Distinguish UI list state from stored records. | Name requiredness/nonempty edge and maximum 255 are relevant after counting convention review. Price has exclusive cutoff zero and no stated maximum; precision/adjacency needs clarification for 2-/3-point analysis. Category membership is categorical, not an ID range. | Use valid Admin identity and existing category while isolating name/price; use nominal product fields while isolating category. Compare target and unrelated product snapshots immediately after edit and after fresh retrieval. Re-read after delete and rejected writes. |
| FR-12 Mobile context | First verify native login/logout/session wiring without adding screens. Obtain and document the identity/token issued during real Mobile login using controlled traffic capture where feasible. For protected endpoints not callable by current UI, use external API tooling with that session and clearly label the activity API replay. | Token presence/validity and role are categorical; no numeric BVA justified. Expired-token handling may be explored if a legitimately expiring token can be obtained, but expiry duration is not an SRS rule. | Keep request payload otherwise valid to isolate authorization. Distinguish denial response from actual prevention of reads/writes. Preserve native screenshots plus request/response provenance. An API login alone is API coverage, not proof that native Mobile login was exercised. |

For FR-12, the existing native app can supply a non-Admin session and an Admin session through its shared login. It cannot directly invoke the protected CRUD/Admin routes. Capturing and replaying a Mobile-issued JWT would test shared backend authorization in a Mobile-session context, but would not prove the native UI contains those actions. This approach requires scope agreement for the assignment; no dependent execution or source modification is performed here.

### 7.3 Execution prerequisites and review decisions

| Prerequisite / decision | Reason |
| --- | --- |
| Review FR-12 Mobile scope (Q-01) before designing that feature's cases. | Prevent claiming native coverage for a nonexistent Admin screen. |
| Resolve Admin shipping-cancellation interpretation (Q-02). | Avoid inventing an allowed transition. |
| Resolve omission/clearing, price precision, name counting, and response-contract questions where needed (Q-03 through Q-06). | These affect equivalence classes, boundary adjacency, and the oracle. |
| Prepare a reachable Mobile/backend environment without source changes (Q-07). | Current hardcoded LAN base is not verified. |
| Plan controlled data lifecycle before backend startup (Q-08). | Startup runs destructive reinitialization; later evidence must survive restarts. |
| Record browser/device/OS, runtime addresses, source revision, identities and starting fixtures during later execution. | Enables reproducibility and accurate separation of Web, Admin, native Mobile, and API evidence. |

**Task 01 completion:** repository architecture, verified source mappings, SRS rule traceability, implemented endpoint inventory, scope, risks/ambiguities, and testing strategy are documented. Application source and the database were not modified. Execution, test-case design, fixes, and confirmed bug reporting remain outside this completed task.
