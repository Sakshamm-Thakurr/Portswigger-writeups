# Information Disclosure — Lab Write-ups

> **Topic:** Information Disclosure  
> **Platform:** PortSwigger Web Security Academy  
> **Labs Completed:** All  
> **Tools Used:** Burp Suite, Browser DevTools, robots.txt, source code review

---

## What is Information Disclosure?

Information disclosure is when an application unintentionally exposes sensitive data — credentials, internal paths, version numbers, source code, debug output, or configuration details. On its own it might seem minor, but in practice it's almost always the first step in a deeper attack chain. You find a version number → look up CVEs → exploit a known vulnerability.

---

## Lab 1 — Information Disclosure in Error Messages
**Level:** Apprentice

**What the lab is about:**  
The application throws a verbose error message that includes the exact version of a third-party framework being used.

**What I did:**  
Triggered an error by passing an unexpected data type to the `productId` parameter (sent a string instead of an integer). The full stack trace appeared in the response, including the framework name and version number.

Searched that version for known CVEs and found a publicly documented exploit.

**Key takeaway:**  
Verbose error messages in production are a security issue, not just a code quality one. Stack traces and version disclosure should never reach end users.

**Mitigation:**  
Use generic error pages in production. Log details server-side but return only a reference ID to the user.

---

## Lab 2 — Information Disclosure on Debug Page
**Level:** Apprentice

**What the lab is about:**  
A debug page was left accessible at a non-obvious URL. It exposes environment variables including the `SECRET_KEY` used for cryptographic operations.

**What I did:**  
Noticed a comment in the page source referencing a file path. Navigated to `/cgi-bin/phpinfo.php` — found a full PHP info dump with environment variables, including `SECRET_KEY`.

**Key takeaway:**  
`phpinfo()`, debug endpoints, and `/actuator` endpoints (Spring Boot) are commonly forgotten. Always check these during recon.

---

## Lab 3 — Source Code Disclosure via Backup Files
**Level:** Apprentice

**What the lab is about:**  
A text editor backup file (a `.bak` or `~` suffixed file) was accidentally left in the web root. It exposes application source code including hardcoded database credentials.

**What I did:**  
Noticed the app was running Java. Appended `~` to various URLs — developers' text editors (like gedit, emacs) create these backup files:
```
/backup/ProductTemplate.java.bak
```
Downloaded the backup file and found a hardcoded database password in the source.

**Key takeaway:**  
Text editor backup files, `.swp` files, `.orig` files — all of these can end up deployed to production. Web server configs should explicitly deny access to these extensions.

---

## Lab 4 — Authentication Bypass via Information Disclosure
**Level:** Apprentice

**What the lab is about:**  
The admin panel is protected, but a TRACE request reveals that an internal proxy adds a custom HTTP header `X-Custom-IP-Authorization` for requests from the local machine. You can spoof this header.

**What I did:**  
Sent a `TRACE /admin` request. The response echoed back all headers — including `X-Custom-IP-Authorization: 1.3.3.7` added by the proxy. 

Resent the request as `GET /admin` with the header:
```
X-Custom-IP-Authorization: 127.0.0.1
```
Gained access to the admin panel.

**Key takeaway:**  
The `TRACE` method should almost always be disabled. It reflects request headers back and can expose internal proxy headers, session tokens, and other sensitive data.

---

## Lab 5 — Information Disclosure in Version Control History
**Level:** Practitioner

**What the lab is about:**  
The `.git` directory was accidentally deployed to production. You can download the entire Git history and recover deleted files — like a config file that had a password in it before being "removed" in a later commit.

**What I did:**  
Confirmed `.git` was accessible by visiting `/.git/HEAD`. Used `wget --mirror` (or Burp to manually fetch files) to download the git object store. Reconstructed the repo locally using `git checkout` and browsed the commit history:
```
git log --oneline
git show <commit-hash>
```
Found a commit that deleted `admin.conf` — diffed it and recovered the admin password from the deleted file.

**Key takeaway:**  
`.git`, `.svn`, `.env`, `.DS_Store` — all of these exposed in web root are serious. Deployment pipelines should explicitly exclude them. One of the most common findings in real bug bounties.

---
---

# Business Logic Vulnerabilities — Lab Write-ups

