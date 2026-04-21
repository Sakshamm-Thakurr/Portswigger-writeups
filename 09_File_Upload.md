# File Upload Vulnerabilities — Lab Write-ups

**Topic:** File Upload Vulnerabilities  
**Platform:** PortSwigger Web Security Academy  
**Labs Completed:** All  
**Tools Used:** Burp Suite (Repeater, Intercept), PHP webshells, exiftool

---

## What are File Upload Vulnerabilities?

File upload features are high-risk because if you can get the server to execute a file you uploaded, you have remote code execution. Even if execution isn't possible, you might achieve stored XSS, path traversal, or denial of service through oversized files.

The attack surface is broader than people think: avatar uploads, document imports, profile pictures, backup restore features, report generators, and any "attach a file" functionality. The core questions as a tester are: will the server execute this? Can I control where it's stored? Can I access it afterward?

---

## Lab 1 — Remote Code Execution via Web Shell Upload

**Level:** Apprentice

**What the lab is about:**  
The avatar upload feature has no file type validation whatsoever. You can upload a PHP file and it will be served and executed by the server.

**What I did:**  
Created a simple PHP webshell:

\<?php echo system($\_GET\['cmd'\]); ?\>

Uploaded it as `webshell.php`. The server accepted it and stored it in `/files/avatars/`. Fetched:

GET /files/avatars/webshell.php?cmd=cat+/home/carlos/secret

The PHP executed server-side. Got the secret file contents back in the response body.

**Key takeaway:**  
Upload functionality without any content-type validation is critical severity — it's a direct, trivial path to RCE. This is the baseline worst case. Never allow arbitrary file uploads without thorough validation.

**Mitigation:**  
Validate file type server-side based on actual content (magic bytes), not just extension or MIME type. Serve uploaded files from a separate domain or storage bucket where script execution is impossible. Never execute uploaded files.

---

## Lab 2 — Web Shell Upload via Content-Type Restriction Bypass

**Level:** Apprentice

**What the lab is about:**  
The application checks the `Content-Type` header to validate file type — but doesn't inspect the actual file content. You can upload a PHP file by spoofing the content type to `image/jpeg`.

**What I did:**  
Uploaded `webshell.php` and captured the request in Burp. Changed the `Content-Type` header:

Content-Type: image/jpeg

The server checked the header — it said JPEG — and accepted the PHP file. Executed the webshell the same way as Lab 1\.

**Key takeaway:**  
The `Content-Type` header is completely attacker-controlled. It's sent by the browser (or Burp), not determined by the file's actual content. Server-side validation must inspect the actual file bytes, not just the MIME type header.

**Mitigation:**  
Inspect actual file content using magic byte / file signature validation. Libraries like `python-magic` or similar can detect the true file type. Reject anything that doesn't match an explicit whitelist of allowed types.

---

## Lab 3 — Web Shell Upload via Path Traversal

**Level:** Practitioner

**What the lab is about:**  
The upload directory has a server configuration preventing PHP execution. But by using path traversal in the filename, you can write the file to a parent directory where execution is allowed.

**What I did:**  
Changed the filename in the multipart upload request to use a traversal sequence:

filename="../webshell.php"

URL-encoded version:

filename="..%2Fwebshell.php"

The file was written to the parent directory — one level above the intended uploads folder — where PHP execution wasn't restricted. Accessed it at the correct path and got RCE.

**Key takeaway:**  
Filename sanitization must strip or reject path traversal sequences (`../`, `..%2F`, `..%5C`) before saving. Never pass user-supplied filenames directly to filesystem operations without normalization.

**Mitigation:**  
Normalize and sanitize filenames before saving. Generate a random server-side filename for uploads rather than using user-supplied names at all. Apply execution restrictions to the entire uploads area, not just the target directory.

---

## Lab 4 — Web Shell Upload via Extension Blacklist Bypass

**Level:** Practitioner

