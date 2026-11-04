# Experiment 4 — Triggers, stored procedures and transaction management (ACID)
Platform: MySQL 8. Slips: **4A Electricity billing, 4B Online wallet, 4C Exam seat registration**.

> In the `mysql` command-line client use `DELIMITER $$ ... $$ DELIMITER ;` around triggers/procedures/functions (MySQL Workbench handles this itself – remove the DELIMITER lines there).
> If you get error 1418 when creating a function, run once as root: `SET GLOBAL log_bin_trust_function_creators = 1;`

---
# Slip 4A — Electricity billing

## Q1. Create tables, insert consumers
```sql
CREATE DATABASE IF NOT EXISTS elecDB;  USE elecDB;

CREATE TABLE Consumer (
  consumer_id INT PRIMARY KEY,
  name        VARCHAR(40),
  category    VARCHAR(15)
);
CREATE TABLE Bill (
  bill_id     INT AUTO_INCREMENT PRIMARY KEY,
  consumer_id INT,
  units       INT,
  amount      DECIMAL(10,2),
  bill_month  VARCHAR(10),
  FOREIGN KEY (consumer_id) REFERENCES Consumer(consumer_id)
);
CREATE TABLE Bill_Log (
  log_id      INT AUTO_INCREMENT PRIMARY KEY,
  consumer_id INT,
  amount      DECIMAL(10,2),
  log_time    DATETIME DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO Consumer VALUES (501,'Ramesh','Residential'),(502,'Om Traders','Commercial'),(503,'Sunita','Residential');
```

## Q3. BEFORE INSERT trigger – block negative units (created first so it runs first)
```sql
DELIMITER $$
CREATE TRIGGER trg_bill_check
BEFORE INSERT ON Bill
FOR EACH ROW
BEGIN
  IF NEW.units < 0 THEN
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Units cannot be negative';
  END IF;
END$$
DELIMITER ;
```

## Q2. BEFORE INSERT trigger – compute amount
```sql
DELIMITER $$
CREATE TRIGGER trg_bill_amount
BEFORE INSERT ON Bill
FOR EACH ROW FOLLOWS trg_bill_check
BEGIN
  DECLARE cat VARCHAR(15);
  SELECT category INTO cat FROM Consumer WHERE consumer_id = NEW.consumer_id;
  IF cat = 'Residential' THEN
    SET NEW.amount = NEW.units * 6;
  ELSE
    SET NEW.amount = NEW.units * 9;
  END IF;
END$$
DELIMITER ;
```

## Q4. AFTER INSERT trigger – copy to Bill_Log
```sql
DELIMITER $$
CREATE TRIGGER trg_bill_log
AFTER INSERT ON Bill
FOR EACH ROW
BEGIN
  INSERT INTO Bill_Log(consumer_id, amount) VALUES (NEW.consumer_id, NEW.amount);
END$$
DELIMITER ;
```

## Q5. Insert bills and display
```sql
INSERT INTO Bill(consumer_id, units, bill_month) VALUES
 (501, 120, 'Oct-2026'),     -- 720
 (502, 300, 'Oct-2026'),     -- 2700
 (503,  80, 'Oct-2026');     -- 480
SELECT * FROM Bill;
SELECT * FROM Bill_Log;

-- test the blocking trigger
INSERT INTO Bill(consumer_id, units, bill_month) VALUES (501, -5, 'Oct-2026');
-- ERROR 1644 (45000): Units cannot be negative
```

## Q6. Procedure add_bill
```sql
DELIMITER $$
CREATE PROCEDURE add_bill(IN cid INT, IN u INT, IN m VARCHAR(10))
BEGIN
  INSERT INTO Bill(consumer_id, units, amount, bill_month) VALUES (cid, u, 0, m);  -- trigger recomputes amount
END$$
DELIMITER ;

CALL add_bill(501, 100, 'Nov-2026');
SELECT * FROM Bill;
```

## Q7. Function total_paid
```sql
DELIMITER $$
CREATE FUNCTION total_paid(cid INT) RETURNS DECIMAL(12,2)
READS SQL DATA
BEGIN
  DECLARE t DECIMAL(12,2);
  SELECT COALESCE(SUM(amount), 0) INTO t FROM Bill WHERE consumer_id = cid;
  RETURN t;
END$$
DELIMITER ;

SELECT name, total_paid(consumer_id) AS total FROM Consumer;   -- Ramesh 1320, Om Traders 2700, Sunita 480
```

