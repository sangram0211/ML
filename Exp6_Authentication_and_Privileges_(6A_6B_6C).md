# Experiment 6 — User authentication and access privileges
Platform: MySQL 8. Slips: **6A College ERP roles, 6B Bank staff privileges, 6C Application login module**.

Run the admin statements as `root`. To "log in as" a user, open a second terminal: `mysql -u <user> -p <database>`.
(Use `DELIMITER $$ … $$ DELIMITER ;` for procedures in the command-line client.)

---
# Slip 6A — College ERP roles

## Q1. Create table and data
```sql
CREATE DATABASE IF NOT EXISTS collegeDB;  USE collegeDB;
CREATE TABLE Student_Record (
  roll_no INT PRIMARY KEY,
  name    VARCHAR(40),
  dept    VARCHAR(10),
  cgpa    DECIMAL(3,1),
  fee_due INT
);
INSERT INTO Student_Record VALUES (1,'Aarti','AIML',8.2,25000),(2,'Bhushan','CSE',7.5,0),(3,'Chaitali','AIML',9.0,10000);
```

## Q2. Create roles
```sql
CREATE ROLE 'erp_admin', 'faculty', 'student_role';
```

## Q3. Grant privileges to roles
```sql
GRANT ALL PRIVILEGES ON collegeDB.Student_Record TO 'erp_admin';

GRANT SELECT ON collegeDB.Student_Record TO 'faculty';
GRANT UPDATE (cgpa) ON collegeDB.Student_Record TO 'faculty';       -- column-level

GRANT SELECT (roll_no, name, cgpa) ON collegeDB.Student_Record TO 'student_role';   -- column-level
```

## Q4. Create users, assign roles, default roles
```sql
CREATE USER 'admin1'@'localhost' IDENTIFIED BY 'Admin@123';
CREATE USER 'fac1'@'localhost'   IDENTIFIED BY 'Fac@1234';
CREATE USER 'stud1'@'localhost'  IDENTIFIED BY 'Stud@1234';

GRANT 'erp_admin'    TO 'admin1'@'localhost';
GRANT 'faculty'      TO 'fac1'@'localhost';
GRANT 'student_role' TO 'stud1'@'localhost';

SET DEFAULT ROLE 'erp_admin'    TO 'admin1'@'localhost';
SET DEFAULT ROLE 'faculty'      TO 'fac1'@'localhost';
SET DEFAULT ROLE 'student_role' TO 'stud1'@'localhost';
```
(The users also need access to the database; the column/table grants on the roles give that.)

## Q5. fac1 updates CGPA – success
```sql
-- mysql -u fac1 -p collegeDB
SELECT CURRENT_ROLE();
UPDATE Student_Record SET cgpa = 8.5 WHERE roll_no = 1;     -- Query OK, 1 row affected
SELECT * FROM Student_Record;
```

## Q6. fac1 updates fee_due – error
```sql
UPDATE Student_Record SET fee_due = 0 WHERE roll_no = 1;
-- ERROR 1143 (42000): UPDATE command denied to user 'fac1'@'localhost' for column 'fee_due' in table 'Student_Record'
```

## Q7. stud1 selects fee_due – column-level error
```sql
-- mysql -u stud1 -p collegeDB
SELECT roll_no, name, cgpa FROM Student_Record;     -- works
SELECT fee_due FROM Student_Record;
-- ERROR 1143 (42000): SELECT command denied to user 'stud1'@'localhost' for column 'fee_due' in table 'Student_Record'
SELECT * FROM Student_Record;                        -- also denied (includes fee_due, dept)
```

## Q8. SHOW GRANTS
```sql
SHOW GRANTS FOR 'admin1'@'localhost' USING 'erp_admin';
SHOW GRANTS FOR 'fac1'@'localhost'   USING 'faculty';
SHOW GRANTS FOR 'stud1'@'localhost'  USING 'student_role';
SHOW GRANTS FOR 'faculty';
```

