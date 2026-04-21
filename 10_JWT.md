# JWT Vulnerabilities — Lab Write-ups

> **Topic:** JSON Web Tokens (JWT)  
> **Platform:** PortSwigger Web Security Academy  
> **Labs Completed:** All  
> **Tools Used:** Burp Suite JWT Editor extension, hashcat, Python (jwt library), CyberChef

---

## What are JWT Vulnerabilities?

JWTs are widely used for authentication and session management in modern web applications and APIs. They consist of three base64url-encoded parts separated by dots: header, payload, and signature. The signature is cryptographically computed from the header and payload using a secret key — it's supposed to make the token tamper-proof.

When JWT implementations are broken, attackers can forge tokens and impersonate any user, including admins. The attack surface includes: accepting unsigned tokens (`alg:none`), algorithm confusion (RS256 → HS256), weak secrets crackable offline, trusting key material embedded in the token itself, and using the `kid` parameter as a filesystem path.

---

## Lab 1 — JWT Authentication Bypass via Unverified Signature

**Level:** Apprentice

**What the lab is about:**  
The server doesn't actually verify the JWT signature at all. It just base64url-decodes the payload and trusts whatever is in it.

**What I did:**  
Decoded my JWT by splitting on `.` and base64url-decoding the payload section. Changed `"sub":"wiener"` to `"sub":"administrator"`. Re-encoded the payload. Reassembled the token with the original header and signature (unchanged) and sent the request.

Gained full admin access. The server never checked the signature.

**Key takeaway:**  
This is the most basic JWT failure — forgetting to actually call the verify function. It's embarrassingly common in custom JWT implementations and poorly configured libraries. Signature verification is the entire point of JWTs. Without it, they're just base64-encoded JSON that anyone can modify.

**Mitigation:**  
Always verify the signature before trusting any data from the token. Use well-maintained JWT libraries and never build your own JWT parsing logic from scratch.

---

## Lab 2 — JWT Authentication Bypass via Flawed Signature Verification (`alg:none`)

**Level:** Apprentice

**What the lab is about:**  
The server accepts the `alg: none` algorithm, which explicitly means "no signature required." You can strip the signature entirely and set the algorithm to none — the server accepts it.

**What I did:**  
Decoded the JWT header: `{"alg":"RS256","typ":"JWT"}`. Changed it to `{"alg":"none","typ":"JWT"}`. Modified the payload to set `"sub":"administrator"`. Re-encoded both parts. Removed the signature while keeping the trailing dot:

```
MODIFIED_HEADER.MODIFIED_PAYLOAD.
```

The server accepted the unsigned token and granted admin access.

**Key takeaway:**  
`alg:none` support must be completely disabled in production. It exists as a spec option for non-sensitive data transfer — it has no place in authentication flows. The application must explicitly whitelist acceptable algorithms and reject everything else, including `none`.

**Mitigation:**  
Whitelist acceptable algorithms server-side (e.g., only `RS256`). Never accept the algorithm from the token header without validation — the token is attacker-controlled.

---

## Lab 3 — JWT Authentication Bypass via Weak Signing Secret

**Level:** Practitioner

**What the lab is about:**  
The JWT is signed with HMAC-SHA256 (HS256) but the secret key is a weak, common string. Since the signature verification is done client-side (the full token including signature is visible), an attacker can crack the secret offline.

**What I did:**  
Used `hashcat` with PortSwigger's JWT secrets wordlist in JWT cracking mode:

```bash
hashcat -a 0 -m 16500 <JWT_TOKEN> /wordlists/jwt-secrets.txt
```

Cracked the secret: `secret1`. Then used the Burp JWT Editor extension to modify the payload (set `sub` to `administrator`) and re-sign the token with the cracked secret. Server accepted it.

**Key takeaway:**  
JWT HMAC secrets must be cryptographically random and long — at least 256 bits (32 bytes) of entropy. Dictionary words, short strings, or anything predictable can be cracked offline in seconds. The full token is always visible to the attacker, making it a perfect offline brute-force target.