> **Topic:** Business Logic  
> **Platform:** PortSwigger Web Security Academy  
> **Labs Completed:** All  
> **Tools Used:** Burp Suite (Repeater, Intercept), logical reasoning

---

## What are Business Logic Vulnerabilities?

Unlike injection flaws, business logic vulnerabilities aren't about malformed input — they're about using the application exactly as designed, but in ways the developers didn't anticipate. The app works correctly from a technical standpoint; the flaw is in the assumptions baked into the logic.

These are hard to find with scanners. You need to actually understand how the application is supposed to work and then ask: "what happens if I do the opposite of what's expected?"

---

## Lab 1 — Excessive Trust in Client-Side Controls
**Level:** Apprentice

**What the lab is about:**  
Product prices are sent from the client in the purchase request. The server trusts whatever price it receives. You can just set the price to whatever you want.

**What I did:**  
Added a product to the cart. Intercepted the POST request that added the item. The body included a `price` field. Changed it to `1` (1 cent).

Completed the purchase. The server accepted the client-supplied price.

**Key takeaway:**  
Never trust the client. Prices, quantities, discounts, roles — all should be looked up server-side at the time of the transaction, not passed from the client.

---

## Lab 2 — High-Level Logic Vulnerability
**Level:** Apprentice

**What the lab is about:**  
The app validates that you don't add more items than you can afford, but doesn't validate negative quantities. Adding -1 of a cheap item reduces the cart total.

**What I did:**  
Added an expensive item to the cart. Then added a negative quantity of a cheap item:
```
productId=1&quantity=-100
```
The cart total dropped dramatically (negative items subtracted from the total). Completed the purchase with a near-zero balance.

**Key takeaway:**  
Validate input ranges on the server, not just the client. Negative quantities, negative prices, zero values — all should be explicitly rejected at the business logic layer.

---

## Lab 3 — Low-Level Logic Flaw (Integer Overflow)
**Level:** Practitioner

**What the lab is about:**  
The cart total is stored as a 32-bit integer. Adding enough items causes it to overflow and wrap around to a large negative number. Once negative, you've effectively got infinite money.

**What I did:**  
Used Burp Intruder to repeat adding the maximum quantity (99) of a high-priced item hundreds of times. After enough iterations, the total wrapped to a negative number due to integer overflow. Adjusted with cheap items to get the total between $0 and my available funds, then checked out.

**Key takeaway:**  
Integer overflow in financial calculations is a real vulnerability class. Currency and pricing fields should use appropriate data types (like `BigDecimal` in Java) and have reasonable maximum limits enforced.

---

## Lab 4 — Inconsistent Security Controls
**Level:** Apprentice

**What the lab is about:**  
The admin panel requires a company email address (`@dontwannacry.com`). But the app lets you change your email address after registration. It validates the email during registration but not during the change.

**What I did:**  
Registered with any email. Then changed my email to `attacker@dontwannacry.com` in the account settings. The change was accepted without verification. Now had access to the admin panel.

**Key takeaway:**  
Security controls applied at registration need to be consistently enforced throughout the account lifecycle. Privilege checks based on email domain need to be re-validated whenever the email changes.

---

## Lab 5 — Flawed Enforcement of Business Rules
**Level:** Apprentice

**What the lab is about:**  
The app has two discount codes — a sign-up code and a newsletter code. Each can only be used once. But you can alternate between them indefinitely.

**What I did:**  
Applied code 1 → applied code 2 → applied code 1 again → applied code 2 again. The "already used" check only prevented using the same code twice consecutively, not alternating between two codes. Applied them back and forth until the item was free.

**Key takeaway:**  
Business rule enforcement needs to think about sequences and combinations, not just individual actions. Discount code systems need to track all applied codes across a session, not just the last one.

---

## Lab 6 — Infinite Money Logic Flaw
**Level:** Practitioner

**What the lab is about:**  
A gift card system combined with a store credit mechanism creates an infinite money loop: buy gift card → redeem for store credit → buy another gift card → repeat.

**What I did:**  
Bought a £10 gift card using a 30% discount code → paid £7. Redeemed the gift card for £10 store credit. Net gain: £3 per cycle. Automated this with a Burp macro to repeat the loop until I had enough credit to buy the target item.

**Key takeaway:**  
Complex multi-feature interactions create emergent vulnerabilities. Each individual feature may be secure, but their combination creates a flaw. You need to test not just features in isolation but their interactions.

---