## Q9. REVOKE UPDATE from faculty
```sql
REVOKE UPDATE (cgpa) ON collegeDB.Student_Record FROM 'faculty';
SHOW GRANTS FOR 'faculty';                 -- only SELECT remains
```
Now reconnect as `fac1` (role changes apply to new sessions; or run `SET ROLE 'faculty';`):
```sql
UPDATE Student_Record SET cgpa = 9.9 WHERE roll_no = 1;
-- ERROR 1142/1143 (42000): UPDATE command denied to user 'fac1'@'localhost' ...
```

## Q10. Lock account
```sql
ALTER USER 'stud1'@'localhost' ACCOUNT LOCK;
-- mysql -u stud1 -p
-- ERROR 3118 (HY000): Access denied for user 'stud1'@'localhost'. Account is locked.
-- unlock when needed: ALTER USER 'stud1'@'localhost' ACCOUNT UNLOCK;
```

---
# Slip 6B — Bank staff privileges

## Q1. Create table and data
```sql
CREATE DATABASE IF NOT EXISTS bankDB;  USE bankDB;
CREATE TABLE Account (
  acc_no   INT PRIMARY KEY,
  holder   VARCHAR(40),
  acc_type VARCHAR(10),
  balance  DECIMAL(12,2)
);
INSERT INTO Account VALUES (9001,'Rohit','Savings',40000),(9002,'Swati','Current',150000),(9003,'Vinay','Savings',7000);
```

## Q2. User with password expiry
```sql
CREATE USER 'teller'@'localhost' IDENTIFIED BY 'Teller@123' PASSWORD EXPIRE INTERVAL 30 DAY;
SELECT user, host, password_lifetime FROM mysql.user WHERE user = 'teller';   -- 30
```

## Q3. Grant SELECT and UPDATE(balance)
```sql
GRANT SELECT, UPDATE (balance) ON bankDB.Account TO 'teller'@'localhost';
SHOW GRANTS FOR 'teller'@'localhost';
```

## Q4. View and grant on the view only
```sql
CREATE VIEW savings_accounts AS
SELECT acc_no, holder, balance FROM Account WHERE acc_type = 'Savings';

CREATE USER 'auditor'@'localhost' IDENTIFIED BY 'Audit@123';
GRANT SELECT ON bankDB.savings_accounts TO 'auditor'@'localhost';
```

## Q5. auditor can read view, not table
```sql
-- mysql -u auditor -p bankDB
SELECT * FROM savings_accounts;     -- works (Rohit, Vinay)
SELECT * FROM Account;
-- ERROR 1142 (42000): SELECT command denied to user 'auditor'@'localhost' for table 'Account'
```

## Q6. Procedure deposit and EXECUTE grant
```sql
DELIMITER $$
CREATE PROCEDURE deposit(IN acc INT, IN amt DECIMAL(12,2))
BEGIN
  UPDATE Account SET balance = balance + amt WHERE acc_no = acc;
END$$
DELIMITER ;

GRANT EXECUTE ON PROCEDURE bankDB.deposit TO 'teller'@'localhost';
```

## Q7. teller calls deposit; DELETE fails
```sql
-- mysql -u teller -p bankDB      (first login forces nothing yet; the password has not expired)
CALL deposit(9001, 5000);
SELECT * FROM Account;               -- 9001 = 45000
DELETE FROM Account WHERE acc_no = 9003;
-- ERROR 1142 (42000): DELETE command denied to user 'teller'@'localhost' for table 'Account'
```

