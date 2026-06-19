### **Technical Notes: Authentication and Authorization for Backend Engineers**

---

#### **1. Definitions: The "Who" vs. The "What"**

- **Authentication (AuthN):** The process of assigning an identity to a subject. It answers the question: **"Who are you?"** in a given context (platform, OS, phone).
- **Authorization (AuthZ):** The process of determining permissions or capabilities. It answers the question: **"What can you do?"** in that context.

---

#### **2. Historical Evolution of Authentication**

- **Implicit Trust (Pre-Industrial):** Based on human contextual trust and recognition (e.g., a Village Elder vouching for someone). Deals were sealed with handshakes. It failed to scale as populations grew.
- **Explicit Proofs (Medieval):**
  - **Wax Seals:** The first widely adopted authentication tokens. They relied on "Something you possess.".
  - **Bypass Attacks:** Forging seals was the early version of modern authentication bypass attacks.
  - **Advancements:** Watermarks and encrypted codes in trade documentation set the foundation for cryptographic thinking.
- **Passphrases (Industrial Revolution):**
  - **Telegraph Era:** Introduction of pre-agreed passphrases (shared secrets).
  - **Shift in Principle:** Moved from "Something you possess" to **"Something you know."**.
- **Digital Era (Mainframes):**
  - **1961 MIT Project MAC:** Introduced passwords for multi-user systems (CTSS) to prevent data sharing between users.
  - **Genesis of Secure Storage:** A researcher once printed the plaintext password file; this incident motivated the philosophy of hashing and secure storage.
- **Cryptographic Research (1970s):**
  - **Diffie-Hellman Key Exchange:** Enabled establishing shared secrets over untrusted mediums.
  - **PKI (Public Key Infrastructure):** Became the backbone of modern protocols.
  - **Kerberos:** Introduced ticket-based authentication, a precursor to modern token systems.
- **Modern Expansion (1990s - 21st Century):**
  - **Multiactor Authentication (MFA):** Combines Something you know (password), Something you have (OTP generator/card), and Something you are (biometrics).
  - **Biometrics:** Uses pattern recognition (fingerprints, retina) but faces challenges like false positives/negatives and template security.
  - **Frameworks:** Emergence of OAuth 2.0, JWT, Zero Trust Architecture, and Passwordless systems (WebAuthn).
- **The Future:**
  - **Decentralized Identity:** Built using Blockchain.
  - **Post-Quantum Cryptography:** Developing algorithms (like RSA alternatives) that can withstand quantum computers, which are fast enough to break current cryptographic standards.

---

#### **3. Core Components: Sessions, JWTs, and Cookies**

**A. Sessions (Stateful)**

- **Origin:** HTTP is **stateless** by design (no memory of past interactions), which was fine for static sites but failed for dynamic needs like e-commerce carts.
- **Mechanism:**
  1.  User logs in; Server creates a **Session ID**.
  2.  Server stores Session ID + User Data in a **Persistent Store** (DB or Redis/In-memory).
  3.  Session ID is sent to the browser as a **Cookie**.
  4.  Subsequent requests include the cookie, allowing the server to fetch user data.
- **Storage Evolution:** File-based -> Database-backed -> Distributed In-memory (Redis/Memcached) for speed and scalability.

**B. JWT (JSON Web Tokens - Stateless)**

- **Origin:** Created to solve session overhead (memory costs) and replication latency in globally distributed systems.
- **Structure:** Base64 encoded string with three parts:
  1.  **Header:** Metadata (e.g., signing algorithm).
  2.  **Payload:** Contains **Claims** (data like `sub` for User ID, `iat` for Issued At, and `role`).
  3.  **Signature:** Used to verify the token hasn't been tampered with using a **Secret Key**.
- **Pros:** Stateless (no server storage), scalable (shared secret across microservices), and portable.
- **Cons:** Hard to revoke. If a JWT is stolen, the attacker can impersonate the user until it expires, unless the server changes its global secret (which logs out _every_ user).
- **Hybrid Approach:** Using JWTs but maintaining a **Blacklist** in Redis to temporarily revoke specific tokens.

**C. Cookies**

- **Definition:** A mechanism for the server to store a string/value in the user's browser.
- **Security:** Browsers ensure cookies are only accessible to the server that set them. The **HTTP-only** flag prevents JavaScript from accessing the value.

---

#### **4. Major Authentication Types**

