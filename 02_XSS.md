# Cross-Site Scripting (XSS) — Lab Write-ups

> **Topic:** Cross-Site Scripting (XSS)  
> **Platform:** PortSwigger Web Security Academy  
> **Labs Completed:** Apprentice + Practitioner  
> **Tools Used:** Burp Suite, Browser DevTools

---

## What is XSS?

XSS lets an attacker inject client-side scripts into web pages that other users view. Unlike SQLi which attacks the server, XSS attacks the victim's browser. You're essentially making someone else's browser run your code — and that code can steal cookies, hijack sessions, redirect users, deface pages, or silently capture keystrokes.

Three main types:
- **Reflected** — payload comes from the HTTP request and is immediately reflected back
- **Stored** — payload is saved in the database and served to every user who loads the page
- **DOM-based** — the vulnerability lives in client-side JavaScript, never touches the server

---

## Lab 1 — Reflected XSS into HTML Context, No Encoding
**Level:** Apprentice

**What the lab is about:**  
The search box reflects user input directly into the HTML response without any encoding. Basic proof-of-concept.

**What I did:**  
Searched for:
```
<script>alert(1)</script>
```
The page reflected it straight into the HTML and the browser executed it. Alert popped.

**Key takeaway:**  
If an application echoes your input back without encoding `<`, `>`, or `"`, you have reflected XSS. Test every input field.

**Mitigation:**  
HTML-encode all user-controlled output. Use a templating engine that auto-escapes by default.

---

## Lab 2 — Stored XSS into HTML Context, No Encoding
**Level:** Apprentice

**What the lab is about:**  
The comment section on blog posts stores user input without sanitization and renders it back to every visitor.

**What I did:**  
Posted a comment with the body:
```
<script>alert(1)</script>
```
Every time anyone loaded that blog post, the script executed in their browser.

**Key takeaway:**  
Stored XSS is worse than reflected because it doesn't require tricking a victim into clicking a link — it runs automatically for every user who visits the page.

**Mitigation:**  
Sanitize input on the way in, encode output on the way out. Use a proper HTML sanitization library (not a regex).

---

## Lab 3 — DOM XSS in `document.write` Sink Using Source `location.search`
**Level:** Apprentice

**What the lab is about:**  
The page reads from `location.search` (the URL query string) and passes it directly into `document.write()`. No server involved — the vulnerability lives entirely in the client-side JavaScript.

**What I did:**  
Looked at the page source and found something like:
```javascript
document.write('<img src="/resources/images/tracker.gif?searchTerms=' + query + '">');
```
Injected into the search parameter:
```
"><svg onload=alert(1)>
```
The `">` broke out of the `img` tag attribute, and the SVG element triggered the alert on load.

**Key takeaway:**  
DOM XSS requires reading JavaScript source, not just looking at HTTP responses. Tools like Burp's DOM Invader help massively.

**Mitigation:**  
Never pass untrusted data to dangerous sinks like `document.write`, `innerHTML`, or `eval`. Use `textContent` instead.

---

## Lab 4 — DOM XSS in `innerHTML` Sink
**Level:** Apprentice

**What the lab is about:**  
Similar to the previous but using `innerHTML` as the sink. Note that `<script>` tags don't execute when injected via `innerHTML` — you need an event-based payload.

**What I did:**  
```
<img src=1 onerror=alert(1)>
```
The `src=1` immediately fails (invalid URL), triggering `onerror` which runs the alert.

**Key takeaway:**  
`innerHTML` won't run `<script>` tags, but HTML event handlers like `onerror`, `onload`, `onmouseover` work fine. Always adapt your payload to the context.

---

## Lab 5 — DOM XSS in jQuery Anchor `href` Attribute
**Level:** Apprentice

**What the lab is about:**  
A jQuery script takes a URL parameter and sets it as the `href` of a "Back" link using `attr()`. If the value starts with `javascript:`, the browser will execute it when clicked.

**What I did:**  
Modified the `returnPath` parameter in the URL to:
```
javascript:alert(document.cookie)
```
The back link now had `href="javascript:alert(document.cookie)"`. Clicking it executed the script.

**Key takeaway:**  
`javascript:` URIs in `href` attributes are a classic vector. jQuery's `attr()` doesn't sanitize URLs.

**Mitigation:**  
Validate that URLs start with `http://` or `https://` before assigning them to `href`.

---

## Lab 6 — DOM XSS in jQuery Selector Sink using Hashchange Event
**Level:** Apprentice

**What the lab is about:**  
The page uses `$(location.hash)` to auto-scroll to a section. jQuery's `$()` selector can accept HTML — and will parse and execute it if given a string starting with `<`.

**What I did:**  
Crafted a URL with the hash:
```
#<img src=1 onerror=alert(1)>
```
When the page loaded, the hashchange handler passed this to `$()`, which created and inserted the `img` element, triggering `onerror`.

