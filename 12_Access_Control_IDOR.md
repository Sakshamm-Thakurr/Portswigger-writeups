# Access Control Vulnerabilities (IDOR) — Lab Write-ups

> **Topic:** Access Control & Insecure Direct Object References (IDOR)  
> **Platform:** PortSwigger Web Security Academy  
> **Labs Completed:** All (Apprentice + Practitioner + Expert)  
> **Tools Used:** Burp Suite (Proxy, Repeater, Intruder), Browser DevTools

---

## What are Access Control Vulnerabilities?

Access control is the mechanism that enforces what authenticated users are allowed to do. When it breaks, users can access resources or perform actions they're not supposed to. IDOR (Insecure Direct Object Reference) is the most common flavour — it happens when an application uses user-controlled input (like an ID or filename) to directly reference objects like database records, files, or user accounts without verifying that the requesting user is authorised to access them.

The impact can be severe: reading other users' private data, modifying or deleting their accounts, escalating to admin, or bypassing entire payment/logic flows. It's one of the most consistently found bugs in real-world VAPT engagements because developers often implement authentication but forget authorization entirely.

---

## Lab 1 — Unprotected Admin Functionality

**Level:** Apprentice

**What the lab is about:**  
An admin panel exists at a predictable URL but is only hidden — not actually protected. Anyone who knows the URL can access it.

**What I did:**  
Checked `robots.txt` first. Found a disallowed path: `/administrator-panel`. Navigated directly to it in the browser. Admin panel loaded without any authentication check. Used it to delete the target user.

**Key takeaway:**  
Security through obscurity is not access control. Hiding a URL in `robots.txt` actually advertises it. Every privileged endpoint must enforce authorization server-side regardless of how "secret" the URL is.

**Mitigation:**  
Enforce role-based access checks on every admin endpoint server-side. Never rely on the URL being unknown.

---

## Lab 2 — Unprotected Admin Functionality with Unpredictable URL

**Level:** Apprentice

**What the lab is about:**  
The admin panel URL isn't in `robots.txt` this time — it's dynamically generated and embedded in the JavaScript of the homepage. Still no actual access control enforced.

**What I did:**  
Opened the browser DevTools → Sources tab. Searched through the homepage JavaScript. Found a script that conditionally revealed the admin panel link — something like `if (user.isAdmin) adminLink = '/admin-h4rdtogue55'`. Navigated directly to that URL without being an admin. Full access granted.

**Key takeaway:**  
Client-side code is always readable. Hiding URLs or logic in JavaScript is useless because the browser downloads and executes it. Server-side enforcement is the only thing that matters.

**Mitigation:**  
Server must check admin role before rendering or responding to any privileged route.

---

## Lab 3 — User Role Controlled by Request Parameter

**Level:** Apprentic

**What the lab is about:**  
The application passes an `Admin` parameter (usually a cookie or query parameter) to determine user role. There's no server-side session validation — you can just set it yourself.

**What I did:**  
Logged in with regular credentials. Intercepted the request in Burp. Found a cookie like `Admin=false`. Changed it to `Admin=true`. Forwarded the request. Application now treated me as an admin and showed the admin panel. Deleted the target user.

**Key takeaway:**  
Never trust client-supplied role or privilege indicators. Session-based role assignment stored server-side is the correct approach. Cookies, headers, and URL parameters are all attacker-controlled.

**Mitigation:**  
Store user roles in the server-side session, not in client-controlled values.

---

## Lab 4 — User Role Can Be Modified in User Profile

**Level:** Apprentice

**What the lab is about:**  
The application exposes a role field in the JSON response when updating a user profile — and the server accepts it as writable input.

**What I did:**  
Went to account settings and updated my email. Captured the POST request in Burp Repeater. The JSON body looked like:

```json
{"email":"attacker@test.com"}
```

Added the `roleid` field:

```json
{"email":"attacker@test.com","roleid":2}
```

Forwarded it. The response confirmed `roleid:2` was accepted. Navigated to `/admin` — full admin access.