## Lab 7 — Authentication Bypass via Flawed State Machine
**Level:** Practitioner

**What the lab is about:**  
During the login flow, a role selection step sets your user role. If you drop the role-selection request (so it never completes), the application falls back to a default admin role.

**What I did:**  
Logged in and intercepted the POST request for the role selection page. Dropped the request entirely in Burp. Navigated directly to `/admin`. Because the role selection never completed, the application defaulted to a privileged role.

**Key takeaway:**  
Multi-step flows need to validate state at every step. "What's the default state if a step is skipped?" is an important question. Default to least privilege, never most privilege.

---
---

# File Upload Vulnerabilities — Lab Write-ups

> **Topic:** File Upload  
> **Platform:** PortSwigger Web Security Academy  
> **Labs Completed:** All  
> **Tools Used:** Burp Suite (Repeater, Intercept), PHP webshells

---

## What are File Upload Vulnerabilities?

File upload features are high-risk because if you can get the server to execute a file you uploaded, you have remote code execution. Even if execution isn't possible, you might achieve stored XSS, path traversal, or denial of service through oversized files.

The attack surface includes: avatar uploads, document uploads, import features, backup restore features, and any "attach a file" functionality.

---

## Lab 1 — Remote Code Execution via Web Shell Upload
**Level:** Apprentice

**What the lab is about:**  
The avatar upload feature has no file type validation. You can upload a PHP file and it will be served and executed by the server.

**What I did:**  
Created a simple PHP webshell:
```php
<?php echo system($_GET['cmd']); ?>
```

Uploaded it as `webshell.php`. Found where uploads are stored (usually `/files/avatars/`). Fetched:
```
GET /files/avatars/webshell.php?cmd=cat+/home/carlos/secret
```

Got the secret file contents back in the response.

**Key takeaway:**  
Upload functionality without content-type validation is critical severity — it's a direct path to RCE. Always validate file type server-side based on actual content (magic bytes), not just the extension or MIME type.

---

## Lab 2 — Web Shell Upload via Content-Type Restriction Bypass
**Level:** Apprentice

**What the lab is about:**  
The application checks the `Content-Type` header but not the actual file content. You can upload a PHP file by spoofing the content type to `image/jpeg`.

**What I did:**  
Uploaded `webshell.php` but changed the `Content-Type` header in Burp from `application/octet-stream` to `image/jpeg`. The server accepted it — checked the header, not the file content. Executed the shell the same way as Lab 1.

**Key takeaway:**  
Client-supplied `Content-Type` is completely untrusted. Server-side validation must inspect actual file contents (magic bytes/file signatures), not just the MIME type header.

---

## Lab 3 — Web Shell Upload via Path Traversal
**Level:** Practitioner

**What the lab is about:**  
The upload directory has a server config preventing script execution. But by using path traversal in the filename, you can write the file to a parent directory where execution is allowed.

**What I did:**  
Set the filename in the upload request to:
```
../webshell.php
```

The file was saved one directory above the intended uploads folder — a location where PHP execution was allowed. Then accessed it at the correct path.

**Key takeaway:**  
Filename sanitization must strip or reject directory traversal sequences. Never pass user-supplied filenames directly to filesystem operations.

---

## Lab 4 — Web Shell Upload via Extension Blacklist Bypass
**Level:** Practitioner

**What the lab is about:**  
Common PHP extensions (`.php`, `.php3`, `.php4`) are blacklisted. But `.php5` or `.phtml` are not. Or you can upload a `.htaccess` file to redefine what extensions get executed.

**What I did:**  
Uploaded a `.htaccess` file with:
```
AddType application/x-httpd-php .l33t
```

Then uploaded `webshell.l33t` — which the server now treated as PHP. Got RCE.

**Key takeaway:**  
Extension blacklists are fundamentally flawed — there are always obscure extensions or configuration overrides. Whitelist allowed extensions instead.

---

## Lab 5 — Web Shell Upload via Obfuscated File Extension
**Level:** Practitioner

**What the lab is about:**  
The blacklist checks the extension but can be confused by various obfuscation tricks: case variation, double extensions, null bytes, URL encoding.

**What I did:**  
Tried multiple bypasses:
- `webshell.PHP` (uppercase — some systems are case-sensitive)
- `webshell.php.jpg` (double extension — some servers execute based on first matching extension)
- `webshell.php%00.jpg` (null byte — truncates the string in some older systems)
- `webshell.pHp` (mixed case)