| Type           | Best For               | Workflow                                                                    |
| :------------- | :--------------------- | :-------------------------------------------------------------------------- |
| **Stateful**   | Standard Web Apps/SAS  | Session ID stored in Redis; checked every request.                          |
| **Stateless**  | Scalable APIs / Mobile | Self-contained JWT; verified via secret key without DB lookup.              |
| **API Key**    | Machine-to-Machine     | Cryptographically random string for programmatic access (e.g., OpenAI API). |
| **OAuth/OIDC** | 3rd Party Logins       | Delegating access/identity (e.g., "Sign in with Google").                   |

---

#### **5. OAuth 2.0 and OpenID Connect (OIDC)**

- **The Delegation Problem:** Initially solved by sharing passwords, which was disastrous (full access, impossible to revoke).
- **OAuth Solution:** Sharing **Tokens** with specific permissions instead of passwords.
- **Key Roles:** Resource Owner (User), Client (App requesting access), Resource Server (e.g., Google Photos), Authorization Server (Issues tokens).
- **OAuth 2.0 Flows:**
  - **Auth Code Flow:** For server-side apps.
  - **Implicit Flow:** For browsers (now discouraged).
  - **Client Credentials:** Machine-to-machine.
  - **Device Code:** For limited-input devices (Smart TVs).
- **OpenID Connect (OIDC):** An identity layer on top of OAuth 2.0. It introduces the **ID Token** (a JWT) to provide user profile info (name, email).

---

#### **6. Authorization & RBAC**

- **Concept:** Providing specific permissions to specific users (e.g., not everyone can access the "Dead Zone" notes in a recycle bin).
- **RBAC (Role-Based Access Control):**
  - Roles are assigned to users (e.g., Admin, Moderator, User).
  - Permissions (Read, Write, Delete) are assigned to Roles.
  - **Forbidden (403):** The error sent when a user is authenticated but lacks the permission for a resource.

---

#### **7. Critical Security Warnings**

- **Error Messages:** Never send specific messages like "User not found" or "Incorrect password." This helps attackers confirm valid usernames. Always send **Generic Messages** (e.g., "Authentication failed").
- **Timing Attacks:**
  - Attackers measure response times. If "User not found" returns faster than "Incorrect password" (due to hashing time), they know the username is valid.
  - **Defense:** Use **Constant Time Operations** or **Simulate Delays** to equalize response times.

---

### **Code Snippets**

🧪 **Generated learning example: Implementing Generic Error Responses**

```javascript
// Security Best Practice: Preventing information leakage via error messages
app.post("/login", async (req, res) => {
  try {
    const user = await findUserByEmail(req.body.email);

    // ❌ WRONG: res.status(404).send("User not found");
    // This confirms the email doesn't exist to an attacker.
    if (!user)
      return res.status(401).json({ message: "Authentication failed" });

    const isMatch = await comparePassword(req.body.password, user.hash);

    // ❌ WRONG: res.status(401).send("Incorrect password");
    // This confirms the email exists but the password is wrong.
    if (!isMatch)
      return res.status(401).json({ message: "Authentication failed" });

    // Success logic...
  } catch (err) {
    res.status(401).json({ message: "Authentication failed" });
  }
});
```

_This demonstrates using a single, ambiguous message for all auth failures to protect against enumeration attacks._

🧪 **Generated learning example: Defending Against Timing Attacks**

```go
// Go example: Simulating a response delay to prevent timing analysis
func LoginHandler(w http.ResponseWriter, r *http.Request) {
    start := time.Now()

    // Standard Auth logic
    isAuthenticated := authenticateUser(r.FormValue("user"), r.FormValue("pass"))

    // Ensure the response always takes at least 200ms
    elapsed := time.Since(start)
    if elapsed < 200*time.Millisecond {
        time.Sleep(200*time.Millisecond - elapsed)
    }

    if !isAuthenticated {
        http.Error(w, "Authentication failed", http.StatusUnauthorized)
        return
    }
    // Proceed with success...
}
```

_This implements the "Fake Delay" strategy to hide the time difference between finding a user and hashing a password._

🧪 **Generated learning example: RBAC Middleware Logic**

```javascript
// Middleware to enforce Role-Based Access Control (RBAC)
const authorize = (requiredRole) => {
  return (req, res, next) => {
    // Assuming user info was attached to req by an earlier AuthN middleware
    const userRole = req.user.role;

    if (userRole !== requiredRole && userRole !== "admin") {
      return res.status(403).json({
        error: "Forbidden",
        message: "You do not have permission to access this resource.",
      });
    }
    next();
  };
};

// Usage: Only admins can access the 'dead-zone'
app.get("/api/admin/dead-zone", authorize("admin"), (req, res) => {
  res.send("Accessing sensitive archived data...");
});
```

_This shows the typical workflow of using a role deduced from a token to gate specific API endpoints._
