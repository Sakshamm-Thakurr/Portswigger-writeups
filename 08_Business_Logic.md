# Business Logic Vulnerabilities — Lab Write-ups

> **Topic:** Business Logic Vulnerabilities  
> **Platform:** PortSwigger Web Security Academy  
> **Labs Completed:** All  
> **Tools Used:** Burp Suite (Repeater, Intercept), logical reasoning

---

## What are Business Logic Vulnerabilities?

Unlike injection flaws, business logic vulnerabilities aren't about malformed input — they're about using the application exactly as designed, but in ways the developers didn't anticipate. The app works correctly from a technical standpoint; the flaw is in the assumptions baked into the logic.

These are hard to find with scanners. You need to actually understand how the application is supposed to work and then ask: "what happens if I do the opposite of what's expected?" or "what happens if I skip a step?" or "what if I supply a negative value?" It's about breaking assumptions.

---

## Lab 1 — Excessive Trust in Client-Side Controls

**Level:** Apprentice

**What the lab is about:**  
Product prices are sent from the client in the purchase request. The server trusts whatever price it receives. You can just set the price to whatever you want.

**What I did:**  
Added a product to the cart. Intercepted the POST request that added the item. The body included a `price` field. Changed it to `1` (1 cent).

Completed the purchase. The server accepted the client-supplied price.

**Key takeaway:**  
Never trust the client. Prices, quantities, discounts, roles — all should be looked up server-side at the time of the transaction, not passed from the client. The client can always lie.

**Mitigation:**  
Look up all financial values (prices, discounts, totals) from the server-side database at checkout time. Never accept them as input.

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
Validate input ranges on the server, not just the client. Negative quantities, negative prices, zero values — all should be explicitly rejected at the business logic layer. The assumption "users won't submit negative quantities" is exactly the kind of assumption attackers target.

**Mitigation:**  
Enforce minimum quantity constraints (e.g., `quantity >= 1`) server-side. Validate total cart value makes sense before processing payment.

---

## Lab 3 — Low-Level Logic Flaw (Integer Overflow)

**Level:** Practitioner

**What the lab is about:**  
The cart total is stored as a 32-bit integer. Adding enough items causes it to overflow and wrap around to a large negative number. Once negative, you've effectively got infinite money.

**What I did:**  
Used Burp Intruder to repeat adding the maximum quantity (99) of a high-priced item hundreds of times. After enough iterations, the total wrapped to a negative number due to integer overflow. Adjusted with cheap items to get the total between $0 and my available funds, then checked out.

**Key takeaway:**  
Integer overflow in financial calculations is a real vulnerability class. Currency and pricing fields should use appropriate data types (like `BigDecimal` in Java or `DECIMAL` in SQL) and have reasonable maximum limits enforced at the application layer, not just the data type level.

**Mitigation:**  
Use arbitrary precision data types for financial values. Implement sanity checks: cart total must always be positive and within reasonable bounds.

---

## Lab 4 — Inconsistent Security Controls

**Level:** Apprentice

**What the lab is about:**  
The admin panel requires a company email address (`@dontwannacry.com`). But the app lets you change your email address after registration. It validates the email during registration but not during the change.

**What I did:**  
Registered with any email. Then changed my email to `attacker@dontwannacry.com` in the account settings. The change was accepted without verification. Now had access to the admin panel.

**Key takeaway:**  
Security controls applied at registration need to be consistently enforced throughout the account lifecycle. Privilege checks based on email domain need to be re-validated whenever the email changes — and email changes themselves should require verification of the new address.

**Mitigation:**  
Re-evaluate role/privilege assignments whenever account details change. Require email verification for all email changes. Don't grant privileges based on unverified email addresses.

---

## Lab 5 — Flawed Enforcement of Business Rules

**Level:** Apprentice

**What the lab is about:**  
The app has two discount codes — a sign-up code and a newsletter code. Each can only be used once. But you can alternate between them indefinitely.

**What I did:**  
Applied code 1 → applied code 2 → applied code 1 again → applied code 2 again. The "already used" check only prevented using the same code twice consecutively, not alternating between two codes. Applied them back and forth until the item was free.

**Key takeaway:**  
Business rule enforcement needs to think about sequences and combinations, not just individual actions. Discount code systems need to track all applied codes across a session/order, not just the most recently applied one.

**Mitigation:**  
Track all applied discount codes at the order level. Enforce "used" status globally per order, not just per consecutive application.

---

## Lab 6 — Infinite Money Logic Flaw

**Level:** Practitioner

**What the lab is about:**  
A gift card system combined with a store credit mechanism creates an infinite money loop: buy gift card → redeem for store credit → buy another gift card → repeat.

**What I did:**  
Bought a £10 gift card using a 30% discount code → paid £7. Redeemed the gift card for £10 store credit. Net gain: £3 per cycle. Automated this with a Burp macro to repeat the loop until I had enough credit to buy the target item.

**Key takeaway:**  
Complex multi-feature interactions create emergent vulnerabilities. Each individual feature may be secure, but their combination creates a flaw. You need to test not just features in isolation but their interactions — especially when gift cards, credit, and discounts are all present.

**Mitigation:**  
Restrict discount code use to specific product categories (e.g., not applicable to gift cards). Add rate limiting on gift card purchases and redemptions. Design gift card flows to not generate net profit when combined with discounts.

---

## Lab 7 — Authentication Bypass via Flawed State Machine

**Level:** Practitioner

**What the lab is about:**  
During the login flow, a role selection step sets your user role. If you drop the role-selection request (so it never completes), the application falls back to a default admin role.

**What I did:**  
Logged in and intercepted the POST request for the role selection page. Dropped the request entirely in Burp. Navigated directly to `/admin`. Because the role selection never completed, the application defaulted to a privileged role instead of the least-privileged user role.

**Key takeaway:**  
Multi-step flows need to validate state at every step. "What's the default state if a step is skipped?" is an important design question. The answer should always be: least privilege. Defaulting to admin when a setup step is incomplete is a catastrophic assumption failure.

**Mitigation:**  
Default to the most restrictive role when setup is incomplete. Re-verify role/state on every subsequent request, not just during the setup flow. Never assume the flow completed just because the user arrived at the next step.

---

*Business logic vulnerabilities are my favourite to hunt in real engagements because they're completely invisible to automated scanners. They require you to actually read the application, understand its intent, and think adversarially about what happens when you break the assumptions. Every app has different logic — and therefore different logic flaws.*
