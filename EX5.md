# Experiment 5 — Concurrency control and log-based recovery
Platform: MySQL 8 (5A, 5B – two sessions) and Python 3 (5C). Slips: **5A Train seat booking, 5B Warehouse stock, 5C Log-based recovery**.

For 5A/5B open **two terminals** (`mysql -u root -p`), label them **S1** and **S2**, and `USE` the same database in both. Run the numbered statements in the order shown by the timeline.

---
# Slip 5A — Train seat booking (isolation levels and locking)

## Q1. Create table (InnoDB), insert data, open two sessions
```sql
CREATE DATABASE IF NOT EXISTS trainDB;  USE trainDB;
CREATE TABLE Train_Seat (
  seat_no VARCHAR(5) PRIMARY KEY,
  status  VARCHAR(10)
) ENGINE=InnoDB;
INSERT INTO Train_Seat VALUES ('S1','Free'),('S2','Free'),('S3','Booked'),('S4','Free');
```

## Q2. Show isolation level (both sessions)
```sql
SELECT @@transaction_isolation;      -- default: REPEATABLE-READ
```

## Q3. Dirty read (READ UNCOMMITTED)
| Step | S1 | S2 |
|---|---|---|
| 1 | | `SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;` |
| 2 | `START TRANSACTION;` `UPDATE Train_Seat SET status='Booked' WHERE seat_no='S1';` (no commit) | |
| 3 | | `SELECT * FROM Train_Seat WHERE seat_no='S1';` → **Booked** (dirty read) |
| 4 | `ROLLBACK;` | |
| 5 | | `SELECT * FROM Train_Seat WHERE seat_no='S1';` → **Free** (S2 had read a value that never existed) |

## Q4. READ COMMITTED – dirty read prevented
| Step | S1 | S2 |
|---|---|---|
| 1 | | `SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;` |
| 2 | `START TRANSACTION;` `UPDATE Train_Seat SET status='Booked' WHERE seat_no='S1';` | |
| 3 | | `SELECT * FROM Train_Seat WHERE seat_no='S1';` → **Free** (uncommitted change not visible) |
| 4 | `ROLLBACK;` | |

## Q5. Non-repeatable read (READ COMMITTED)
| Step | S1 | S2 |
|---|---|---|
| 1 | | `SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;` `START TRANSACTION;` |
| 2 | | `SELECT status FROM Train_Seat WHERE seat_no='S2';` → **Free** |
| 3 | `START TRANSACTION;` `UPDATE Train_Seat SET status='Booked' WHERE seat_no='S2';` `COMMIT;` | |
| 4 | | `SELECT status FROM Train_Seat WHERE seat_no='S2';` → **Booked** (same query, different result = non-repeatable read) |
| 5 | | `COMMIT;` |

Reset first: `UPDATE Train_Seat SET status='Free' WHERE seat_no='S2';`

## Q6. REPEATABLE READ – compare
Reset S2's seat to Free, then repeat Q5 with S2 using `SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;` before `START TRANSACTION`.

| Step | S2 result |
|---|---|
| First SELECT | Free |
| After S1 commits the update, second SELECT (same transaction) | **Free** (snapshot is fixed – repeatable) |
| After S2 `COMMIT;` and a new SELECT | Booked |

Comparison: READ COMMITTED gives a new snapshot for every statement (non-repeatable reads possible); REPEATABLE READ uses one snapshot for the whole transaction.

## Q7. Prevent double booking of S4 with SELECT … FOR UPDATE
| Step | S1 | S2 |
|---|---|---|
| 1 | `START TRANSACTION;` `SELECT * FROM Train_Seat WHERE seat_no='S4' FOR UPDATE;` → Free (row locked) | |
| 2 | | `START TRANSACTION;` `SELECT * FROM Train_Seat WHERE seat_no='S4' FOR UPDATE;` → **waits** (blocked) |
| 3 | `UPDATE Train_Seat SET status='Booked' WHERE seat_no='S4';` `COMMIT;` | |
| 4 | | the SELECT now returns **Booked** → S2 sees the seat is gone and does not book it; `ROLLBACK;` |

Only one user gets the seat; no double booking.

