# PortSwigger Web Security Academy — Lab Write-ups

**Author:** Saksham Thakur  
**Profile:** M.Tech Cybersecurity | VAPT & Web Application Security  
**Platform:** [PortSwigger Web Security Academy](https://portswigger.net/web-security)

---

## About This Repository

This repo contains my personal write-ups for PortSwigger Web Security Academy labs. I've been working through these systematically as part of building practical hands-on skills in web application penetration testing and vulnerability assessment.

Each write-up covers:
- What the vulnerability is and why it matters
- What I did step-by-step to exploit it
- Key takeaways from a tester's perspective
- Mitigation / how to fix it

These aren't copy-pasted solutions — they're my actual notes from working through each lab, written in a way that helps me (and hopefully others) understand the *why* behind each technique, not just the *what*.

> **Disclaimer:** All labs are deliberately vulnerable environments provided by PortSwigger for educational purposes. Nothing here is intended for use against real systems without authorization.

---

## Labs Completed

| # | Topic | Labs Done | Level |
|---|-------|-----------|-------|
| 01 | [SQL Injection](./01_SQL_Injection.md) | All (16 labs) | Apprentice → Expert |
| 02 | [Cross-Site Scripting (XSS)](./02_XSS.md) | Apprentice + Practitioner | Apprentice → Practitioner |
| 03 | [XXE Injection](./03_XXE_Injection.md) | All (8 labs) | Apprentice → Practitioner |
| 04 | [OS Command Injection](./04_OS_Command_Injection.md) | All (5 labs) | Apprentice → Practitioner |
| 05 | [Path Traversal](./05_Path_Traversal.md) | All (6 labs) | Apprentice → Practitioner |
| 06 | [Authentication](./06_Authentication.md) | All (12 labs) | Apprentice → Expert |
| 07 | [Information Disclosure](./07_08_09_10_InfoDisc_BizLogic_FileUpload_JWT_CacheDeception.md#information-disclosure--lab-write-ups) | All (5 labs) | Apprentice → Practitioner |
| 08 | [Business Logic Vulnerabilities](./07_08_09_10_InfoDisc_BizLogic_FileUpload_JWT_CacheDeception.md#business-logic-vulnerabilities--lab-write-ups) | All (7 labs) | Apprentice → Practitioner |
| 09 | [File Upload Vulnerabilities](./07_08_09_10_InfoDisc_BizLogic_FileUpload_JWT_CacheDeception.md#file-upload-vulnerabilities--lab-write-ups) | All (6 labs) | Apprentice → Practitioner |
| 10 | [JWT Vulnerabilities](./07_08_09_10_InfoDisc_BizLogic_FileUpload_JWT_CacheDeception.md#jwt-vulnerabilities--lab-write-ups) | All (6 labs) | Apprentice → Practitioner |
| 11 | [Web Cache Deception](./07_08_09_10_InfoDisc_BizLogic_FileUpload_JWT_CacheDeception.md#web-cache-deception--lab-write-ups) | All | Practitioner |

**Total labs documented: 80+**

---

## Tools Used Across These Labs

- **Burp Suite Community Edition** — Proxy, Repeater, Intruder, Decoder, JWT Editor extension
- **Kali Linux** — Primary testing OS
- **Python** — Custom scripts for automation (brute-force, payload generation)
- **hashcat** — JWT secret cracking
- **exiftool** — Polyglot file creation for file upload labs
- **Burp Collaborator** — Out-of-band interaction detection

---

## Topics I'm Currently Working On

- Access Control Vulnerabilities
- CSRF
- Server-Side Request Forgery (SSRF)
- HTTP Request Smuggling
- Insecure Deserialization

---

## Connect With Me

- LinkedIn: [linkedin.com/in/saksham-thakur-244450268](https://www.linkedin.com/in/saksham-thakur-244450268/)
- Email: sakshamthakur2804@gmail.com

---

*Working through these labs has genuinely changed how I look at web applications. Every parameter is a potential injection point. Every feature is a potential logic flaw. Highly recommend this platform to anyone getting into web security.*
