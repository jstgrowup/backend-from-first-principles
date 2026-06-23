### **Technical Notes: Backend Security—Everything You Need to Know**

---

#### **1. The Core Philosophy of Security**

- **Significance:** Security is arguably the most critical aspect of backend engineering. Failure leads to catastrophic financial loss, business destruction, and resource depletion.
- **Security Contexts:**
  - **Browser Security:** Dealing with HTML, cookies, and local storage.
  - **Network Security:** Focused on the transition layer (HTTP/HTTPS, encryption, and compression).
  - **Server Security:** Protecting the Operating System (OS) and the processes running within it.
  - **Backend Security:** Specifically focused on the application code and the vulnerabilities arising from its logic.
- **The Paranoid Mindset:** No application is ever "truly secure" as technology and vulnerabilities evolve. The goal is to build a mindset that constantly asks, **"Where did the developer make an assumption?"**.
- **Common Fatal Assumptions:**
  - User input is clean.
  - The user is who they say they are.
  - Requests are coming only from the intended frontend.
  - No one will inspect network calls or modify parameters.

---

#### **2. Injection Attacks: Confusion Between Code and Data**

Injection occurs when data from one language context (e.g., a user's browser input) crosses a boundary into another language (e.g., SQL, Shell, HTML) and is interpreted as **code** instead of **data**.

**A. SQL Injection (SQLi)**

- **The Vulnerability:** Building database queries by concatenating strings with user input.
- **Mechanism:** Special characters like single quotes (`'`), semicolons (`;`), and hyphens (`--`) are used to break the query logic.
- **Common Payloads:**
  - `' OR '1'='1' --`: Creates a constant true condition to bypass authentication or dump all rows.
  - `' ; DROP TABLE users; --`: Executes a second, destructive command.
- **Advanced SQLi:** Attackers can use `UNION` statements to extract data from other tables, read server files via DB functions, or even execute OS commands.

**B. NoSQL Injection**

- Even without SQL, JSON-based databases like **MongoDB** are vulnerable. Attackers can inject operators like `$ne` (not equal), `$gt` (greater than), or `$exists` to bypass filters if the application passes user-controlled objects directly to queries.

**C. Command Injection**

- **Target:** The server's Operating System.
- **Example:** An image processing service using a CLI tool like `ffmpeg`. If the filename is concatenated into the command, an attacker can append `; rm -rf /` or use pipes (`|`) and ampersands (`&`) to run malicious background processes or spyware.

**D. Prevention: Parameterized Queries & Argument Arrays**

- **The Fix:** Use APIs that separate the query structure from the data.
- **SQL:** Use **Prepared Statements** where user data is passed into "slots" (e.g., `$1` or `?`).
- **OS Commands:** Pass arguments as an array so they are sent directly to the process without being interpreted by a shell.

✅ **Shown in Video: SQL Injection & Parameterized Fix (Go/Pseudo)**

```go
// ❌ DANGEROUS: String Concatenation
query := "SELECT * FROM users WHERE email = '" + userEmail + "'"
// Attacker input: ' OR '1'='1' --
// Becomes: SELECT * FROM users WHERE email = '' OR '1'='1' --'

// ✅ SECURE: Parameterized Query
// The $1 is a slot. The driver ensures the input is treated ONLY as data.
statement := "SELECT * FROM users WHERE email = $1"
db.Query(statement, userEmail)
```

_Demonstrates the separation of the query structure from the runtime user data._

---

#### **3. Authentication (AuthN) & Password Storage**

- **The Goal:** Verifying identity.
- **Developer Tip:** Use an Auth provider (e.g., Clerk, Auth0) for production systems to save time and ensure professional security (revocation, MFA, social auth integration).

**A. Password Storage Rules**

- **Never store plain text:** Insider threats or database breaches will expose every user.
- **Hashing:** A one-way function where the same input always produces the same fixed-length output.
- **Salting:** Adding a random, unique string (**salt**) to the password before hashing to prevent **Rainbow Table** attacks (precomputed maps of common passwords).
- **Work Factor:** Use **Slow Hashing Functions** (Argon2id, BCrypt, Scrypt) rather than general-purpose ones (MD5, SHA-256). These are deliberately slow to make GPU-based brute-forcing (billions of attempts/sec) unfeasible by forcing each hash to take several hundred milliseconds.

🧪 **Generated learning example: Secure Password Hashing (Node.js)**

```javascript
const argon2 = require("argon2");

async function signup(password) {
  // Argon2 handles salting and the 'slow' cost factor automatically
  const hash = await argon2.hash(password, {
    type: argon2.argon2id, // Industry standard
    memoryCost: 2 ** 16, // Adjust based on your server capacity
    timeCost: 3, // The 'Slow' factor
  });
  // Save 'hash' to DB
}

async function login(providedPassword, storedHash) {
  // verify() regenerates the hash using the salt stored in the string
  const isMatch = await argon2.verify(storedHash, providedPassword);
  return isMatch;
}
```

_Demonstrates industry-standard slow hashing using Argon2id to prevent brute-force attacks._

---

#### **4. Sessions vs. JWT (Stateless vs. Stateful)**

| Feature         | Stateful (Sessions)                   | Stateless (JWT)                                 |
| :-------------- | :------------------------------------ | :---------------------------------------------- |
| **Storage**     | DB/Redis stores session metadata.     | Stored entirely on the client.                  |
| **Scalability** | Harder; requires shared cache.        | Easier; no DB lookup needed per request.        |
| **Revocation**  | **Easy:** Delete the session from DB. | **Hard:** Token remains valid until it expires. |
| **Security**    | Opaque ID; hides data.                | Base64 encoded; payload is visible to anyone.   |

**A. Cookie Security Flags**

- **`HttpOnly`:** Prevents JavaScript from reading the cookie (vital against XSS).
- **`Secure`:** Only sends cookie over HTTPS.
- **`SameSite`:** `Strict` or `Lax` prevents the browser from sending cookies during cross-site requests (vital against CSRF).

**B. JWT Best Practices**

- Never store sensitive data in the payload (it's just Base64).
- Use short expiration times (minutes/hours).
- Implement **Refresh Tokens** stored on the server to handle session renewal.

---

#### **5. Authorization (AuthZ): Permission Logic**

AuthZ determines what an authenticated user is allowed to do.

- **Broken Object Level Authorization (BOLA/IDOR):** An attacker accesses resources they don't own by changing an ID (e.g., `/invoices/105`).
  - **The Fix:** Always include a `user_id` check in the database query at the repository level, not just at the routing layer.
  - **Privacy Tip:** Use **UUIDs** instead of sequential IDs (1, 2, 3) to make resources unguessable.
- **Broken Function Level Authorization (BFLA):** Regular users accessing administrative functions (e.g., `/admin/delete-all`).
  - **The Fix:** Implement role-based (Admin/Member) middlewares at the entry point.
- **Information Leakage:** If a user requests a resource they don't own, return a `404 Not Found` instead of `403 Forbidden`. A 403 confirms the resource exists, which helps attackers in **Enumeration** or **Social Engineering** attacks.

---

#### **6. XSS, CSRF & Security Headers**

- **XSS (Cross-Site Scripting):** Malicious JS executing in a user's browser.
  - **Stored XSS:** Saving a `<script>` tag in a database (like a comment) that executes for every user who views it.
  - **Prevention:** Sanitize user input and use a **Content Security Policy (CSP)** header to tell the browser which scripts to trust and block inline code.
- **CSRF (Cross-Site Request Forgery):** Tricking a browser into making a request to a site where the user is already logged in.
  - **Note:** Modern `SameSite` cookie defaults and CORS have made this less of a threat today.
- **Security Headers:** Use modern middleware (like **Helmet** for Node.js) to set headers like `X-Frame-Options` (prevents **Clickjacking**).

---

#### **7. Production Misconfigurations**

- **Secrets Management:** Never commit API keys or DB passwords to Git/GitHub. Use Environment Variables or Secret Managers (AWS Parameter Store, HashiCorp Vault).
- **Debug Mode:** Never leave `DEBUG` level logging on in production. It leaks stack traces, internal file paths, and database query structures.

---

#### **8. The Layered Defense Summary**

1.  **Input Validation:** Strictly validate every structure at the entry point.
2.  **Parameterized Operations:** Never concatenate strings for SQL or Shell commands.
3.  **Authorization:** Check permissions at the point of data access (Repository Layer).
4.  **Security Headers:** Use CSP, HSTS, and SameSite cookies.
5.  **Monitoring:** Log failed AuthZ attempts as "Breach Events" and set alerts.

**Resources for Further Study:**

- **PortSwigger Academy:** Free, high-quality labs for web security.
- **OWASP Top 10:** The industry standard list of current vulnerabilities.
- **OWASP Cheat Sheet Series:** Practical guides for implementation.
