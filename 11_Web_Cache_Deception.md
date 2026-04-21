# Web Cache Deception — Lab Write-ups

> **Topic:** Web Cache Deception  
> **Platform:** PortSwigger Web Security Academy  
> **Labs Completed:** All  
> **Tools Used:** Burp Suite, caching behaviour analysis, browser DevTools

---

## What is Web Cache Deception?

Web cache deception tricks a caching layer (CDN, reverse proxy, or application cache) into storing a private, user-specific page under a URL that an attacker can then fetch without authentication to get the cached victim data.

It's the conceptual opposite of cache poisoning: instead of poisoning what other users see, you're manipulating the cache into serving you something it should never have cached in the first place.

The root cause is always a mismatch between how the cache decides "is this cacheable?" and how the origin server decides "what page should I serve for this URL?" When these two interpretations differ, you can construct a URL that tricks the cache into treating a private page as a public, cacheable resource.

---

## Lab 1 — Web Cache Deception Basic

**Level:** Practitioner

**What the lab is about:**  
The cache stores responses based on file extension — if the URL ends in `.js`, `.css`, or `.png`, it gets cached. The origin application ignores unrecognized path suffixes and returns the authenticated user's page as normal. This creates the deception gap.

**What I did:**  
Crafted a URL that looks like a static file to the cache, but is served as the authenticated account page by the origin:

```
https://TARGET/my-account/exploit.js
```

Tricked the victim (simulated by the lab's "deliver to victim" function) into visiting this URL while logged in. The origin server ignored `/exploit.js`, treated the path as `/my-account`, and returned the victim's full account page — including their API key.

The cache saw a `.js` URL and cached the response. I then fetched the same URL without any authentication. Got the cached victim profile with the API key.

**Key takeaway:**  
The cache and the origin server have different URL interpretation logic. The cache uses extensions to decide cacheability; the origin ignores extra path segments. This mismatch is the entire attack. Any sensitive page can be leaked this way.

**Mitigation:**  
Add `Cache-Control: no-store` to all authenticated/personalized responses — this prevents them from being cached regardless of URL. Configure the cache to base caching decisions on response headers, not just URL patterns. Ensure URL parsing is consistent between the cache and origin.

---

## Lab 2 — Web Cache Deception with URL Normalization

**Level:** Practitioner

**What the lab is about:**  
The cache normalizes URLs before using them as cache keys — for example, resolving `%2F` to `/` or collapsing `../` sequences. The origin server processes the raw, un-normalized URL differently. This inconsistency creates a deception opportunity where the attacker can craft a URL that the cache normalizes to a known key but that the origin serves as a private page.

**What I did:**  
Analysed how the cache normalizes URLs versus how the origin parses them. Constructed a URL using encoded characters or path traversal that the cache would normalize to a predictable static-looking key, while the origin would interpret as the target private page.

Delivered this crafted URL to the victim — their authenticated response was cached under the normalized key. Fetched that key unauthenticated and retrieved the victim's session data.

**Key takeaway:**  
URL normalization inconsistencies between caching layers and origin servers are a precise technical root cause for cache deception. The cache and origin must agree on what URL means what — any divergence is exploitable.

**Mitigation:**  
Apply consistent URL normalization at both the cache and origin before making any caching or routing decision. Use response-header-based cache control (`Cache-Control: no-store`, `Vary`) rather than relying solely on URL pattern matching. Audit all encoding and normalization differences between infrastructure layers.

---

## Lab 3 — Web Cache Deception via Cache Key Injection

**Level:** Practitioner

**What the lab is about:**  
The cache includes certain request headers or parameters in the cache key — but the origin server doesn't use them. By injecting a crafted value into a header that's part of the cache key but not validated by the origin, you can construct a unique cache key for any target user's session and store their response there for retrieval.

**What I did:**  
Identified which headers the cache included in its cache key (e.g., `X-Cache-Salt`, a custom header). Found that the origin ignores this header entirely.

Crafted a URL and injected a known value in the header so the cache key was predictable to me. Delivered the crafted request to the victim — their authenticated response was stored under my crafted cache key. Then fetched the same key and retrieved their private data.

**Key takeaway:**  
Cache key construction is a security boundary. Any header or parameter that's included in the cache key but ignored by the origin is a potential injection vector for cache deception. The attacker just needs to know (or control) the cache key to retrieve any stored response.

**Mitigation:**  
Make cache key construction and origin request handling consistent. Don't include headers in the cache key that the origin doesn't validate. Use `Vary` headers carefully and audit all cache key components.

---

## Lab 4 — Web Cache Deception via Delimiter Confusion

**Level:** Practitioner

**What the lab is about:**  
The cache and origin handle URL delimiters (like `;`, `?`, `#`) differently. A character that the origin treats as a path terminator (ending the route lookup) is treated by the cache as part of the path (affecting the cache key). This creates a gap: you can append cacheable-looking suffixes that the origin ignores.

**What I did:**  
Tested different delimiter characters to find one where the origin ignored everything after it (treating the base path as the route) but the cache included everything in the cache key.

For example:

```
/my-account;exploit.css
```

Origin: parsed up to `;`, served `/my-account` response (private, authenticated).  
Cache: saw a `.css` extension at the end of the full string and cached it.

Delivered to the victim → their account page cached. Retrieved unauthenticated using the same URL.

**Key takeaway:**  
Delimiter handling is another parser inconsistency vulnerability class. The characters `;`, `?`, `#`, and null bytes all behave differently across different frameworks, CDNs, and reverse proxies. Testing for delimiter-based discrepancies is an important part of cache deception research.

**Mitigation:**  
Normalize URLs at the cache layer before key generation. Strip or reject unusual delimiter characters from URLs before passing to the cache or origin. Use `Cache-Control: no-store` on all authenticated responses.

---

## Lab 5 — Web Cache Deception via Static Directory Cache Rules

**Level:** Practitioner

**What the lab is about:**  
The cache is configured to cache everything under `/static/`, `/assets/`, or similar directories. The origin uses URL path traversal-style routing that serves authenticated content even for these paths. Combining these two behaviours exposes private pages under cacheable paths.

**What I did:**  
Found that requests to `/static/../my-account` were served by the origin as the `/my-account` authenticated page (path traversal resolved server-side) but cached by the CDN under a `/static/` path rule (which matched the rule to cache static directories).

Delivered this path-traversal URL to the victim. Their authenticated profile response was cached under the `/static/` directory rule. Retrieved it unauthenticated.

**Key takeaway:**  
Cache rules based on directory prefixes assume those directories only contain static content. Path traversal can break that assumption by making the origin serve dynamic content from an address that looks like a static path to the cache.

**Mitigation:**  
Resolve and normalize path traversal sequences before applying cache rules. Never cache responses to URLs containing `../` or encoded equivalents. Apply `Cache-Control: no-store` on all authenticated responses to act as a final safety net.

---

*Web cache deception is a relatively newer vulnerability class but it's increasingly important as CDNs and caching proxies become standard parts of every web application stack. The key insight that makes all of these labs click: the cache and the origin server are two separate systems with different URL interpretation logic, and any disagreement between them is potentially exploitable.*