The null byte approach worked in this lab. Uploaded `webshell.php%00.jpg` — the blacklist saw `.jpg`, but the filesystem saved it as `webshell.php`.

**Key takeaway:**  
Extension validation needs to be robust. Extract the true extension after decoding and normalizing the filename. Reject anything that doesn't match your whitelist exactly.

---

## Lab 6 — Remote Code Execution via Polyglot Web Shell Upload
**Level:** Practitioner

**What the lab is about:**  
The server validates actual file content (magic bytes) to verify it's a real image. A polyglot file is both a valid image AND contains valid PHP code.

**What I did:**  
Used `exiftool` to inject PHP code into the metadata of a legitimate JPEG:
```bash
exiftool -Comment="<?php echo system(\$_GET['cmd']); ?>" image.jpg -o webshell.php
```

The file starts with the JPEG magic bytes (`FF D8 FF`) so it passes image validation. The PHP interpreter finds the `<?php` tag in the metadata and executes it.

**Key takeaway:**  
Polyglot files defeat magic byte checks. Defense-in-depth is needed: image re-processing (resizing strips metadata), serving files from a separate domain, and using a CDN that strips code from images.

---
---

# JWT Vulnerabilities — Lab Write-ups

> **Topic:** JSON Web Tokens (JWT)  
> **Platform:** PortSwigger Web Security Academy  
> **Labs Completed:** All  
> **Tools Used:** Burp Suite JWT Editor extension, Python (jwt library), CyberChef

---

## What are JWT Vulnerabilities?

JWTs are widely used for authentication and session management. They consist of three base64url-encoded parts: header, payload, and signature. The signature is supposed to prevent tampering — but flawed implementations let attackers forge tokens and impersonate any user, including admins.

Key attack vectors: alg:none, algorithm confusion (RS256 → HS256), weak secrets, missing signature verification, kid injection.

---

## Lab 1 — JWT Authentication Bypass via Unverified Signature
**Level:** Apprentice

**What the lab is about:**  
The server doesn't actually verify the JWT signature at all. It just decodes and trusts the payload.

**What I did:**  
Decoded my JWT token (base64url decode the payload). Changed `"sub":"wiener"` to `"sub":"administrator"`. Re-encoded it. Sent the request with the modified token — no need to re-sign because the server doesn't check.

Gained admin access.

**Key takeaway:**  
This is embarrassingly common in poorly implemented JWT libraries or custom JWT code. Signature verification is mandatory. Libraries that default to not verifying should not be used.

---

## Lab 2 — JWT Authentication Bypass via Flawed Signature Verification
**Level:** Apprentice

**What the lab is about:**  
The server accepts the `alg: none` algorithm, meaning "no signature." You can strip the signature entirely and set the algorithm to none.

**What I did:**  
Decoded the header: `{"alg":"RS256","typ":"JWT"}`. Changed to `{"alg":"none","typ":"JWT"}`. Modified the payload to set the username to `administrator`. Re-encoded header and payload. Removed the signature (kept the trailing dot):
```
HEADER.PAYLOAD.
```
Server accepted it.

**Key takeaway:**  
`alg:none` support should be disabled. The application must explicitly whitelist acceptable algorithms — never use whatever the token header claims.

---

## Lab 3 — JWT Authentication Bypass via Weak Signing Secret
**Level:** Practitioner

**What the lab is about:**  
The JWT is signed with HMAC-SHA256 but the secret key is extremely weak — a common word. You can crack it with a wordlist.

**What I did:**  
Used `hashcat` with the PortSwigger JWT secret list:
```bash
hashcat -a 0 -m 16500 JWT_TOKEN /wordlists/jwt-secrets.txt
```
Cracked the secret: `secret1`. Then used the Burp JWT Editor to re-sign a modified token (admin username) with the cracked secret.

**Key takeaway:**  
JWT HMAC secrets must be cryptographically random and long (at least 256 bits). Dictionary words, short strings, or anything predictable can be cracked offline since the signature is public.

---

## Lab 4 — JWT Authentication Bypass via JWK Header Injection
**Level:** Practitioner

**What the lab is about:**  
The server uses the `jwk` parameter in the JWT header to specify the public key it should use for verification. If the server trusts this value without checking against a known key store, you can inject your own key pair.