**Key takeaway:**  
jQuery's `$()` is a dangerous sink. Passing user-controlled values from `location.hash` into it is a common DOM XSS pattern.

---

## Lab 7 — Reflected XSS into Attribute with Angle Brackets HTML-Encoded
**Level:** Practitioner

**What the lab is about:**  
The application HTML-encodes `<` and `>`, so you can't inject tags. But the reflected value lands inside an HTML attribute — meaning you can inject a new attribute.

**What I did:**  
The search term was reflected inside a `value=""` attribute:
```html
<input value="YOURTERM">
```
I injected:
```
" onmouseover="alert(1)
```
Which rendered as:
```html
<input value="" onmouseover="alert(1)">
```
Hovering over the input field triggered the XSS.

**Key takeaway:**  
Even without angle brackets, if you control a value inside an attribute, you can inject event handlers by breaking out of the attribute with a `"`.

---

## Lab 8 — Stored XSS into Anchor `href` Attribute with Double Quotes HTML-Encoded
**Level:** Practitioner

**What the lab is about:**  
The "website" field in the comment form gets put into an `<a href="">` tag. Double quotes are encoded but the content of `href` isn't validated.

**What I did:**  
Set the website field to:
```
javascript:alert(1)
```
The resulting HTML:
```html
<a href="javascript:alert(1)">Visit website</a>
```
Clicking "Visit website" fires the alert.

**Key takeaway:**  
URL fields need protocol validation, not just quote encoding. Always check `href`, `src`, and `action` attributes for `javascript:` injection.

---

## Lab 9 — Reflected XSS into a JavaScript String with Angle Brackets HTML-Encoded
**Level:** Practitioner

**What the lab is about:**  
The search term is reflected inside a JavaScript string literal in a `<script>` block. Angle brackets are encoded but quotes are not.

**What I did:**  
Found this in the page source:
```javascript
var searchTerms = 'YOURTERM';
```
Injected:
```
'-alert(1)-'
```
Which produced:
```javascript
var searchTerms = ''-alert(1)-'';
```
The string is broken, `alert(1)` runs as a standalone expression.

**Key takeaway:**  
XSS inside JavaScript context is different from HTML context. You need to break out of the JS string, not an HTML tag.

---

## Lab 10 — DOM XSS in `document.write` Sink Inside a Select Element
**Level:** Practitioner

**What the lab is about:**  
User input from a URL parameter flows into `document.write()` which builds a `<select>` element. Breaking out of the select context requires closing the select first.

**What I did:**  
```
product=1</select><img src=1 onerror=alert(1)>
```
Closed the `</select>` tag and injected an img element with an error handler.

**Key takeaway:**  
Context matters. When injecting inside a `<select>`, you need to escape that context before your XSS payload can execute.

---

## Lab 11 — XSS into HTML Context with Most Tags Blocked
**Level:** Practitioner

**What the lab is about:**  
A WAF blocks most common HTML tags. The goal is to find a tag that isn't blocked and use it to execute script.

**What I did:**  
Used Burp Intruder to fuzz all HTML tags from PortSwigger's XSS cheat sheet. Found that `<body>` was allowed. Then fuzzed event handlers and found `onresize` worked:
```html
<body onresize=alert(document.cookie)>
```
Delivered it via an iframe in the exploit server:
```html
<iframe src="https://TARGET/?search=<body onresize=print()>" onload="this.style.width='100px'">
```
Resizing the iframe triggers the `onresize` handler.

**Key takeaway:**  
When obvious tags are blocked, systematically fuzz all tags and all events. The cheat sheet at PortSwigger is invaluable here.

---

## Lab 12 — Reflected XSS with Some SVG Markup Allowed
**Level:** Practitioner

**What the lab is about:**  
`<svg>` tags are allowed but most events are blocked. You need to find an SVG-specific event that gets through.

**What I did:**  
Fuzzed SVG events and found `<animatetransform>` with `onbegin` was not blocked:
```html
<svg><animatetransform onbegin=alert(1) attributeName=transform>
```

**Key takeaway:**  
SVG has its own set of event handlers that basic WAFs don't always account for. `onbegin`, `onend`, `onrepeat` are less commonly filtered.

---

## Lab 13 — Reflected XSS in Canonical Link Tag
**Level:** Practitioner

**What the lab is about:**  
Input is reflected inside a `<link rel="canonical">` tag's `href`. You can't use event handlers normally — but access keys can trigger events.

**What I did:**  
Injected:
```
/?'accesskey='x'onclick='alert(1)
```
Which produced:
```html
<link rel="canonical" href="https://site.com/" accesskey="x" onclick="alert(1)"/>
```
Pressing `Alt+Shift+X` (on Windows) triggered the click event.

**Key takeaway:**  
Canonical tags are often overlooked during security reviews. Access keys are a clever workaround when normal events are unavailable.

---

*XSS is deceptively wide. The core idea is simple but the attack surface is huge — every place user input touches the DOM is a potential target.*