## Q8. Procedure with OUT parameter
```sql
DELIMITER $$
CREATE PROCEDURE category_total(IN cat VARCHAR(15), OUT total DECIMAL(12,2))
BEGIN
  SELECT COALESCE(SUM(b.amount), 0) INTO total
  FROM Bill b JOIN Consumer c ON b.consumer_id = c.consumer_id
  WHERE c.category = cat;
END$$
DELIMITER ;

CALL category_total('Residential', @t);
SELECT @t AS residential_total;           -- 1800
```

## Q9. Transaction + ROLLBACK
```sql
SELECT COUNT(*) FROM Bill;                 -- note the count (4)
START TRANSACTION;
  INSERT INTO Bill(consumer_id, units, bill_month) VALUES (501, 50, 'Dec-2026');
  INSERT INTO Bill(consumer_id, units, bill_month) VALUES (502, 70, 'Dec-2026');
  SELECT * FROM Bill;                      -- 2 extra rows visible inside the transaction
ROLLBACK;
SELECT * FROM Bill;                        -- back to 4 rows
SELECT * FROM Bill_Log;                    -- log rows also rolled back (same transaction)
```

## Q10. SAVEPOINT
```sql
START TRANSACTION;
  INSERT INTO Bill(consumer_id, units, bill_month) VALUES (503, 40, 'Dec-2026');   -- first insert
  SAVEPOINT sp1;
  INSERT INTO Bill(consumer_id, units, bill_month) VALUES (502, 90, 'Dec-2026');   -- second insert
  ROLLBACK TO SAVEPOINT sp1;               -- undo only the second insert
COMMIT;
SELECT * FROM Bill;                        -- Sunita's Dec bill is kept, Om Traders' is not
```

---
# Slip 4B — Online wallet

## Q1. Create and insert
```sql
CREATE DATABASE IF NOT EXISTS walletDB;  USE walletDB;

CREATE TABLE Wallet (
  wallet_id INT PRIMARY KEY,
  owner     VARCHAR(30),
  balance   DECIMAL(10,2)
);
CREATE TABLE Wallet_Txn (
  txn_id    INT AUTO_INCREMENT PRIMARY KEY,
  wallet_id INT,
  txn_type  VARCHAR(10),
  amount    DECIMAL(10,2),
  txn_time  DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (wallet_id) REFERENCES Wallet(wallet_id)
);
INSERT INTO Wallet VALUES (1,'Priya',5000),(2,'Sameer',1200),(3,'Tejal',300);
```

## Q2. BEFORE UPDATE – balance must not go below 0
```sql
DELIMITER $$
CREATE TRIGGER trg_wallet_check
BEFORE UPDATE ON Wallet
FOR EACH ROW
BEGIN
  IF NEW.balance < 0 THEN
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Insufficient balance';
  END IF;
END$$
DELIMITER ;
```

## Q3. AFTER UPDATE – record Debit/Credit
```sql
DELIMITER $$
CREATE TRIGGER trg_wallet_log
AFTER UPDATE ON Wallet
FOR EACH ROW
BEGIN
  IF NEW.balance < OLD.balance THEN
    INSERT INTO Wallet_Txn(wallet_id, txn_type, amount) VALUES (NEW.wallet_id, 'Debit',  OLD.balance - NEW.balance);
  ELSEIF NEW.balance > OLD.balance THEN
    INSERT INTO Wallet_Txn(wallet_id, txn_type, amount) VALUES (NEW.wallet_id, 'Credit', NEW.balance - OLD.balance);
  END IF;
END$$
DELIMITER ;
```

## Q4. Procedure add_money
```sql
DELIMITER $$
CREATE PROCEDURE add_money(IN wid INT, IN amt DECIMAL(10,2))
BEGIN
  UPDATE Wallet SET balance = balance + amt WHERE wallet_id = wid;
END$$
DELIMITER ;

CALL add_money(2, 200);          -- Sameer 1200 → 1400 (logged as Credit)
```
(The demo uses wallet 2 so that Tejal keeps 300, which is what makes `pay(3, 1, 500)` fail in Q7.)