## Q8. WITH GRANT OPTION
```sql
CREATE USER 'branch_mgr'@'localhost' IDENTIFIED BY 'Mgr@1234';
CREATE USER 'clerk1'@'localhost'     IDENTIFIED BY 'Clerk@123';

GRANT SELECT, UPDATE ON bankDB.Account TO 'branch_mgr'@'localhost' WITH GRANT OPTION;

-- mysql -u branch_mgr -p
GRANT SELECT ON bankDB.Account TO 'clerk1'@'localhost';       -- allowed (branch_mgr holds SELECT + GRANT OPTION)
-- branch_mgr cannot grant DELETE – he does not hold it:
GRANT DELETE ON bankDB.Account TO 'clerk1'@'localhost';       -- ERROR 1142/1044
```

## Q9. REVOKE EXECUTE
```sql
REVOKE EXECUTE ON PROCEDURE bankDB.deposit FROM 'teller'@'localhost';
-- mysql -u teller -p bankDB
CALL deposit(9001, 100);
-- ERROR 1370 (42000): execute command denied to user 'teller'@'localhost' for routine 'bankDB.deposit'
```

## Q10. Change password and drop user
```sql
ALTER USER 'teller'@'localhost' IDENTIFIED BY 'NewTeller@456';
DROP USER 'auditor'@'localhost';
SELECT user, host FROM mysql.user WHERE user IN ('teller','auditor');   -- auditor is gone
```

---
# Slip 6C — Application login module (hashed passwords, lockout)

## Q1. Create tables
```sql
CREATE DATABASE IF NOT EXISTS appDB;  USE appDB;

CREATE TABLE App_User (
  user_id         INT AUTO_INCREMENT PRIMARY KEY,
  username        VARCHAR(30) UNIQUE NOT NULL,
  password_hash   CHAR(64) NOT NULL,
  role            VARCHAR(10),
  failed_attempts INT DEFAULT 0,
  is_locked       TINYINT DEFAULT 0
);
CREATE TABLE Login_Log (
  log_id     INT AUTO_INCREMENT PRIMARY KEY,
  username   VARCHAR(30),
  login_time DATETIME DEFAULT CURRENT_TIMESTAMP,
  status     VARCHAR(10)
);
```

## Q2. Insert users with SHA2 hash
```sql
INSERT INTO App_User(username, password_hash, role)
VALUES ('admin', SHA2('Admin@2026', 256), 'admin'),
       ('neha',  SHA2('Neha@2026', 256),  'staff');

SELECT username, password_hash, role FROM App_User;      -- 64-hex-character hashes, no plain text
```

## Q3 & Q4. Procedure register (duplicate username + password length check)
```sql
DELIMITER $$
CREATE PROCEDURE register(IN uname VARCHAR(30), IN pwd VARCHAR(50), IN urole VARCHAR(10))
BEGIN
  IF EXISTS (SELECT 1 FROM App_User WHERE username = uname) THEN
    SELECT 'Username already exists' AS msg;
  ELSEIF CHAR_LENGTH(pwd) < 8 THEN
    SELECT 'Password must be at least 8 characters' AS msg;
  ELSE
    INSERT INTO App_User(username, password_hash, role) VALUES (uname, SHA2(pwd, 256), urole);
    SELECT 'Registered successfully' AS msg;
  END IF;
END$$
DELIMITER ;

CALL register('neha', 'Another@123', 'staff');   -- Username already exists
CALL register('raj',  'abc123',      'staff');   -- Password must be at least 8 characters
CALL register('raj',  'Raj@2026',    'staff');   -- Registered successfully
```

