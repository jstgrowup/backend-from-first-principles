### **Technical Notes: Complete REST API Design**

---

#### **1. Introduction and The Scalability Crisis**

- **The Motivation:** API design is a core skill for backend engineers; following a consistent standard eliminates confusion and allows focus on business logic.
- **Historical Context:**
  - **1990:** Tim Berners-Lee invented the World Wide Web, including **URIs, HTTP, HTML**, the first web server, the first browser, and a "what you see is what you get" (WYSIWYG) editor.
  - **The Crisis:** The web's exponential growth led to a scalability breakdown as the original architecture wasn't built for millions of users.
  - **1993–2000:** **Roy Fielding** (co-founder of Apache) addressed this in his PhD dissertation, proposing **REST (Representational State Transfer)** to standardize web architecture.

#### **2. The 6 Constraints of REST**

To achieve scalability, Roy Fielding proposed these rules:

1.  **Client-Server:** Separates the UI (frontend) from data storage (backend), allowing each to evolve independently.
2.  **Uniform Interface:** Standardizes how components communicate. Includes resource identification, manipulation through representation, self-descriptive messages, and **HATEOAS** (Hypermedia as the Engine of Application State).
3.  **Layered System:** Architecture uses hierarchical layers; a layer only interacts with the one immediately below it. This enables the use of **load balancers and proxy servers** without affecting core logic.
4.  **Cache:** Servers must label responses as cacheable or non-cacheable to reduce load and improve response times.
5.  **Stateless:** Each request must contain **all information** needed for the server to process it. The server stores no client context between requests, improving reliability and scalability.
6.  **Code on Demand (Optional):** Servers can temporarily extend client functionality by transferring executable code (e.g., JavaScript).

---

#### **3. Defining REST (Representational State Transfer)**

- **Representational (R):** Resources (data/objects) are represented in formats like **JSON** (standard for server-to-server), **HTML** (for browsers), or **XML**.
- **State (S):** Refers to the current attributes or condition of a resource (e.g., items and quantities in a shopping cart).
- **Transfer (T):** The movement of resource representations between client and server via **HTTP methods** (GET, POST, etc.).

---

#### **4. Anatomy of an API URL**

- **URL Structure:** Includes the **Scheme** (http/https), **Authority** (domain/subdomain), **Resource/Path**, **Query Parameters**, and **Fragments**.
- **Industry Standards for API Routes:**
  - **Subdomain:** Use `api.example.com`.
  - **Versioning:** Always include versioning in the path (e.g., `/V1/` or `/V2/`) to manage breaking changes.
  - **Plural Nouns:** Resources in the path must **always be plural** (e.g., `/books`, not `/book`), even when fetching a single item.
  - **Formatting:** Use **small case only** to avoid environment mismatches. Do not use spaces or underscores; replace them with **hyphens** (Kebab-case) for readability (e.g., `harry-potter`).
  - **Hierarchical Relationship:** The forward slash (`/`) represents hierarchy (e.g., `/organizations/ID/projects` means projects belonging to a specific org).

---

#### **5. HTTP Methods and Idempotency**

**Idempotency** means performing an action multiple times has the same effect as performing it once.

| Method     | Purpose          | Idempotent? | Technical Detail                                                                                          |
| :--------- | :--------------- | :---------- | :-------------------------------------------------------------------------------------------------------- |
| **GET**    | Retrieve data    | **Yes**     | Multiple fetches cause no side effects on the server.                                                     |
| **PUT**    | Replace resource | **Yes**     | Completely replaces the entity with the client's payload.                                                 |
| **PATCH**  | Partial update   | **Yes**     | Updates only specific fields. Subsequent calls with the same data yield the same state.                   |
| **DELETE** | Remove resource  | **Yes**     | The first call deletes; subsequent calls return an error (404), but the state (deleted) remains the same. |
| **POST**   | Create/Custom    | **No**      | Each call creates a new resource (e.g., a new book entry with a unique ID).                               |

- **Tip:** While often used interchangeably, use **PATCH** for partial updates and **PUT** for complete replacements to stick to semantic standards.

---

#### **6. Practical API Design Workflow**

