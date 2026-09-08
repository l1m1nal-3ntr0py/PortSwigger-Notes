# Race Conditions — Part 2: Hidden Windows
**Source:** PortSwigger Web Security Academy
**Status:** ✅ Completed
**Date:** September 2026
**Author:** [l1m1nal_3ntr0py](https://github.com/l1m1nal-3ntr0py)

---

## Table of Contents
1. [Hidden Multi-Step Sequences](#hidden-multi-step-sequences)
2. [How to Spot Sub-States](#how-to-spot-sub-states)
3. [Multi-Endpoint Race Conditions](#multi-endpoint-race-conditions)
4. [Aligning Race Windows](#aligning-race-windows)
5. [Single-Endpoint Race Conditions](#single-endpoint-race-conditions)
6. [Session-Based Locking](#session-based-locking)
7. [Quick Reference](#quick-reference)

---

## Hidden Multi-Step Sequences

A single request often triggers multiple backend operations — each creating a brief window where the app is in an in-between state.

**Sub-state = temporary state between operations**

**MFA bypass example:**
```python
# Step 1: Set user as logged in
session['userid'] = user.userid      ← logged in here!

# Step 2: Check if MFA needed
if user.mfa_enabled:
    session['enforce_mfa'] = True    ← MFA enforced here!
```

**Race attack:**
```
Login request   ──────────────► userid set ←──────┐
/dashboard ──────────────────► hits here! ✅       │
                               MFA not set yet!    │
                               ACCESS GRANTED!     │
                               MFA enforced ───────┘ (too late!)
```

---

## How to Spot Sub-States

**Trigger different errors on same endpoint:**
Send intentionally wrong data to force server through different code paths. Watch for unexpected responses.

**Look for multi-step processes:**
```
→ Login that sets multiple session variables
→ Registration that creates AND initialises user
→ Payment that validates AND confirms
→ Email change that sends AND records
```

**Watch for second-order effects:**
```
→ Emails with wrong content
→ Unexpected session state changes
→ Confirmation of incomplete actions
→ Responses referencing other request data
```

**Methodology — Predict, Probe, Prove:**
```
1. PREDICT  → Is this endpoint security critical?
              Does it touch shared data?
              Is there collision potential?

2. PROBE    → Benchmark sequential behaviour
              Send group in parallel
              Look for ANY deviation

3. PROVE    → Isolate the collision
              Remove unnecessary requests
              Confirm reproducibility
```

---

## Multi-Endpoint Race Conditions

**Classic basket trick:**
```
Normal:
Add item → Pay → Confirm order

Race:
POST /payment ──────────────────► validates payment
POST /cart/add ──────────────────► adds item
               ↑ sent simultaneously!

Payment validated ──────────────────┐
Add item hits window! 💥 ───────────┤
Order confirmed with extra item! ───┘
```

---

## Aligning Race Windows

**Problem:** Different endpoints process at different speeds.

**Solution 1 — Connection warming:**
```
Step 1: Add GET / to start of tab group
Step 2: Send group in sequence (single connection)
Step 3: First request slower = back-end delay confirmed
Step 4: Remove GET / — subsequent requests now aligned
Step 5: Run parallel attack
```

**Solution 2 — Rate limit abuse:**
```
Step 1: Flood server with 50+ dummy requests
Step 2: Server triggers rate limit → queues everything
Step 3: Fast endpoint slows to match slow endpoint
Step 4: Attack requests join queue → both aligned!

Turbo Intruder:
for i in range(50):
    engine.queue(dummyRequest, gate='1')
engine.queue(paymentRequest, gate='1')
engine.queue(cartAddRequest, gate='1')
engine.openGate('1')
```

**Solution 3 — Request padding:**
```
Add useless data to fast request body to slow it down:
POST /cart/add
body: item=x&padding=aaaaaaaaaaaaaaa...
             ↑ artificial delay
```

---

## Single-Endpoint Race Conditions

**Password reset collision:**
```python
# Vulnerable server code:
session['reset-user'] = username    ← overwritable!
session['reset-token'] = generate_token()
send_email(username, token)
```

**Sequential (safe):**
```
Request 1 (wiener): reset-user=wiener, token=5678 → email to wiener
Request 2 (carlos): reset-user=carlos, token=1234 → email to carlos
```

**Parallel (race):**
```
Request 1 (wiener) ──────────────────── token=5678
Request 2 (carlos) ───── token=1234 (overwrites!)

Final state:
reset-user  = carlos  ← from request 2
reset-token = 5678    ← from request 1 (overwrote!)

Email sent to wiener with token 5678
Token 5678 now resets CARLOS! 💀
```

**Burp steps:**
```
1. Login as wiener → get session cookie
2. Request password reset for wiener → intercept
3. Send to Repeater → duplicate tab
4. Change username to carlos in copy
5. Group both → Send group in parallel
6. Check wiener's email for token
7. Use token → resets carlos password!
```

**Vulnerable vs safe storage:**
```
VULNERABLE — session:
session['reset-user'] = carlos     ← shared! overwritable!
session['reset-token'] = 1234

SAFE — database:
DB.save(user=carlos, token=1234)   ← atomic! linked directly!
```

> **Note:** Email operations sent in background threads after HTTP response — larger race windows!

---

## Session-Based Locking

**Detection:**
```
No locking:
Request 1 → 200ms
Request 2 → 201ms ← almost same! ✅ good for attacks

Session locking:
Request 1 → 200ms
Request 2 → 400ms ← waited! ❌ sequential processing
Request 3 → 600ms ← waited for both!
```

**Bypass — different session tokens:**
```
Instead of: all requests use session=ABC (queued!)
Use:        Request 1 → session=ABC
            Request 2 → session=DEF  ← parallel!
            Request 3 → session=GHI
```

**Important:** Session locking only stops session-stored attacks!
```
Session storage → locking works → attack fails ❌
Database storage → different sessions still race → attack works ✅
```

---

## Quick Reference

```
FIND HIDDEN SUB-STATES
→ Trigger errors to reveal code paths
→ Look for multi-step single requests
→ Watch for second-order effects

MULTI-ENDPOINT
→ Basket + payment simultaneously
→ Fix timing: connection warming
→ Fix timing: rate limit abuse
→ Fix timing: request padding

SINGLE-ENDPOINT
→ Parallel requests with different values
→ Session storage = collision possible
→ Email operations = large windows

SESSION LOCKING
→ Detect: sequential response timing
→ Bypass: different session per request
→ Only stops session-stored attacks
```

---

*Part 3: Partial construction, time-sensitive attacks, prevention*

*Notes by [l1m1nal_3ntr0py](https://github.com/l1m1nal-3ntr0py) | PortSwigger Web Security Academy*
