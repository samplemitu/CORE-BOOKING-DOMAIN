# CORE-BOOKING-DOMAIN

## 1️⃣ What is a “Booking System” REALLY?

A booking system is **NOT**:

* CRUD APIs
* Forms + payment
* Availability table

A booking system **IS**:

> **A distributed state machine that manages scarce resources over time under concurrency.**

Scarce resources =

* Seats (flights, buses)
* Rooms (hotels)
* Cars (Uber)
* Slots (appointments, courts)

Time-bound + conflict-prone + money-involved.

---

## 2️⃣ The 6 Core Domain Objects (Non-Negotiable)

Every serious booking system reduces to these:

### 1. **Inventory**

What can be booked

* Room, seat, vehicle, slot
* Has **capacity**
* Exists across **time**

### 2. **Availability**

Derived, NOT stored blindly

* Inventory – existing reservations – blocks
* Time-based (date, hour, minute)

### 3. **Reservation (Hold)**

Temporary lock

* Prevents overbooking
* Has **expiry (TTL)**
* Not paid yet

### 4. **Booking (Confirmed)**

Permanent allocation

* Paid or guaranteed
* Must survive crashes

### 5. **Pricing**

Dynamic, rule-based

* Demand, season, surge
* Coupons, taxes, fees

### 6. **Cancellation / Modification**

The hardest part

* Refund rules
* Partial release of inventory
* Penalties

---

## 3️⃣ Booking Lifecycle (End-to-End)

This flow must be **muscle memory**.