**Key takeaway:**  
Mass assignment / parameter pollution. If your API blindly maps all incoming JSON fields to the database model without a whitelist, attackers can write to fields they should never touch.

**Mitigation:**  
Explicitly whitelist which fields are user-modifiable. Never accept sensitive fields like `roleid`, `isAdmin`, or `balance` from user input.

---

## Lab 5 — URL-Based Access Control Can Be Circumvented

**Level:** Practitioner

**What the lab is about:**  
The front-end blocks access to `/admin` based on the URL. But there's a custom header (`X-Original-URL`) that the back-end uses to determine the actual path — and the front-end doesn't check it.

**What I did:**  
Direct GET to `/admin` returned 403. Modified the request:

```
GET / HTTP/1.1
X-Original-URL: /admin
```

The front-end saw `/` (allowed) but the back-end routed to `/admin`. Got the admin panel. Then used:

```
GET /?username=carlos
X-Original-URL: /admin/delete
```

to delete the target user.

**Key takeaway:**  
Header-based URL overrides like `X-Original-URL` and `X-Rewrite-URL` are supported by some frameworks (e.g., Symfony) and can completely bypass front-end access controls. Always strip or validate these headers at the perimeter.

**Mitigation:**  
Strip non-standard URL override headers at the load balancer/WAF level. Don't rely on URL-path filtering alone for access control.

---

## Lab 6 — Method-Based Access Control Can Be Circumvented

**Level:** Practitioner

**What the lab is about:**  
Admin actions are restricted based on the HTTP method — POST is blocked for normal users. But POSTGET (or using a different method like `POSTPUT`) isn't checked consistently.

**What I did:**  
Logged in as admin, captured the POST request to `/admin-roles` that upgrades a user. Then logged in as a normal user (wiener). Replayed the admin request but changed the method from POST to GET and moved parameters to the query string:

```
GET /admin-roles?username=wiener&action=upgrade
```

The server only restricted POST for non-admins but didn't check GET for the same endpoint. Request went through — my account was upgraded to admin.

**Key takeaway:**  
Access control must be enforced for all HTTP methods on sensitive endpoints. A common mistake is protecting POST but forgetting GET, PUT, or PATCH on the same route.

**Mitigation:**  
Apply authorization checks at the route level, not just the method level. All methods on privileged endpoints must require appropriate roles.

---

## Lab 7 — User ID Controlled by Request Parameter (IDOR)

**Level:** Apprentice

**What the lab is about:**  
Classic IDOR. The user's account page is loaded using a user ID in the URL parameter. No check that the logged-in user matches the requested ID.

**What I did:**  
Logged in as `wiener`. Navigated to `/my-account?id=wiener`. Changed the `id` parameter to `carlos`. Carlos's account page loaded — including their API key. Submitted the key to solve the lab.

**Key takeaway:**  
This is the textbook IDOR. The parameter is fully attacker-controlled and the server doesn't validate ownership. Every object access needs a server-side ownership check.

**Mitigation:**  
Derive the user ID from the server-side session, not from a URL parameter. Never trust client-supplied identifiers for ownership verification.

---

## Lab 8 — User ID Controlled by Request Parameter with Unpredictable User IDs

**Level:** Apprentice

**What the lab is about:**  
User IDs are GUIDs (non-predictable), so you can't just guess them. But they're disclosed elsewhere in the application — in this case, on blog posts by the target user.

**What I did:**  
Browsed the blog and found a post authored by `carlos`. Inspected the post — the author link contained carlos's GUID. Used that GUID in:

```
/my-account?id=<carlos-guid>
```

Got carlos's account page and API key.

**Key takeaway:**  
Unpredictable IDs are not a substitute for access control. If the GUID is exposed anywhere in the app (blog posts, comments, profile pages), it's still exploitable. Security through obscurity fails again.

**Mitigation:**  
Access control checks must happen regardless of how "random" the ID is.

---

## Lab 9 — User ID Controlled by Request Parameter with Data Leakage in Redirect

