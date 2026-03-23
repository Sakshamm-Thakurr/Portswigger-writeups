# Authentication Vulnerabilities — Lab Write-ups

> **Topic:** Authentication  
> **Platform:** PortSwigger Web Security Academy  
> **Labs Completed:** All  
> **Tools Used:** Burp Suite (Intruder, Repeater), Python scripts

---

## What are Authentication Vulnerabilities?

Authentication is literally the gatekeeper — it's how an application decides who you are. When it's broken, attackers can log in as other users, bypass 2FA, take over accounts, and escalate to admin. Flaws here tend to be high impact because they directly affect every user on the platform.

Common issues: username enumeration, weak brute-force protection, flawed 2FA logic, insecure password reset flows, and predictable tokens.

---

## Lab 1 — Username Enumeration via Different Responses
**Level:** Apprentice

**What the lab is about:**  
The login form gives a different error message for invalid username vs invalid password. This leaks whether a username exists — and you can build a valid username list from it.

**What I did:**  
Used Burp Intruder with the username list from PortSwigger's lab resources. When the response said "Incorrect password" (instead of "Invalid username"), I knew that username was valid. Then brute-forced the password for that username using the provided password list.

**Key takeaway:**  
Error messages matter. "Invalid username" vs "Incorrect password" is a classic enumeration leak. The response should always say the same thing regardless of which field is wrong.

**Mitigation:**  
Use a generic message like "Invalid username or password" for all login failures.

---

## Lab 2 — Username Enumeration via Subtly Different Responses
**Level:** Apprentice

**What the lab is about:**  
Same concept but the error message difference is very subtle — like a trailing space or a single character variation. Automated comparison is necessary.

**What I did:**  
Ran Intruder on the username field and added a "grep match" for the exact error string. Noticed one response was missing a period at the end of the error message, or had slightly different wording. That was the valid username.

**Key takeaway:**  
Developers sometimes try to be clever about consistent error messages but introduce tiny differences. Always grep response bodies and lengths together — a difference of even 1 byte is worth investigating.

---

## Lab 3 — Username Enumeration via Response Timing
**Level:** Practitioner

**What the lab is about:**  
The app returns the same error message regardless of username/password correctness — but internally, it only checks the password if the username exists. Valid usernames cause a slightly longer response time because the password hash comparison actually runs.

**What I did:**  
Added a very long password to maximize the timing difference. Used Burp Intruder in "resource pool" mode with a single thread to measure response times accurately. The valid username had a consistently longer response time than invalid ones.

Also needed to bypass IP-based rate limiting by adding an `X-Forwarded-For` header with incrementing IPs for each request.

**Key takeaway:**  
Timing side channels are subtle but real. Hash computation time leaks whether a user exists. Even milliseconds of difference can be reliable enough to enumerate.

---

## Lab 4 — Broken Brute-Force Protection (IP Block)
**Level:** Practitioner

**What the lab is about:**  
The application blocks an IP after 3 failed login attempts. But it resets the counter if you log in successfully. The trick: intersperse your brute-force with successful logins using your own credentials.

**What I did:**  
Built an Intruder attack with two payload positions (username and password) using the "Pitchfork" attack type. The payload list alternated: `wiener:peter` (successful login to reset counter), then the target username with one candidate password, then `wiener:peter` again, and so on.

Every third attempt was a successful login to reset the block counter. Bypassed the protection completely.

**Key takeaway:**  
Rate limiting based on failed-attempt counters that reset on success is easily bypassable. Real protection needs to account for total login attempts from an IP, not just consecutive failures.

---

## Lab 5 — Username Enumeration via Account Lock
**Level:** Practitioner

**What the lab is about:**  
Accounts lock after too many failed attempts for real usernames — but non-existent usernames never lock. You can enumerate valid usernames by seeing which ones eventually return a "locked" error.

**What I did:**  
Sent 5+ login attempts for each candidate username. Real usernames eventually returned "Your account is locked" — fake ones just kept saying "Invalid username or password." Built the valid username list this way, then waited for the lockout to expire (or found a bypass) to brute-force passwords.

**Key takeaway:**  
Account lockout policies can be turned against the application. The "account is locked" message is itself a confirmation the username exists.

---

## Lab 6 — Broken Brute-Force Protection (Multiple Credentials per Request)
**Level:** Expert

**What the lab is about:**  
The application uses JSON in the login request and only checks the first value if a username is provided as an array. You can send multiple passwords in one request — each one tested against the username.

**What I did:**  
Modified the login request body from:
```json
{"username":"carlos","password":"montoya"}
```
To:
```json
{"username":"carlos","password":["password1","password2","password3",...all 100 passwords...]}
```

The app looped through all passwords in the array in a single request, bypassing per-request rate limiting entirely.

**Key takeaway:**  
JSON arrays as input values are an often-forgotten injection vector. One request, 100 password attempts. Classic logic flaw in how APIs handle multi-value parameters.