![Image](https://www.researchgate.net/publication/220921642/figure/fig1/AS%3A905458810384384%401592889611199/Life-cycle-diagram-of-RESERVATION.png)

![Image](https://miro.medium.com/v2/resize%3Afit%3A1400/1%2A9YdkkJIWwBI1uqGa7fkpXg.png)

![Image](https://online.visual-paradigm.com/repository/images/e2de47a1-d698-46fe-83fd-c45b9600e606.png)

### Step-by-step (idealized):

1. **Search**

   * User asks: “What’s available?”
   * Read-heavy, cached, approximate OK

2. **Check Availability**

   * Real-time calculation
   * Must consider:

     * Existing bookings
     * Active holds
     * Maintenance blocks

3. **Create Reservation (HOLD)**

   * Lock inventory
   * Set TTL (e.g., 10 min)
   * Must be **atomic**

4. **Price Finalization**

   * Recalculate price
   * Lock price version

5. **Payment**

   * Async
   * Can fail, retry, timeout

6. **Confirm Booking**

   * Convert HOLD → BOOKED
   * Persist permanently
   * Emit events

7. **Post-Booking**

   * Notifications
   * Invoice
   * Analytics
   * Cancellation window

---

## 4️⃣ Hard Truths (Most Devs Miss This)

### ❌ Availability is NOT a boolean

It’s:

* Time-ranged
* Capacity-aware
* Concurrently changing

### ❌ Booking ≠ Payment

Payment is **external + unreliable**
Booking must be **internally consistent**

### ❌ Overbooking is a business choice

Sometimes intentional:

* Airlines
* Hotels

Engineering must **support policy**, not assume perfection.

---

## 5️⃣ Concurrency Is the Real Enemy

Two users booking the **same resource at the same second**.

If your system fails here → company loses money.

Key problems:

* Race conditions
* Double booking
* Lost holds
* Zombie reservations
---

## 6️⃣ Mental Model I Expect You to Use

Think like this:

> “Inventory is a timeline, not rows in a table.”

Example (Hotel Room #101):

```
Jan 10 ─────────────── Jan 12
         BOOKED

Jan 12 ─── Jan 14
         AVAILABLE

Jan 14 ─── Jan 15
         HOLD (expires in 7 min)
```

If you **don’t think in timelines**, you’ll design wrong schemas.

---

## 7️⃣ Minimal Vocabulary You MUST Speak Fluently

If you pause on these → Half knowledge.

* Inventory vs Availability
* Hold vs Booking
* Idempotency
* TTL-based reservation
* Eventual consistency
* Overbooking buffer
* Soft lock vs hard lock
* Compensation (rollback via events)

---
# Must know real booking-domain topics 📘🧾 .

---

## 1️⃣ Why do we need a HOLD step?

### Short answer

Because **money is slow and inventory is scarce**.

### Deep truth

A HOLD is a **temporary, expiring ownership claim** on inventory.

Without HOLD:

* Two users can pay at the same time
* Both believe they booked
* One must be refunded → trust loss + ops cost

### HOLD solves 4 hard problems at once

**1. Concurrency**

* Prevents double booking under parallel requests

**2. Payment uncertainty**

* Payment gateways are async, flaky, retry-based

**3. User behavior**

* Users abandon, reload, switch tabs

**4. Time-based fairness**

* “First valid payer within TTL wins”

### Real-world analogy

Think of HOLD like:

> Putting an item in your cart at a physical store
> The cashier waits—but only for a few minutes.

> “HOLD decouples inventory locking from payment finality using time-bound ownership.”

---

## 2️⃣ What happens if payment succeeds but booking confirmation fails?

This **WILL** happen in production. Guaranteed.

### Possible reasons

* DB crash
* Network timeout
* Service restart
* Partial write
* Event bus failure

### Correct mental model

**Payment ≠ Booking**

Payment is:

* External
* Eventually consistent
* Retryable

Booking is:

* Internal
* Must be **exactly-once**
* Source of truth

---
### Step-by-step recovery-safe flow

1. **Payment succeeds**

   * Payment gateway sends success
   * You store `payment_reference_id`

2. **Booking confirmation fails**

   * HOLD still exists
   * Inventory still locked

3. **Recovery job / consumer**

   * Detects: `PAID + HOLD_ACTIVE + BOOKING_NOT_CREATED`
   * Retries booking confirmation **idempotently**

4. **If booking eventually succeeds**

   * Convert HOLD → BOOKED
   * Emit confirmation events

5. **If HOLD expires before recovery**

   * Trigger **compensation**
   * Auto-refund payment

### Golden rule

> Never refund immediately unless recovery has failed conclusively.

---

## Absolute MUST

Booking confirmation **must be idempotent**.

Same request 10 times → same result.

---

# 3️⃣ How do you release inventory safely on timeout?

This is where most systems silently break.

---

## WRONG approach ❌

* Cron job deletes expired holds
* Inventory recalculated blindly

Race condition city.

---

## CORRECT approach ✅

### Principles

* Release must be **atomic**
* Must handle:

  * Late payment
  * Concurrent confirmation
  * Retry storms

---

### Safe release flow

1. HOLD has:

   * `expires_at`
   * `status = ACTIVE`

2. Background worker:

   * Finds expired HOLDs
   * Tries **CAS update**:

     ```
     UPDATE holds
     SET status = EXPIRED
     WHERE id = ?
       AND status = ACTIVE
       AND expires_at < now()
     ```

3. If update succeeds:

   * Emit `HOLD_EXPIRED` event
   * Inventory is implicitly freed

4. If update fails:

   * Someone else acted (payment/booking)
   * Do nothing

---

### Key idea

> Never “free inventory” directly.
> Inventory is freed by **state transition**, not deletion.

---

# 4️⃣ Why availability should NOT be precomputed fully?

Because **availability is a moving target**.

---

## Why people try precomputing

* Faster reads
* Simple queries

## Why it fails at scale

### 1. Time dimension explosion

Rooms × Dates × Slots = insane cardinality

### 2. Concurrency drift

* HOLDs change availability every second
* Precomputed data becomes stale immediately

### 3. Business rules change

* Overbooking
* Blackouts
* Promotions
* Maintenance blocks

### 4. Partial failures

* One missed update = corrupted availability

---

## Correct strategy (used by top platforms)

### Hybrid approach

**Strong source of truth**

* Bookings
* Active holds
* Inventory capacity

**Derived availability**

* Calculated on read
* Cached short-term (seconds)
* Invalidated via events

> “Availability is a derived, time-scoped projection, not a primary data model.”

---

# 5️⃣ What data must be STRONGLY consistent vs EVENTUALLY consistent?

---

## STRONGLY CONSISTENT (Never compromise)

These **must be correct immediately**:

* Inventory capacity allocation
* HOLD creation
* HOLD → BOOKED transition
* Booking confirmation
* Payment ↔ booking linkage
* Cancellation that frees inventory

💡 Rule:

> If it affects **money or inventory**, it must be strongly consistent.

---

## EVENTUALLY CONSISTENT (Safe to lag)

These can be delayed:

* Search results
* Availability cache
* Notifications
* Emails / SMS
* Analytics
* Recommendations
* Reporting dashboards

💡 Rule:

> If delay does not cause double booking or money loss, eventual is fine.

---

# 6️⃣ Extra Booking Topic You MUST Know (⌛)
---

## Q: Why not lock DB rows directly?

**A:** Doesn’t scale, causes deadlocks, kills throughput.

---

## Q: Soft lock vs Hard lock?

* **Soft lock**: Logical (HOLD with TTL)
* **Hard lock**: DB lock (avoid unless unavoidable)

---

## Q: What if payment comes after HOLD expiry?

**A:** Treat as late → refund or reattempt booking only if inventory still free.

---

## Q: Why not confirm booking before payment?

**A:** Fraud risk + rollback complexity.

---

## Q: How do you prevent duplicate bookings on retries?

**A:** Idempotency keys + unique booking constraints.

---

## Q: Can bookings be eventually consistent?

**A:** No. Bookings are the system of record.

---

## Q: What breaks booking systems most often?

* Bad TTL handling
* Non-idempotent APIs
* Assuming payment is reliable
* Treating availability as static

---

# 7️⃣ One Mental Model to Rule Everything

If you remember **only this**, you’re safe:

> Booking systems are **state machines**, not CRUD apps.

States:

```
AVAILABLE → HOLD → BOOKED → (CANCELLED)
              ↓
           EXPIRED
```

All correctness comes from **controlled state transitions**.

---


