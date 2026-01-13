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


---