---

## Lab 7 — 2FA Simple Bypass
**Level:** Apprentice

**What the lab is about:**  
After entering valid credentials, you're asked for a 2FA code. But the 2FA check is a separate page — and if you navigate directly to `/my-account` after login (skipping the 2FA page), the application doesn't enforce the check.

**What I did:**  
Logged in with `carlos:montoya`. Was redirected to `/login2` for the 2FA code. Instead of entering the code, manually navigated to `/my-account`. The application had already set the session as authenticated after the password step — the 2FA was just a UI redirect, not a real access control check.

**Key takeaway:**  
2FA must be enforced server-side. If the session gets a "logged in" flag before the 2FA step is completed, the second factor is useless. The check needs to happen on every restricted page, not just on the 2FA submission endpoint.

---

## Lab 8 — 2FA Broken Logic
**Level:** Practitioner

**What the lab is about:**  
The 2FA code is tied to the username, not the session. After logging in as yourself, you can change the username cookie to `carlos` and request a new 2FA code — which goes to carlos's email, but the code gets verified against whatever username is in the cookie.

**What I did:**  
Logged in as `wiener`. Noticed the POST request to complete 2FA included a `verify=wiener` cookie. Changed it to `verify=carlos` and requested a new 2FA code. The code was sent to carlos's email, but since I control the verification step, I just brute-forced the 4-digit code (10,000 possibilities) using Intruder — no lockout on this endpoint.

**Key takeaway:**  
2FA codes must be bound to an authenticated session, not a user-supplied value. Anything the client sends can be tampered with.

---

## Lab 9 — 2FA Bypass Using a Brute-Force Attack
**Level:** Expert

**What the lab is about:**  
Valid credentials are known, but the 2FA code (4 digits) needs to be brute-forced. The application logs you out after 2 wrong attempts — but you can script the full login + 2FA attempt flow.

**What I did:**  
Used a Burp macro to automate:
1. GET `/login` (grab CSRF token)
2. POST `/login` with credentials (logs in)
3. POST `/login2` with the 2FA candidate

Used Session Handling Rules to run the macro before each Intruder payload. The macro resets the session so every attempt starts fresh — bypassing the 2-attempt logout.

Brute-forced all 10,000 4-digit codes. Found the valid one.

**Key takeaway:**  
Short numeric 2FA codes are brute-forceable if there's no lockout. Either use longer codes, implement strict lockouts, or use TOTP with time-based invalidation.

---

## Lab 10 — Password Reset Broken Logic
**Level:** Apprentice

**What the lab is about:**  
The password reset flow uses a token sent via email. But the server doesn't actually verify that the token matches the username in the reset form — you can reset any account's password.

**What I did:**  
Requested a reset for my own account. Got the token from the email. Submitted the reset form, but changed the username parameter from `wiener` to `carlos`. The application accepted it and changed carlos's password.

**Key takeaway:**  
Reset tokens must be cryptographically bound to the specific user and single-use. Never trust client-supplied username values during a reset flow.

---

## Lab 11 — Password Reset via Email Header Injection
**Level:** Practitioner

**What the lab is about:**  
The "forgot password" form sends a reset email. The host/domain is taken from the HTTP `Host` header and embedded in the reset link. Injecting a Collaborator URL into the Host header redirects the reset link to your server.

**What I did:**  
Modified the `Host` header in the password reset request:
```
Host: TARGET.com
Host: TARGET.com:BURP-COLLABORATOR-SUBDOMAIN
```

(Used the second Host header injection trick, or modified to `TARGET.com:exploit-server`.)

The reset email was sent to carlos with a link pointing to my Collaborator host. Captured the reset token from the DNS/HTTP log and used it to reset carlos's password.

**Key takeaway:**  
Never build password reset URLs using the `Host` header. Use a hardcoded domain from server-side configuration.

---

## Lab 12 — Password Brute-Force via Password Change
**Level:** Practitioner

**What the lab is about:**  
The password change form is vulnerable to brute-force through an indirect path. If the current password is wrong, the error message differs depending on whether the new passwords match. This can be used to enumerate the current password.

**What I did:**  
Set up an Intruder attack on the `current-password` field. Set `new-password-1` and `new-password-2` to different values. When the current password is wrong and new passwords don't match, the error is "New passwords do not match." When the current password is correct (but new passwords still don't match), the error changes to "Current password is incorrect." Wait — actually it's reversed. When the correct current password is submitted, a different code path runs.

Enumerated each password from the list against carlos's account with no lockout — found the matching response body that indicated success.

**Key takeaway:**  
Business logic in account management features often opens side channels. Password change flows need the same protection as login forms.

---

*Authentication vulnerabilities are always high-severity because they directly lead to account takeover. Understanding these patterns — enumeration, brute-force, 2FA flaws, reset logic bugs — is core to any VAPT engagement.*
