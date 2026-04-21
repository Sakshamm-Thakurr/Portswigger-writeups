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
`phpinfo()`, debug endpoints, and `/actuator` endpoints (Spring Boot) are commonly forgotten. Always check these during recon. These pages can expose database credentials, API keys, and internal network topology all in one shot.

**Mitigation:**  
Remove or disable debug pages before deployment. Restrict access to diagnostic endpoints by IP at the network level.

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

**Mitigation:**  
Deployment pipelines should strip all backup and temporary files. Implement a web server deny rule for common backup extensions (`.bak`, `.swp`, `.orig`, `~`).

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
The `TRACE` method should almost always be disabled. It reflects request headers back and can expose internal proxy headers, session tokens, and other sensitive data. Many security scanners check for this automatically — it's a quick win.

**Mitigation:**  
Disable the `TRACE` HTTP method on all web servers. Never rely on client-supplied IP headers for access control.

---

## Lab 5 — Information Disclosure in Version Control History

**Level:** Practitioner

**What the lab is about:**  
The `.git` directory was accidentally deployed to production. You can download the entire Git history and recover deleted files — like a config file that had a password in it before being "removed" in a later commit.

**What I did:**  
Confirmed `.git` was accessible by visiting `/.git/HEAD`. Used `wget --mirror` (or Burp to manually fetch files) to download the git object store. Reconstructed the repo locally using `git checkout` and browsed the commit history:

```bash
git log --oneline
git show <commit-hash>
```

Found a commit that deleted `admin.conf` — diffed it and recovered the admin password from the deleted file.

**Key takeaway:**  
`.git`, `.svn`, `.env`, `.DS_Store` — all of these exposed in web root are serious. Deployment pipelines should explicitly exclude them. One of the most common findings in real bug bounties because developers think deleting from git history removes the data — it doesn't without a rewrite.

**Mitigation:**  
Add `.git/` and `.env` to `.gitignore` and web server deny rules. Use `git-secrets` or similar tools to prevent committing secrets in the first place.

---

*Information disclosure bugs are often underrated in severity but overrate in frequency. In a real engagement, they're invaluable — every piece of internal info narrows the attack surface and can be the key that unlocks a deeper chain.*