## Q5. Procedure pay (transaction)
```sql
DELIMITER $$
CREATE PROCEDURE pay(IN from_wid INT, IN to_wid INT, IN amt DECIMAL(10,2))
BEGIN
  DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
    ROLLBACK;
    SELECT 'Payment failed - transaction rolled back' AS status;
  END;

  START TRANSACTION;
    UPDATE Wallet SET balance = balance - amt WHERE wallet_id = from_wid;   -- trigger blocks if < 0
    UPDATE Wallet SET balance = balance + amt WHERE wallet_id = to_wid;
  COMMIT;
  SELECT 'Payment successful' AS status;
END$$
DELIMITER ;
```

## Q6. Call pay(1, 2, 1000)
```sql
CALL pay(1, 2, 1000);
SELECT * FROM Wallet;        -- Priya 4000, Sameer 2400, Tejal 300
SELECT * FROM Wallet_Txn;    -- Debit 1000 (wallet 1), Credit 1000 (wallet 2)
```

## Q7. Call pay(3, 1, 500) – atomicity
```sql
SELECT * FROM Wallet;                -- note Tejal's and Priya's balances
CALL pay(3, 1, 500);                 -- Tejal does not have 500 → trigger raises error → handler rolls back
SELECT * FROM Wallet;                -- identical balances: nothing was changed
SELECT * FROM Wallet_Txn;            -- no new rows
```
Atomicity: either both updates (debit + credit) happen or neither.

## Q8. Function get_balance
```sql
DELIMITER $$
CREATE FUNCTION get_balance(wid INT) RETURNS DECIMAL(10,2)
READS SQL DATA
BEGIN
  DECLARE b DECIMAL(10,2);
  SELECT balance INTO b FROM Wallet WHERE wallet_id = wid;
  RETURN b;
END$$
DELIMITER ;

SELECT owner, get_balance(wallet_id) AS bal FROM Wallet;
SELECT get_balance(1) AS priya_balance;
```

## Q9. Isolation – two sessions
```sql
-- Session 1                                  -- Session 2
START TRANSACTION;
UPDATE Wallet SET balance = balance + 100
 WHERE wallet_id = 2;                         -- (no COMMIT yet)
                                              SELECT balance FROM Wallet WHERE wallet_id = 2;
                                              -- shows the OLD value (REPEATABLE READ / READ COMMITTED:
                                              -- uncommitted change is invisible)
COMMIT;
                                              SELECT balance FROM Wallet WHERE wallet_id = 2;
                                              -- (READ COMMITTED) now shows the new value
```
In Session 2 an `UPDATE` of the same row would block until Session 1 commits/rolls back (row lock).

## Q10. Drop the AFTER UPDATE trigger
```sql
DROP TRIGGER trg_wallet_log;
SELECT COUNT(*) FROM Wallet_Txn;           -- note the count
CALL pay(1, 3, 100);
SELECT COUNT(*) FROM Wallet_Txn;           -- unchanged: payments are no longer logged
SELECT * FROM Wallet;                      -- but balances did change
```

---
# Slip 4C — Exam seat registration

## Q1. Create and insert
```sql
CREATE DATABASE IF NOT EXISTS examDB;  USE examDB;

CREATE TABLE Exam (
  exam_id    VARCHAR(5) PRIMARY KEY,
  subject    VARCHAR(30),
  seats_left INT
);
CREATE TABLE Registration (
  reg_id  INT AUTO_INCREMENT PRIMARY KEY,
  exam_id VARCHAR(5),
  student VARCHAR(40),
  status  VARCHAR(10) DEFAULT 'Active',
  FOREIGN KEY (exam_id) REFERENCES Exam(exam_id)
);
INSERT INTO Exam VALUES ('EX1','DBMS',3),('EX2','Python',40);
```

## Q3. BEFORE INSERT – block when no seats
```sql
DELIMITER $$
CREATE TRIGGER trg_reg_check
BEFORE INSERT ON Registration
FOR EACH ROW
BEGIN
  DECLARE s INT;
  SELECT seats_left INTO s FROM Exam WHERE exam_id = NEW.exam_id;
  IF s <= 0 THEN
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'No seats left for this exam';
  END IF;
END$$
DELIMITER ;
```

## Q2. AFTER INSERT – reduce seats_left
```sql
DELIMITER $$
CREATE TRIGGER trg_reg_seat
AFTER INSERT ON Registration
FOR EACH ROW
BEGIN
  UPDATE Exam SET seats_left = seats_left - 1 WHERE exam_id = NEW.exam_id;
END$$
DELIMITER ;
```