## Q8. LOCK TABLES
```sql
-- S1
LOCK TABLES Train_Seat WRITE;
-- S2
SELECT * FROM Train_Seat;        -- waits (hangs)
-- S1
UNLOCK TABLES;
-- S2: the SELECT now completes
```

## Q9. Lock wait timeout
```sql
-- S1
START TRANSACTION;
UPDATE Train_Seat SET status='Booked' WHERE seat_no='S1';     -- leave it open
-- S2
START TRANSACTION;
UPDATE Train_Seat SET status='Free' WHERE seat_no='S1';       -- waits ~50 s
-- ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout';               -- 50 (seconds)
-- S1: ROLLBACK;
```
(To see it faster: `SET SESSION innodb_lock_wait_timeout = 5;` in S2.)

## Q10. Isolation level × anomaly
| Isolation level | Dirty read | Non-repeatable read | Phantom read |
|---|---|---|---|
| READ UNCOMMITTED | Occurred | Occurred | Occurred |
| READ COMMITTED | Prevented | Occurred | Occurred |
| REPEATABLE READ | Prevented | Prevented | Possible by the SQL standard; InnoDB largely prevents it (consistent reads use the snapshot, next-key locks for locking reads) |
| SERIALIZABLE | Prevented | Prevented | Prevented |

---
# Slip 5B — Warehouse stock (lost update and deadlock)

## Q1. Setup
```sql
CREATE DATABASE IF NOT EXISTS stockDB;  USE stockDB;
CREATE TABLE Stock (
  item_id   INT PRIMARY KEY,
  item_name VARCHAR(30),
  qty       INT
) ENGINE=InnoDB;
INSERT INTO Stock VALUES (1,'Cement bags',500),(2,'Steel rods',300);

-- in BOTH sessions
SET autocommit = 0;
```

## Q2. Lost update
| Time | S1 | S2 |
|---|---|---|
| t1 | `SELECT qty FROM Stock WHERE item_id=1;` → 500 | |
| t2 | | `SELECT qty FROM Stock WHERE item_id=1;` → 500 |
| t3 | `UPDATE Stock SET qty=400 WHERE item_id=1;` (500−100 computed by the application) | |
| t4 | `COMMIT;` | |
| t5 | | `UPDATE Stock SET qty=450 WHERE item_id=1;` (500−50) |
| t6 | | `COMMIT;` |
| t7 | `SELECT qty FROM Stock WHERE item_id=1;` → **450** | |

Expected final value is 500 − 100 − 50 = **350**, but S2 overwrote S1's update → **lost update** (S1's −100 is lost).

## Q3. Schedule (time-line)
Same as the table in Q2: R1(Q) → R2(Q) → W1(Q=400) → C1 → W2(Q=450) → C2. Because both read before either wrote, the schedule is not serializable.

## Q4. Fix with SELECT … FOR UPDATE
```sql
UPDATE Stock SET qty = 500 WHERE item_id = 1;  COMMIT;     -- reset
```
| Time | S1 | S2 |
|---|---|---|
| t1 | `SELECT qty FROM Stock WHERE item_id=1 FOR UPDATE;` → 500 (X-lock) | |
| t2 | | `SELECT qty FROM Stock WHERE item_id=1 FOR UPDATE;` → **waits** |
| t3 | `UPDATE Stock SET qty=400 WHERE item_id=1;` `COMMIT;` | |
| t4 | | gets **400**; `UPDATE Stock SET qty=350 WHERE item_id=1;` `COMMIT;` |
| t5 | `SELECT qty FROM Stock WHERE item_id=1;` → **350** ✔ | |

## Q5. Relative update
```sql
-- S1
UPDATE Stock SET qty = qty - 100 WHERE item_id = 1;   -- takes X-lock
-- S2
UPDATE Stock SET qty = qty - 50  WHERE item_id = 1;   -- waits
-- S1
COMMIT;
-- S2 continues
COMMIT;
SELECT qty FROM Stock WHERE item_id = 1;              -- 350
```
Comparison: the single-statement update reads and writes atomically under one lock, so no lost update even without `FOR UPDATE`.