**Level:** Apprentice

**What the lab is about:**  
Accessing another user's account page redirects you back — but the redirect response body still contains the target user's data before the redirect executes.

**What I did:**  
Changed the `id` parameter to `carlos`. The server returned a 302 redirect, but the response body contained carlos's account details and API key. In Burp Repeater, I captured the full response before the browser followed the redirect. API key was right there in the body.

**Key takeaway:**  
Redirect responses (302) still have bodies. Browsers may not render them, but tools like Burp do. Never assume a redirect means the data wasn't sent.

**Mitigation:**  
Perform the authorization check before generating the response body. Don't populate the page then redirect — redirect first.

---

## Lab 10 — IDOR with Direct Reference to Database Objects

**Level:** Practitioner

**What the lab is about:**  
A chat history feature loads conversation records using a numeric message ID. No ownership check — any authenticated user can access any message.

**What I did:**  
Opened my chat history. Noticed the download request:

```
GET /download-transcript/2.txt
```

Changed `2.txt` to `1.txt`. Downloaded another user's chat transcript. The password for the target user was disclosed in their chat history. Used it to log in.

**Key takeaway:**  
Sequential numeric IDs in file references are trivially enumerable. One IDOR here gave a full account takeover because sensitive data was in the file.

**Mitigation:**  
Map user-accessible resources to their account server-side. Never allow direct file or database object references without an ownership check.

---

## Lab 11 — IDOR with Direct Reference to Static Files

**Level:** Practitioner

**What the lab is about:**  
User chat transcripts are stored as static files in a predictable directory. The application serves them without any authentication or authorization check.

**What I did:**  
Observed the transcript download URL pattern. Files are stored at:

```
/static/76.txt
```

Enumerated nearby IDs. Found a transcript containing another user's credentials. Logged in as that user.

**Key takeaway:**  
Static files need access control too. Serving sensitive files directly from a web root without authentication is a fundamental mistake. The file server doesn't know anything about application-level sessions.

**Mitigation:**  
Serve sensitive files through an authenticated application endpoint that checks ownership before streaming the file. Never store private data in a publicly accessible web directory.

---

## Lab 12 — Multi-Step Process with No Access Control on One Step

**Level:** Practitioner

**What the lab is about:**  
An admin action (promoting a user) uses a multi-step flow. The first step is protected. The second step (the actual action) is not — it assumes anyone who reaches it came through step one legitimately.

**What I did:**  
As admin, walked through the user promotion flow and captured both POST requests in Burp. Logged in as a normal user. Replayed only the second POST (the confirmation step) directly — skipping the first step entirely. The server executed the promotion without checking if I was an admin.

**Key takeaway:**  
Access control must be enforced on every step of a multi-step flow independently. Never assume that reaching step 2 implies step 1 was legitimately completed by an authorized user.

**Mitigation:**  
Re-check authorization at every step. Don't use "which step are you on" as a proxy for "are you authorized."

---

## Lab 13 — Referer-Based Access Control

**Level:** Practitioner

**What the lab is about:**  
The admin endpoint checks the `Referer` header to confirm the request came from the admin panel page. If the Referer matches, the request is allowed — regardless of who the user is.

**What I did:**  
As admin, captured the GET request to upgrade a user. Noted the `Referer: https://TARGET/admin` header. Logged in as a normal user. Replayed the upgrade request with the forged Referer header:

```
Referer: https://TARGET/admin
```

Server accepted it. My account was upgraded to admin.

**Key takeaway:**  
The `Referer` header is completely attacker-controlled and can be spoofed. It is not an access control mechanism. Never use it for authorization decisions.

**Mitigation:**  
Enforce authorization based on server-side session roles, not on any client-supplied headers.

---

*Access control vulnerabilities are consistently one of the most impactful bug classes. IDOR in particular is easy to miss in code review because the code looks right — the issue is what's missing, not what's there. Every time you see a user-controlled identifier, ask: does the server verify ownership?*