## Q4. Register three students, fourth fails
```sql
INSERT INTO Registration(exam_id, student) VALUES ('EX1','Asha');
INSERT INTO Registration(exam_id, student) VALUES ('EX1','Bipin');
INSERT INTO Registration(exam_id, student) VALUES ('EX1','Chinmay');
SELECT * FROM Exam;                         -- EX1 seats_left = 0

INSERT INTO Registration(exam_id, student) VALUES ('EX1','Disha');
-- ERROR 1644 (45000): No seats left for this exam
```

## Q5. AFTER UPDATE – restore seat on withdrawal
```sql
DELIMITER $$
CREATE TRIGGER trg_reg_withdraw
AFTER UPDATE ON Registration
FOR EACH ROW
BEGIN
  IF NEW.status = 'Withdrawn' AND OLD.status <> 'Withdrawn' THEN
    UPDATE Exam SET seats_left = seats_left + 1 WHERE exam_id = NEW.exam_id;
  END IF;
END$$
DELIMITER ;
```

## Q6. Withdraw one student
```sql
UPDATE Registration SET status = 'Withdrawn' WHERE student = 'Bipin' AND exam_id = 'EX1';
SELECT * FROM Exam;                         -- EX1 seats_left = 1
SELECT * FROM Registration;
```

## Q7. Procedure register_student (as a transaction)
```sql
DELIMITER $$
CREATE PROCEDURE register_student(IN eid VARCHAR(5), IN sname VARCHAR(40))
BEGIN
  DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
    ROLLBACK;
    SELECT 'Registration failed (no seats or error)' AS msg;
  END;

  START TRANSACTION;
    INSERT INTO Registration(exam_id, student) VALUES (eid, sname);
  COMMIT;
  SELECT 'Registered successfully' AS msg;
END$$
DELIMITER ;

CALL register_student('EX1', 'Disha');   -- succeeds (1 seat free after the withdrawal)
CALL register_student('EX1', 'Esha');    -- fails: no seats
```

## Q8. Function seats_filled
```sql
DELIMITER $$
CREATE FUNCTION seats_filled(eid VARCHAR(5)) RETURNS INT
READS SQL DATA
BEGIN
  DECLARE n INT;
  SELECT COUNT(*) INTO n FROM Registration WHERE exam_id = eid AND status = 'Active';
  RETURN n;
END$$
DELIMITER ;

SELECT exam_id, seats_filled(exam_id) AS active_registrations FROM Exam;
```

## Q9. Transaction with SAVEPOINT
```sql
START TRANSACTION;
  INSERT INTO Registration(exam_id, student) VALUES ('EX2','Isha');
  SAVEPOINT after_first;
  INSERT INTO Registration(exam_id, student) VALUES ('EX2','Jay');
  ROLLBACK TO SAVEPOINT after_first;        -- Jay's registration (and its seat deduction) is undone
COMMIT;
SELECT * FROM Registration WHERE exam_id = 'EX2';   -- only Isha
SELECT * FROM Exam;                                  -- EX2 seats_left = 39
```

## Q10. SHOW TRIGGERS and ACID
```sql
SHOW TRIGGERS;
```
Lists trg_reg_check (BEFORE INSERT), trg_reg_seat (AFTER INSERT), trg_reg_withdraw (AFTER UPDATE).

| Property | How it was shown |
|---|---|
| **Atomicity** | In Q9 the second insert and its trigger side-effect (seat deduction) were rolled back together; in `register_student` the handler rolls back the whole transaction on error. |
| **Consistency** | Triggers keep `seats_left` equal to capacity minus active registrations and never allow it to go below 0 (the 4th registration is rejected). |
| **Isolation** | Another session cannot see uncommitted registrations/seat counts; concurrent updates to the same Exam row are serialized by row locks. |
| **Durability** | After `COMMIT` the registrations and updated seats persist (InnoDB redo log), even if the server restarts. |

---
### Viva quick points
- Trigger = stored program fired automatically on INSERT/UPDATE/DELETE (BEFORE/AFTER); `NEW` and `OLD` give the new/old row values. MySQL has no triggers for SELECT.
- Procedure (CALL, can have IN/OUT/INOUT, can run transactions) vs function (returns one value, usable inside SELECT, cannot COMMIT/ROLLBACK).
- ACID = Atomicity, Consistency, Isolation, Durability. DDL statements auto-commit; use InnoDB for transactions.
