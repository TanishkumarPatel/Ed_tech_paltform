# 🔐 ACID & Concurrency Case Study — The Last-Seat Problem

> A walkthrough of a real concurrency issue in this schema: what happens when two students try to grab the last open seat in a batch at the same moment, why the obvious approach breaks, and how to fix it correctly.

---

## 📌 The Scenario

A course/batch has a fixed capacity — say **50 seats**. 49 students have already enrolled, so **1 seat is left**. Two students, A and B, hit "Enroll" at (essentially) the same instant.

What **should** happen: exactly one of them gets the seat, and the other is told the batch is full.

What **can** happen without concurrency control: both get enrolled, and the batch silently ends up with 51/50 — no error, no warning, just a broken constraint.

---

## 🧬 Why This Is an ACID Problem

| Property | How it applies here |
|---|---|
| **Atomicity** | "Check remaining seats" and "create the enrollment" must succeed or fail together as one unit, not as two separate steps with a gap in between |
| **Consistency** | The business rule *enrollments for a course ≤ its seat capacity* must hold true before and after every transaction |
| **Isolation** | Two concurrent enrollment transactions must not both act on a seat count that the other is about to invalidate |
| **Durability** | Once the database confirms a seat is booked, it stays booked even after a crash — not the bug in this scenario, but relevant once a payment/confirmation step is added |

The bug here is really an **isolation failure** that produces a **consistency violation**.

---

## 🐛 What Goes Wrong Without Concurrency Control

The obvious-looking implementation checks and inserts as two separate steps:

```sql
-- Step 1: check remaining seats
SELECT COUNT(*) FROM enrollment WHERE course_name = 'JEE Master Course';
-- returns 49 → 1 seat free out of 50

-- Step 2: if a seat is free, insert
INSERT INTO enrollment (id, course_name, enrollment_date, phone)
VALUES (101, 'JEE Master Course', CURRENT_DATE, '9876543299');
```

Trace what happens when Person A and Person B both run this at the same time:

| Time | Transaction A | Transaction B |
|---|---|---|
| t1 | `SELECT COUNT(*)` → 49 (1 seat free) | |
| t2 | | `SELECT COUNT(*)` → 49 (A hasn't committed yet, so B sees the same stale count) |
| t3 | `INSERT` succeeds | |
| t4 | | `INSERT` succeeds |
| t5 | `COMMIT` | `COMMIT` |

Both transactions read the same "1 seat free" snapshot before either had written anything, so both proceeded. Result: **51 students enrolled in a 50-seat batch.** This is a textbook **lost update / race condition** — nothing ever told the database these two operations needed to be mutually exclusive.

---

## ✅ What Should Happen Instead

Exactly one of the following, deterministically:

- Person A enrolls successfully and **Person B's request is rejected** with a "batch full" error (or vice versa) — whoever's transaction commits first wins, or
- Both requests are serialized so the second one **re-checks the real seat count** before deciding, instead of trusting a count it read before the first request finished.

Either way, the database — not the application's timing — should be what guarantees only 50 rows ever exist for that course.

---

## 🛠️ The Fix

### 1. Schema addition

`courses` needs a capacity to check against:

```sql
ALTER TABLE courses ADD COLUMN max_seats INTEGER NOT NULL DEFAULT 50;
ALTER TABLE courses ADD COLUMN seats_filled INTEGER NOT NULL DEFAULT 0;
```

`seats_filled` is a running counter kept in sync with `enrollment`. (The alternative is to always compute `COUNT(*) FROM enrollment` live — also correct, just slightly more expensive at scale.)

### 2. Recommended fix — atomic conditional `UPDATE`

Combine the check and the reservation into **one statement** instead of two:

```sql
BEGIN;

UPDATE courses
SET seats_filled = seats_filled + 1
WHERE name = 'JEE Master Course'
  AND seats_filled < max_seats
RETURNING seats_filled;

-- 0 rows affected → batch is full → ROLLBACK and reject the request
-- 1 row affected  → the seat is reserved → safe to proceed:

INSERT INTO enrollment (id, course_name, enrollment_date, phone)
VALUES (101, 'JEE Master Course', CURRENT_DATE, '9876543299');

COMMIT;
```

**Why this works:** an `UPDATE ... WHERE` is atomic on that row. PostgreSQL locks the row for the duration of the update, so if A and B run this at the same moment, one of them **waits** for the other's lock — then re-evaluates `seats_filled < max_seats` against the *post-commit* value, not the stale value it started with. The second transaction correctly gets 0 rows affected once the batch is full. No gap, no race.

### 3. Alternative — explicit row locking (`SELECT ... FOR UPDATE`)

```sql
BEGIN;

SELECT seats_filled, max_seats FROM courses
WHERE name = 'JEE Master Course' FOR UPDATE;   -- locks the row

-- application checks seats_filled < max_seats in code, then:

UPDATE courses SET seats_filled = seats_filled + 1 WHERE name = 'JEE Master Course';
INSERT INTO enrollment (id, course_name, enrollment_date, phone)
VALUES (101, 'JEE Master Course', CURRENT_DATE, '9876543299');

COMMIT;
```

`FOR UPDATE` makes the second transaction's `SELECT` **block** until the first commits or rolls back, so it always reads the true, up-to-date seat count before deciding.

### 4. Heavier alternative — `SERIALIZABLE` isolation

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

PostgreSQL detects the conflict between the two transactions itself and aborts one with a `serialization_failure`, which the application then retries. This is the strongest, most "textbook" isolation guarantee — but it means writing retry logic, and a popular last seat can cause a lot of aborts under heavy contention. Options 2 and 3 scale better for this specific hot-row case.

### 5. Defense in depth — a trigger, done correctly

A tempting "quick fix" is a trigger that recounts enrollments before every insert:

```sql
-- ⚠️ this naive version still has the race condition:
CREATE OR REPLACE FUNCTION check_seat_limit() RETURNS TRIGGER AS $$
BEGIN
  IF (SELECT COUNT(*) FROM enrollment WHERE course_name = NEW.course_name)
      >= (SELECT max_seats FROM courses WHERE name = NEW.course_name) THEN
    RAISE EXCEPTION 'Batch % is full', NEW.course_name;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

This looks like a fix, but on its own it isn't: two concurrent inserts can both run this `COUNT(*)` check before either commits, both see 49, and both pass. **A trigger only closes the gap if it also takes a lock** — by locking the parent `courses` row first:

```sql
CREATE OR REPLACE FUNCTION check_seat_limit() RETURNS TRIGGER AS $$
BEGIN
  PERFORM 1 FROM courses WHERE name = NEW.course_name FOR UPDATE;  -- serializes concurrent inserts
  IF (SELECT COUNT(*) FROM enrollment WHERE course_name = NEW.course_name)
      >= (SELECT max_seats FROM courses WHERE name = NEW.course_name) THEN
    RAISE EXCEPTION 'Batch % is full', NEW.course_name;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

Paired with the `max_seats` column, this is a good **safety net** to keep alongside option 2 — in case some other code path ever inserts into `enrollment` directly and skips the counter update.

---

## 🧪 How to Prove It (Two-Session Test)

Watch the locking happen using two `psql` terminals side by side:

**Session 1:**
```sql
BEGIN;
UPDATE courses SET seats_filled = seats_filled + 1
WHERE name = 'JEE Master Course' AND seats_filled < max_seats
RETURNING seats_filled;
-- don't COMMIT yet
```

**Session 2 (run immediately after):**
```sql
UPDATE courses SET seats_filled = seats_filled + 1
WHERE name = 'JEE Master Course' AND seats_filled < max_seats
RETURNING seats_filled;
-- hangs here, waiting on Session 1's row lock
```

Now go back to **Session 1** and run `COMMIT;`. Session 2 immediately unblocks and re-evaluates its `WHERE` clause against the new value — if that was the last seat, Session 2 now correctly returns **0 rows**.

---

## 📋 Summary

| Risk | Root Cause | Fix |
|---|---|---|
| Overbooking the last seat | Check-then-insert done as two separate, unlocked steps | Atomic `UPDATE ... WHERE ... RETURNING` (§2) or `SELECT ... FOR UPDATE` (§3) |
| App-level bugs bypassing the check | No DB-level backstop | Locking trigger on `enrollment` (§5) |
| Need the strongest possible guarantee | Default `READ COMMITTED` allows this anomaly | `SERIALIZABLE` isolation + retry logic (§4) |

---