**What the lab is about:**  
Common PHP extensions (`.php`, `.php3`, `.php4`, `.php5`, `.phtml`) are blacklisted. But you can upload a `.htaccess` file to redefine what file extensions get executed as PHP.

**What I did:**  
First uploaded a `.htaccess` file with the following content:

AddType application/x-httpd-php .l33t

The server accepted `.htaccess` since it wasn't blacklisted. Then uploaded `webshell.l33t` — which the server now interpreted as PHP thanks to the `.htaccess` override. Got RCE.

**Key takeaway:**  
Extension blacklists are fundamentally flawed — there are always obscure extensions, less-known PHP aliases, and server configuration overrides. Allowing `.htaccess` uploads is especially dangerous. Use whitelists only.

**Mitigation:**  
Whitelist allowed extensions — only accept `.jpg`, `.png`, etc. Block `.htaccess` and any other server configuration files explicitly. Use `Options -ExecCGI` and `php_flag engine off` in upload directory configs and don't allow them to be overridden.

---

## Lab 5 — Web Shell Upload via Obfuscated File Extension

**Level:** Practitioner

**What the lab is about:**  
The blacklist checks the extension but can be confused by various obfuscation tricks. The goal is to slip `.php` past the filter in disguise.

**What I did:**  
Tried multiple bypass techniques:

- `webshell.PHP` — uppercase bypass on case-sensitive checks  
- `webshell.php.jpg` — double extension (some servers execute based on first match)  
- `webshell.php%00.jpg` — null byte truncation (older systems strip at `\x00`)  
- `webshell.pHp` — mixed case  
- `webshell.php5` — alternate PHP extension

In this lab, the null byte approach worked:

filename="webshell.php%00.jpg"

The blacklist saw `.jpg` (after the null). The filesystem saved it as `webshell.php` (truncated at null byte). Got RCE.

**Key takeaway:**  
Extension validation needs to be robust against encoding tricks. Always URL-decode and null-strip the filename before extracting the extension. Validate after full normalization.

**Mitigation:**  
Extract the true extension after decoding, normalizing, and null-stripping the filename. Use a whitelist. Generate your own filenames server-side to avoid relying on user input entirely.

---

## Lab 6 — Remote Code Execution via Polyglot Web Shell Upload

**Level:** Practitioner

**What the lab is about:**  
The server validates actual file content (magic bytes) to verify it's a real image — not just the extension or MIME type. A polyglot file is simultaneously a valid JPEG image and contains executable PHP code. It passes the content check and gets executed.

**What I did:**  
Used `exiftool` to inject PHP code into the metadata of a legitimate JPEG image:

exiftool \-Comment="\<?php echo system(\\$\_GET\['cmd'\]); ?\>" real\_image.jpg \-o webshell.php

The output file starts with the correct JPEG magic bytes (`FF D8 FF`) so it passes magic byte validation. When the PHP interpreter processes the file, it finds the `<?php` tag inside the EXIF Comment field and executes it.

Uploaded as `webshell.php` — passed the content check, executed as PHP.

**Key takeaway:**  
Polyglot files defeat magic byte checks alone. A file can be simultaneously a valid image and valid PHP. Defense-in-depth is essential: image re-processing (resize/strip metadata using ImageMagick or similar) removes embedded code, serving from a separate non-executing domain prevents execution even if the file is stored, and CDNs that process images strip code as a side effect.

**Mitigation:**  
Re-process uploaded images server-side (resize, recompress) using a library that strips metadata. Serve uploads from a separate domain or bucket where script execution is impossible regardless of file contents. Never execute user-uploaded files.

---

*File upload vulnerabilities are consistently one of the most impactful findings in web application assessments. The path from "accepts arbitrary uploads" to "full server compromise" is often just a few steps. These labs cover the full spectrum from the trivially obvious to the genuinely creative — the polyglot lab in particular is one of the most elegant attacks in the entire academy.*  