**What I did:**  
Generated a new RSA key pair in Burp's JWT Editor. Modified the JWT header to include the public key as a `jwk` parameter. Signed the modified token (admin payload) with the corresponding private key. The server used the embedded `jwk` to verify — and since I signed it with the matching private key, verification passed.

**Key takeaway:**  
The server should maintain its own trusted key store. Never use a key supplied within the token itself for verification — that defeats the entire purpose of the signature.

---

## Lab 5 — JWT Authentication Bypass via JKU Header Injection
**Level:** Practitioner

**What the lab is about:**  
Similar to JWK injection but the server fetches the public key from a URL specified in the `jku` header. You can host your own JWKS endpoint and make the server fetch your public key.

**What I did:**  
Generated an RSA key pair. Hosted a JWKS JSON file on the exploit server containing my public key. Modified the JWT header:
```json
{"alg":"RS256","jku":"https://exploit-server/my-jwks.json","kid":"my-key-id"}
```
Signed the modified token with my private key. The server fetched the JWKS from my exploit server, found the matching key, and verified the signature successfully.

**Key takeaway:**  
`jku` URLs must be validated against a strict allowlist of trusted domains. Fetching arbitrary URLs from a JWT header is a critical flaw.

---

## Lab 6 — JWT Authentication Bypass via Kid Header Path Traversal
**Level:** Practitioner

**What the lab is about:**  
The `kid` (key ID) header is used to look up the signing key from a file on the server. If it's passed unsanitized to a file path, you can use path traversal to point to a file with known contents.

**What I did:**  
Modified the `kid` value to:
```
../../../dev/null
```
`/dev/null` is an empty file — its "contents" is an empty string. Used `HS256` algorithm and signed the token with an empty string as the secret. The server looked up the key from `/dev/null`, got an empty string, and my signature matched.

**Key takeaway:**  
The `kid` parameter must be treated as untrusted input. Never use it as a filesystem path. Use a lookup table or database to map key IDs to keys.

---
---

# Web Cache Deception — Lab Write-ups

> **Topic:** Web Cache Deception  
> **Platform:** PortSwigger Web Security Academy  
> **Labs Completed:** All  
> **Tools Used:** Burp Suite, caching behavior analysis

---

## What is Web Cache Deception?

Web cache deception tricks a cache into storing a private, user-specific response under a URL that an attacker can then access. It's the opposite of cache poisoning — instead of poisoning what others see, you're manipulating the cache into storing something it shouldn't.

It works by exploiting inconsistencies between how the cache and the origin server parse URLs.

---

## Lab 1 — Web Cache Deception Basic
**Level:** Practitioner

**What the lab is about:**  
The cache stores responses based on file extension. The application ignores path suffixes it doesn't recognize and returns the same authenticated response. A victim visiting a crafted URL like `/account/profile.css` will have their profile page cached — and you can fetch it unauthenticated.

**What I did:**  
As the victim (simulated): accessed `https://TARGET/my-account/test.js`.

The cache saw `.js` and decided to cache it. The origin server ignored `/test.js` and returned the real `/my-account` page with the user's details.

Then fetched the same URL without authentication — got the cached victim profile page including the API key.

**Key takeaway:**  
Cache rules must be defined based on the actual response sensitivity, not just the URL extension. The cache and origin server need consistent URL parsing behavior.

**Mitigation:**  
Use `Cache-Control: no-store` on all authenticated/sensitive responses. Ensure the cache only stores responses that are truly public.

---

## Lab 2 — Web Cache Deception with URL Normalization
**Level:** Practitioner

**What the lab is about:**  
The cache normalizes URLs before caching (e.g., `/account/..%2Fprofile` resolves to `/profile` for cache key purposes), while the origin server processes the raw URL differently. This mismatch allows cache deception.

**What I did:**  
Crafted a URL using path traversal encoded in a way the cache would normalize but the origin would serve differently. Tricked the cache into storing the victim's profile page under a key that I could predict and retrieve unauthenticated.

**Key takeaway:**  
Inconsistent URL normalization between cache and origin is the root cause here. Both layers need to apply the same normalization rules, or the cache needs to use the origin's normalized URL as the cache key.

---

*These topics cover the foundation of what a VAPT analyst needs to know about modern web application vulnerabilities. Each lab builds intuition for how attackers think and how defenses fail.*