## Q6. Deadlock
```sql
UPDATE Stock SET qty = 500 WHERE item_id = 1;
UPDATE Stock SET qty = 300 WHERE item_id = 2;  COMMIT;     -- reset
```
| Time | S1 | S2 |
|---|---|---|
| t1 | `UPDATE Stock SET qty=qty-10 WHERE item_id=1;` (locks item 1) | |
| t2 | | `UPDATE Stock SET qty=qty-20 WHERE item_id=2;` (locks item 2) |
| t3 | `UPDATE Stock SET qty=qty-10 WHERE item_id=2;` → **waits for S2** | |
| t4 | | `UPDATE Stock SET qty=qty-20 WHERE item_id=1;` → waits for S1 → cycle |
| | | `ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction` |

## Q7. Victim and final quantities
InnoDB detects the cycle and rolls back one transaction (the one with less work – here S2 received error 1213). View details:
```sql
SHOW ENGINE INNODB STATUS\G      -- section "LATEST DETECTED DEADLOCK"
```
Then S1's pending update proceeds; run `COMMIT;` in S1 and check:
```sql
SELECT * FROM Stock;             -- item 1 = 490, item 2 = 290 (only S1's changes survive)
```
S2's changes were rolled back (it can be retried).

## Q8. Wait-for graph
```
 T1 ──waits for item 2 (held by T2)──▶ T2
 ▲                                      │
 └──waits for item 1 (held by T1)───────┘
```
Nodes = transactions, edge Ti → Tj means Ti waits for a lock held by Tj. A **cycle (T1 → T2 → T1)** means deadlock.

## Q9. Same order – no deadlock
```sql
-- S1: UPDATE item 1; then item 2.
-- S2: UPDATE item 1 (waits for S1) ...
-- S1: UPDATE item 2; COMMIT;
-- S2: now gets item 1, then item 2; COMMIT;   -- no cycle, no deadlock
```
Rule: always acquire locks in a fixed global order.

## Q10. Two-phase locking and the lost update
2PL has a **growing phase** (acquire locks, no release) and a **shrinking phase** (release only after). With strict 2PL, the exclusive lock taken by `SELECT … FOR UPDATE`/`UPDATE` is held until COMMIT, so a second transaction cannot read-then-write the same item in between: it waits and then sees the committed value. The schedule becomes equivalent to a serial one (S1 then S2), so the lost update cannot occur. (2PL can cause deadlock, which is why detection/ordering is needed.)

---
# Slip 5C — Log-based recovery program (Python)

Log (immediate-modification scheme, undo/redo with checkpoint), initial P=500, Q=800, R=300, S=1000.

**Assumption for the program:** the checkpoint flushed all updates written *before* it to disk, while updates *after* the checkpoint were still in memory when the system crashed. So the database on disk at crash time is: P=450, Q=900, R=300, S=1000.

## Q1. log.txt
```text
<T1, start>
<T1, P, 500, 450>
<T1, commit>
<T2, start>
<T2, Q, 800, 900>
<checkpoint {T2}>
<T3, start>
<T3, R, 300, 250>
<T3, commit>
<T2, S, 1000, 1100>
```