## Q5, Q6 & Q7. Procedure login (hash compare, lockout after 3 failures, logging)
```sql
DELIMITER $$
CREATE PROCEDURE login(IN uname VARCHAR(30), IN pwd VARCHAR(50), OUT msg VARCHAR(60))
BEGIN
  DECLARE v_hash   CHAR(64) DEFAULT NULL;
  DECLARE v_locked TINYINT  DEFAULT 0;
  DECLARE v_fail   INT      DEFAULT 0;

  SELECT password_hash, is_locked, failed_attempts
    INTO v_hash, v_locked, v_fail
    FROM App_User WHERE username = uname;

  IF v_hash IS NULL THEN
    SET msg = 'User not found';
    INSERT INTO Login_Log(username, status) VALUES (uname, 'Failed');

  ELSEIF v_locked = 1 THEN
    SET msg = 'Account is locked';
    INSERT INTO Login_Log(username, status) VALUES (uname, 'Locked');

  ELSEIF v_hash = SHA2(pwd, 256) THEN
    UPDATE App_User SET failed_attempts = 0 WHERE username = uname;
    SET msg = 'Login successful';
    INSERT INTO Login_Log(username, status) VALUES (uname, 'Success');

  ELSE
    UPDATE App_User SET failed_attempts = failed_attempts + 1 WHERE username = uname;
    IF v_fail + 1 >= 3 THEN
      UPDATE App_User SET is_locked = 1 WHERE username = uname;
      SET msg = 'Wrong password - account locked after 3 failed attempts';
    ELSE
      SET msg = 'Invalid password';
    END IF;
    INSERT INTO Login_Log(username, status) VALUES (uname, 'Failed');
  END IF;
END$$
DELIMITER ;
```

## Q8. Demonstration for neha
```sql
CALL login('neha', 'Neha@2026', @m);  SELECT @m;     -- Login successful
CALL login('neha', 'wrong1', @m);     SELECT @m;     -- Invalid password            (failed_attempts = 1)
CALL login('neha', 'wrong2', @m);     SELECT @m;     -- Invalid password            (2)
CALL login('neha', 'wrong3', @m);     SELECT @m;     -- Wrong password - account locked after 3 failed attempts
CALL login('neha', 'Neha@2026', @m);  SELECT @m;     -- Account is locked (even with the right password)

SELECT username, failed_attempts, is_locked FROM App_User;
-- unlock for testing: UPDATE App_User SET is_locked=0, failed_attempts=0 WHERE username='neha';
```

## Q9. Procedure change_password
```sql
DELIMITER $$
CREATE PROCEDURE change_password(IN uname VARCHAR(30), IN old_pwd VARCHAR(50), IN new_pwd VARCHAR(50))
BEGIN
  IF NOT EXISTS (SELECT 1 FROM App_User
                 WHERE username = uname AND password_hash = SHA2(old_pwd, 256) AND is_locked = 0) THEN
    SELECT 'Old password incorrect or account locked' AS msg;
  ELSEIF CHAR_LENGTH(new_pwd) < 8 THEN
    SELECT 'New password must be at least 8 characters' AS msg;
  ELSE
    UPDATE App_User SET password_hash = SHA2(new_pwd, 256) WHERE username = uname;
    SELECT 'Password changed successfully' AS msg;
  END IF;
END$$
DELIMITER ;

CALL change_password('admin', 'Admin@2026', 'NewAdmin@2026');
CALL change_password('admin', 'wrongold', 'Whatever@1');
```

## Q10. Login history with failed-attempt counts
```sql
SELECT * FROM Login_Log ORDER BY username, login_time;

SELECT username,
       COUNT(*)                          AS total_attempts,
       SUM(status = 'Success')           AS successes,
       SUM(status = 'Failed')            AS failed_attempts,
       SUM(status = 'Locked')            AS blocked_attempts
FROM Login_Log
GROUP BY username;
-- neha: 5 attempts, 1 success, 3 failed, 1 locked
```

---
### Viva quick points
- Authentication (who you are: user + password) vs authorization (what you may do: GRANT/REVOKE).
- Privilege levels: global, database, table, column, routine. `WITH GRANT OPTION` lets a user pass on privileges he holds.
- Roles group privileges; `SET DEFAULT ROLE` activates them at login. Views and stored procedures (SQL SECURITY DEFINER) give controlled access.
- Never store plain-text passwords; use a hash (production systems should use a salted, slow hash such as bcrypt/Argon2 – SHA2 is used here as the exam asks).
