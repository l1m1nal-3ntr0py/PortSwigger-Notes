# Race Conditions — Part 3: Advanced Techniques & Prevention
**Source:** PortSwigger Web Security Academy
**Status:** ✅ Completed
**Date:** September 2026
**Author:** [l1m1nal_3ntr0py](https://github.com/l1m1nal-3ntr0py)

---

## Table of Contents
1. [Partial Construction Race Conditions](#partial-construction-race-conditions)
2. [NULL Injection Syntax](#null-injection-syntax)
3. [Time-Sensitive Attacks](#time-sensitive-attacks)
4. [Prevention](#prevention)
5. [Complete Quick Reference](#complete-quick-reference)

---

## Partial Construction Race Conditions

Applications often create objects in multiple steps — leaving a temporary incomplete state between operations.

**Example — user registration:**
```
Step 1: INSERT user → exists! api_key = NULL ← partial!
Step 2: Generate API key
Step 3: UPDATE user with api_key → complete ✅

Race window = between Step 1 and Step 3
```

**SQL:**
```sql
-- Step 1: User created (incomplete!)
INSERT INTO users (username, email)
VALUES ('carlos', 'carlos@test.com')
-- api_key = NULL here!

-- Step 2: Set API key
UPDATE users SET api_key = 'abc123'
WHERE username = 'carlos'
```

**Attack — submit NULL during window:**
```
Security check:
if submitted_key == stored_key: grant_access()

During partial state:
submitted_key = NULL (empty array)
stored_key    = NULL (not set yet!)
NULL == NULL  → TRUE! 🎯 Access granted!
```

**Attack timeline:**
```
t=0ms: POST /register received
t=1ms: User created → api_key = NULL ← partial!
t=2ms: ← RACE WINDOW — your request hits!
       submitted = [] (NULL)
       stored = NULL
       MATCH! Access granted! 🎯
t=3ms: Real api_key generated (too late!)
```

---

## NULL Injection Syntax

**PHP:**
```
param[]          → empty array → NULL
param[]=foo      → ['foo']
param[key]=foo   → {'key': 'foo'}
```

**Ruby on Rails:**
```
param[key]=      → {'key': nil} → NULL!
```

**Express/Node:**
```
param[]=         → [] → NULL
```

**HTTP request example:**
```
GET /api/user/info?user=victim&api-key[]= HTTP/2
                                       ↑ empty = NULL!
```

**Password version:**
```
API key: plaintext → NULL == NULL (direct)

Password: hashed → need hash(NULL) or hash("")

If hash("") = d41d8cd98f00b204e9800998ecf8427e
Submit: password= (empty string)
Server hashes → same value → LOGIN! 💀
```

---

## Time-Sensitive Attacks

When timestamp-based tokens are used instead of cryptographically secure random values.

**Detection:**
```
Send two reset requests simultaneously (single-packet)
Check both emails:

Same token in both emails → timestamp-based! ❌
Different tokens → secure random! ✅
```

**Example:**
```
Request 1: Reset wiener → token = 1693847291234
Request 2: Reset carlos → token = 1693847291234
                                   ↑ identical!
Timestamp-based confirmed! 💀
```

**Impact:** Can predict or reproduce tokens for any user.

---

## Prevention

### Make State Changes Atomic

```sql
-- Vulnerable: two operations with window
SELECT used FROM coupons WHERE code = 'SAVE50';
-- race window!
UPDATE coupons SET used = true WHERE code = 'SAVE50';

-- Safe: single atomic operation
UPDATE coupons SET used = true
WHERE code = 'SAVE50' AND used = false;
-- 0 rows affected = already used!
```

### Database Constraints

```sql
-- Add uniqueness at DB level
ALTER TABLE coupon_uses
ADD UNIQUE (user_id, coupon_id);
-- Second insert → DB error → race stopped!
```

### Never Mix Storage Layers

```
❌ Check session → Update database
   (session check can be raced)

✅ Atomic database transaction
   (DB handles own concurrency)
```

### Batch Session Updates

```python
# Vulnerable — separate updates create windows:
session['userid'] = user.id
# window here!
session['enforce_mfa'] = True

# Safe — batch update:
session.update({
    'userid': user.id,
    'enforce_mfa': True
})
```

### Cryptographically Secure Tokens

```python
# Vulnerable:
token = str(time.time_ns())     ← predictable!

# Safe:
import secrets
token = secrets.token_hex(32)   ← unpredictable ✅
```

### Database Transaction for Payment

```python
# Safe — atomic payment + confirmation:
with db.transaction():
    validate_payment(cart, payment)
    confirm_order(cart)
    # Either both succeed or both fail!
    # No window between validation and confirmation
```

---

## Complete Quick Reference

```
PART 1 — LIMIT OVERRUN
Target:   Single-use features, rate-limited endpoints
Attack:   Parallel requests before update fires
Tool:     Burp Repeater parallel / Turbo Intruder
Fix:      Atomic UPDATE with WHERE condition

PART 2 — HIDDEN WINDOWS
Target:   Multi-step single requests, email operations
Attack:   Race between sub-states, multi-endpoint
Tool:     Connection warming, rate limit abuse
Fix:      Batch session updates, atomic transactions

PART 3 — ADVANCED
Target:   Incomplete objects, timestamp tokens
Attack:   NULL injection during partial construction
Tool:     Framework NULL syntax, timing observation
Fix:      DB constraints, secure random tokens

TURBO INTRUDER TEMPLATE
engine = RequestEngine(
    endpoint=target.endpoint,
    concurrentConnections=1,
    engine=Engine.BURP2
)
for i in range(20):
    engine.queue(target.req, gate='1')
engine.openGate('1')

BURP REPEATER STEPS
1. Send to Repeater
2. Create tab group
3. Duplicate 10-20 times
4. Send group in parallel

FIND RACE CONDITIONS
→ Read-then-write endpoints
→ Single-use features
→ Multi-step single requests
→ Email background operations
→ Registration/creation flows
→ Payment + confirmation flows
→ Timestamp-based token generation
```

---

*Notes by [l1m1nal_3ntr0py](https://github.com/l1m1nal-3ntr0py) | PortSwigger Web Security Academy*
*Part 1 | Part 2 | Part 3*