## Q2–Q8. Program `recovery.py`
```python
import re
import sys

INITIAL = {"P": 500, "Q": 800, "R": 300, "S": 1000}


def read_log(fname):
    """Q2: read the log file and return a list of tuples."""
    records = []
    with open(fname) as f:
        for line in f:
            line = line.strip()
            if not line:
                continue
            body = line.strip("<>").strip()
            if body.startswith("checkpoint"):
                records.append(("checkpoint", re.findall(r"T\d+", body)))
            else:
                p = [x.strip() for x in body.split(",")]
                if p[1] in ("start", "commit", "abort"):
                    records.append((p[1], p[0]))
                else:                                   # <T, item, old, new>
                    records.append(("update", p[0], p[1], int(p[2]), int(p[3])))
    return records


def main(fname):
    log = read_log(fname)

    print("=== Q2: Log records ===")
    for i, r in enumerate(log, 1):
        print(i, r)

    # Q3: committed / uncommitted
    started = [r[1] for r in log if r[0] == "start"]
    committed = [r[1] for r in log if r[0] == "commit"]
    uncommitted = [t for t in started if t not in committed]
    print("\n=== Q3 ===")
    print("Committed  :", committed)
    print("Uncommitted:", uncommitted)

    # Q4: last checkpoint and active transactions
    cp_idx = None
    for i, r in enumerate(log):
        if r[0] == "checkpoint":
            cp_idx = i
    active_at_cp = log[cp_idx][1] if cp_idx is not None else []
    print("\n=== Q4 ===")
    print("Last checkpoint at log record:", None if cp_idx is None else cp_idx + 1)
    print("Active at checkpoint:", active_at_cp)

    # Q5: UNDO and REDO lists
    scan_from = cp_idx + 1 if cp_idx is not None else 0
    started_after_cp = [r[1] for r in log[scan_from:] if r[0] == "start"]
    undo_list = uncommitted
    redo_list = [t for t in committed if t in active_at_cp or t in started_after_cp]
    print("\n=== Q5 ===")
    print("UNDO list:", undo_list)
    print("REDO list:", redo_list)

    # database state on disk at crash time (changes before the checkpoint were flushed)
    db = dict(INITIAL)
    for r in log[:scan_from]:
        if r[0] == "update":
            db[r[2]] = r[4]
    print("\nDatabase on disk at crash:", db)

    # Q6: UNDO – scan backwards
    print("\n=== Q6: UNDO (backward scan) ===")
    for r in reversed(log):
        if r[0] == "update" and r[1] in undo_list:
            _, t, item, old, new = r
            db[item] = old
            print(f"UNDO {t}: {item} restored to {old}")
    print("State after UNDO:", db)

    # Q7: REDO – scan forwards from the checkpoint
    print("\n=== Q7: REDO (forward scan after checkpoint) ===")
    for r in log[scan_from:]:
        if r[0] == "update" and r[1] in redo_list:
            _, t, item, old, new = r
            db[item] = new
            print(f"REDO {t}: {item} set to {new}")
    print("State after REDO:", db)

    # Q8: final values
    print("\n=== Q8: Final values ===")
    for k in sorted(db):
        print(f"{k} = {db[k]}")


if __name__ == "__main__":
    main(sys.argv[1] if len(sys.argv) > 1 else "log.txt")
```
Run: `python recovery.py log.txt`

### Expected output (key parts)
```
Committed  : ['T1', 'T3']
Uncommitted: ['T2']
Last checkpoint at log record: 6
Active at checkpoint: ['T2']
UNDO list: ['T2']
REDO list: ['T3']
Database on disk at crash: {'P': 450, 'Q': 900, 'R': 300, 'S': 1000}
UNDO T2: S restored to 1000
UNDO T2: Q restored to 800
REDO T3: R set to 250
Final values: P = 450, Q = 800, R = 250, S = 1000
```

## Q9. Why T1 needs no action
T1 started and committed **before** the checkpoint. At the checkpoint all its changes (P = 450) were forced to disk, so they are already durable and T1 is neither in the UNDO nor in the REDO list. Only transactions active at the checkpoint or started after it need examination.

## Q10. Add `<T2, commit>` before the crash
```python
# make log2.txt = log.txt + commit record for T2
lines = open("log.txt").read().splitlines()
open("log2.txt", "w").write("\n".join(lines + ["<T2, commit>"]) + "\n")
```
Run: `python recovery.py log2.txt`

| | Original log | With `<T2, commit>` |
|---|---|---|
| Committed | T1, T3 | T1, T3, T2 |
| Uncommitted | T2 | none |
| UNDO list | T2 | empty |
| REDO list | T3 | T3, T2 (T2 was active at the checkpoint and committed) |
| Final P, Q, R, S | 450, 800, 250, 1000 | 450, 900, 250, 1100 |

With T2 committed its updates must survive (durability), so S = 1100 is redone and Q = 900 stays; nothing is undone.

---
### Viva quick points
- Dirty read / non-repeatable read / phantom read; isolation levels RU, RC, RR, SERIALIZABLE; MySQL default is REPEATABLE READ.
- Lost update, deadlock (wait-for-graph cycle), strict 2PL, lock ordering.
- Log-based recovery: write-ahead logging, `<T, X, old, new>`, UNDO uses old value, REDO uses new value, checkpoints limit the log scanned.