1.  **Analyze UI/Wireframes:** Start with Figma or user stories to understand how users interact with data.
2.  **Identify Resources:** Extract **nouns** from requirements (e.g., Projects, Users, Tasks, Organizations).
3.  **Database Schema:** Design tables based on these resources (Organization Table → Project Table → Task Table).
4.  **Interface Design:** Before coding logic, design the API interface using tools like **Insomnia** or **Postman**.

---

#### **7. Designing CRUD Endpoints**

**A. List & Pagination (GET `/resources`)**

- **Pagination:** Necessary to prevent overwhelming the network or server with large datasets.
- **Serialization Overhead:** Converting thousands of objects to JSON is resource-heavy and introduces perceived delay.
- **Standard Query Params:** `limit` (count per page) and `page` (specific portion).
- **Standard Metadata Response:** Should include `data` (the list), `total` (total count in DB), `page` (current), and `totalPages`.
- **Sorting:** Support `sortBy` (field name) and `sortOrder` (ascending/descending).
- **Filtering:** Use query params based on field names (e.g., `?status=active`).

**B. Create (POST `/resources`)**

- **Status Code:** Returns **201 Created**.
- **Response:** Returns the newly created entity including server-generated fields like `ID` and `createdAt`.

**C. Update (PATCH `/resources/:id`)**

- **Status Code:** Returns **200 OK**.
- **Best Practice:** In Single Page Applications (SPAs), use **PATCH** for partial JSON payloads.

**D. Delete (DELETE `/resources/:id`)**

- **Status Code:** Returns **204 No Content**. The response body is typically empty.

**E. Custom Actions (POST `/resources/:id/action`)**

- **Definition:** Actions that don't fit CRUD (e.g., `archive`, `clone`, `send-email`).
- **Design:** Append the action to the resource path: `POST /organizations/5/archive`.
- **Method:** **POST** is used as an "open-ended" method for these custom operations.

---

#### **8. Professional Standards & Best Practices**

- **Consistency:** Use the same patterns across all resources (e.g., if one uses `description`, don't use `desc` elsewhere).
- **JSON Casing:** Use **camelCase** for all JSON keys in payloads and responses.
- **Sane Defaults:** The server should assume logical values if the client omits them (e.g., `page=1`, `limit=10`, `status=active` for new orgs).
- **No Abbreviations:** Use `description` instead of `DSC` or `DESC` to remain intuitive for external developers.
- **Documentation:** Always maintain an interactive playground like **Swagger (OpenAPI)**.
- **Error Handling:**
  - **404 Not Found:** Use for specific resource requests (GET/PATCH/DELETE by ID) if the ID doesn't exist.
  - **200 OK (Empty Array):** If a **list API** has no results for a filter, return an empty array, **not a 404**.

---

### **Code Snippets**

✅ **Shown in video: Typical Paginated Response Structure**

```json
{
  "data": [
    { "id": 5, "name": "Org Five", "status": "active" },
    { "id": 4, "name": "Org Four", "status": "active" }
  ],
  "total": 5,
  "page": 1,
  "totalPages": 3
}
```

_Demonstrates the metadata fields required for the client to handle pagination UI._

🧪 **🧪 Generated learning example: Consistent API Routing Patterns**

```http
-- LIST (with Pagination/Sorting) --
GET /api/v1/projects?page=2&limit=10&sortBy=name&sortOrder=asc

-- CREATE --
POST /api/v1/projects
Content-Type: application/json
{
  "name": "E-commerce Redesign",
  "organizationId": "org_123",
  "description": "Upgrading the legacy storefront."
}

-- CUSTOM ACTION (Cloning) --
POST /api/v1/projects/proj_999/clone
```

_This demonstrates plural naming, versioning, and the difference between standard CRUD and custom actions._

🧪 **🧪 Generated learning example: Server-Side "Sane Defaults" Logic**

```javascript
// Example logic for a List API Handler
const listResources = (req, res) => {
  // Applying Sane Defaults if client parameters are missing
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 20;
  const sortBy = req.query.sortBy || "createdAt";
  const sortOrder = req.query.sortOrder || "desc";

  // Logic to fetch from DB using these parameters...
};
```

_Illustrates the principle of providing defaults for page, limit, and sorting to improve client DX._