**Mitigation:**  
Use a cryptographically secure random secret of at least 256 bits. Prefer asymmetric algorithms (RS256/ES256) so the private key never needs to be shared. Store secrets securely — not in source code.

---

## Lab 4 — JWT Authentication Bypass via JWK Header Injection

**Level:** Practitioner

**What the lab is about:**  
The JWT header can contain a `jwk` parameter — a JSON Web Key — that specifies the public key the server should use to verify the signature. If the server trusts this embedded key without checking it against a known key store, an attacker can inject their own public key and sign tokens with the corresponding private key.

**What I did:**  
Generated a new RSA key pair in Burp's JWT Editor. Modified the JWT to include the public key in the header as a `jwk` parameter, and signed the modified payload (admin username) with my private key:

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "jwk": {
    "kty": "RSA",
    "e": "AQAB",
    "n": "<my public modulus>"
  }
}
```

The server used the embedded `jwk` to verify the signature — and since I signed it with the matching private key, verification passed. Gained admin access.

**Key takeaway:**  
The server should maintain its own trusted public key store. Never use a key supplied within the token itself for verification — that's circular logic. The attacker fully controls the token, including any keys embedded in it.

**Mitigation:**  
Maintain a server-side allowlist of trusted public keys or a JWKS endpoint at a hardcoded internal URL. Ignore `jwk`, `jku`, and `x5u` headers entirely when they appear in untrusted tokens.

---

## Lab 5 — JWT Authentication Bypass via JKU Header Injection

**Level:** Practitioner

**What the lab is about:**  
Similar to JWK injection but instead of embedding the key directly, the `jku` header contains a URL where the server fetches the JSON Web Key Set (JWKS). If the server fetches from any URL without validating it, you can host your own JWKS and make the server use your public key.

**What I did:**  
Generated an RSA key pair in Burp JWT Editor. Created a JWKS JSON file containing my public key and hosted it on the exploit server. Modified the JWT header:

```json
{
  "alg": "RS256",
  "jku": "https://exploit-server.com/my-jwks.json",
  "kid": "my-key-id"
}
```

Signed the token (with admin payload) using my private key. The server fetched my JWKS from the exploit server, found the key matching the `kid`, and successfully verified my signature.

**Key takeaway:**  
`jku` URLs must be validated against a strict allowlist of trusted domains before fetching. Fetching from an arbitrary attacker-controlled URL effectively lets the attacker choose what key to verify against.

**Mitigation:**  
Validate `jku` against a hardcoded whitelist of trusted JWKS endpoints. Better yet, fetch the JWKS from a known internal URL on startup and cache it — don't accept the URL from the token at all.

---

## Lab 6 — JWT Authentication Bypass via Kid Header Path Traversal

**Level:** Practitioner

**What the lab is about:**  
The `kid` (key ID) header is used to look up the signing key from the filesystem — the application reads the key from a file path derived from the `kid` value. If unsanitized, path traversal in `kid` lets you point to any file on the system, including files with known or empty contents.

**What I did:**  
Changed the JWT algorithm to `HS256` and modified the `kid` header value to:

```
../../../dev/null
```

`/dev/null` is always an empty file — its contents are an empty string. I signed the modified token (admin payload) using an empty string as the HMAC secret. The server resolved the `kid` path traversal to `/dev/null`, read the "key" (empty string), and my signature matched perfectly.

**Key takeaway:**  
The `kid` parameter is attacker-supplied and must be treated as untrusted input. Using it directly as a filesystem path is extremely dangerous. `/dev/null` as a key source is a brilliant attack — the attacker controls both the "secret" and the signature.

**Mitigation:**  
Never use `kid` as a filesystem path. Map key IDs to keys using a lookup table in a database or a hardcoded dictionary. Treat `kid` as an opaque identifier only, not a path or query.

---

*JWT vulnerabilities are fascinating because the format is designed to be secure but the security entirely depends on correct implementation. The `alg:none` and unverified signature bugs are developer mistakes. The JWK/JKU injection bugs are spec features that are dangerous by default. Understanding all of these is essential for any API or modern web app assessment.*
