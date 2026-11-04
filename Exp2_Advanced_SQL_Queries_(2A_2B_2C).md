# Experiment 2 — Advanced SQL queries (joins, nested queries, views, aggregates, GROUP BY, HAVING)
Platform: MySQL 8. Slips: **2A Hotel reservations, 2B Courier service, 2C Cricket league**.

---
# Slip 2A — Hotel reservations

## Setup
```sql
CREATE DATABASE IF NOT EXISTS hotelDB;  USE hotelDB;

CREATE TABLE Room (room_no INT PRIMARY KEY, room_type VARCHAR(15), tariff DECIMAL(8,2), floor INT);
CREATE TABLE Guest (guest_id INT PRIMARY KEY, name VARCHAR(40), city VARCHAR(30), loyalty VARCHAR(10));
CREATE TABLE Reservation (
  res_id INT PRIMARY KEY, guest_id INT, room_no INT,
  check_in DATE, check_out DATE, status VARCHAR(12),
  FOREIGN KEY (guest_id) REFERENCES Guest(guest_id),
  FOREIGN KEY (room_no)  REFERENCES Room(room_no));

INSERT INTO Room VALUES (101,'Deluxe',3500,1),(102,'Deluxe',3500,1),(201,'Suite',7000,2),
                        (202,'Standard',2000,2),(301,'Standard',2000,3);
INSERT INTO Guest VALUES (1,'Vivek Rao','Pune','Gold'),(2,'Meghana Iyer','Bengaluru','Silver'),
                         (3,'Arjun Naik','Goa','Gold'),(4,'Ritu Sharma','Delhi','None');
INSERT INTO Reservation VALUES
 (1,1,101,'2026-10-01','2026-10-04','Completed'),
 (2,2,201,'2026-10-03','2026-10-05','Completed'),
 (3,1,202,'2026-10-15','2026-10-16','Completed'),
 (4,3,201,'2026-10-20','2026-10-23','Completed'),
 (5,3,101,'2026-11-05','2026-11-07','Booked'),
 (6,2,301,'2026-11-10','2026-11-12','Cancelled');
```

## Q1. INNER JOIN
```sql
SELECT g.name, r.room_no, r.check_in
FROM Reservation r INNER JOIN Guest g ON r.guest_id = g.guest_id;
```

## Q2. LEFT JOIN (rooms never reserved)
```sql
SELECT rm.room_no, rm.room_type, r.res_id, r.check_in, r.status
FROM Room rm LEFT JOIN Reservation r ON rm.room_no = r.room_no;
-- Room 102 appears with NULLs (never reserved)
```

## Q3. Number of nights
```sql
SELECT res_id, check_in, check_out, DATEDIFF(check_out, check_in) AS nights FROM Reservation;
-- 3, 2, 1, 3, 2, 2
```

## Q4. Reservations per room type
```sql
SELECT rm.room_type, COUNT(*) AS total_reservations
FROM Reservation r JOIN Room rm ON r.room_no = rm.room_no
GROUP BY rm.room_type;
-- Deluxe 2, Suite 2, Standard 2
```

## Q5. HAVING (> 2 reservations)
```sql
SELECT rm.room_type, COUNT(*) AS total_reservations
FROM Reservation r JOIN Room rm ON r.room_no = rm.room_no
GROUP BY rm.room_type
HAVING COUNT(*) > 2;
```
With the given data every room type has exactly 2 reservations, so this returns an **empty set** (that is correct). To show a non-empty HAVING result, run `HAVING COUNT(*) >= 2`, or insert one more reservation first.

## Q6. Highest, lowest, average tariff
```sql
SELECT MAX(tariff) AS highest, MIN(tariff) AS lowest, AVG(tariff) AS average FROM Room;
-- 7000, 2000, 3600
```

## Q7. Guests who booked a Suite (IN)
```sql
SELECT name FROM Guest
WHERE guest_id IN (SELECT guest_id FROM Reservation
                   WHERE room_no IN (SELECT room_no FROM Room WHERE room_type = 'Suite'));
-- Meghana Iyer, Arjun Naik
```

## Q8. Rooms above average tariff
```sql
SELECT * FROM Room WHERE tariff > (SELECT AVG(tariff) FROM Room);   -- Room 201 only
```

## Q9. View
```sql
CREATE VIEW upcoming_bookings AS
SELECT * FROM Reservation WHERE status = 'Booked';
SELECT * FROM upcoming_bookings;     -- res_id 5
```

## Q10. Guests alphabetically with reservation count
```sql
SELECT g.name, COUNT(r.res_id) AS total_reservations
FROM Guest g LEFT JOIN Reservation r ON g.guest_id = r.guest_id
GROUP BY g.guest_id, g.name
ORDER BY g.name;
-- Arjun 2, Meghana 2, Ritu 0, Vivek 2
```

---
# Slip 2B — Courier service

## Setup
```sql
CREATE DATABASE IF NOT EXISTS courierDB;  USE courierDB;

CREATE TABLE Branch (branch_id VARCHAR(3) PRIMARY KEY, city VARCHAR(30), zone VARCHAR(15));
CREATE TABLE Employee (
  emp_id INT PRIMARY KEY, name VARCHAR(30), branch_id VARCHAR(3), supervisor_id INT,
  role VARCHAR(15), salary DECIMAL(10,2),
  FOREIGN KEY (branch_id)     REFERENCES Branch(branch_id),
  FOREIGN KEY (supervisor_id) REFERENCES Employee(emp_id));
CREATE TABLE Shipment (
  ship_id INT PRIMARY KEY, from_branch VARCHAR(3), to_branch VARCHAR(3), agent_id INT,
  weight_kg DECIMAL(6,2), charge DECIMAL(8,2), ship_date DATE, status VARCHAR(15),
  FOREIGN KEY (from_branch) REFERENCES Branch(branch_id),
  FOREIGN KEY (to_branch)   REFERENCES Branch(branch_id),
  FOREIGN KEY (agent_id)    REFERENCES Employee(emp_id));

INSERT INTO Branch VALUES ('B1','Sangli','West'),('B2','Pune','West'),('B3','Nagpur','Central'),('B4','Nashik','North');
-- supervisor first (Manoj has NULL supervisor), then the rest
INSERT INTO Employee VALUES (1,'Manoj','B1',NULL,'Manager',45000),
                            (2,'Kiran','B1',1,'Agent',22000),
                            (3,'Asha','B2',1,'Manager',42000),
                            (4,'Deepak','B2',3,'Agent',21000),
                            (5,'Farah','B2',3,'Agent',23000),
                            (6,'Nilesh','B3',1,'Agent',20000);
INSERT INTO Shipment VALUES
 (1001,'B1','B2',2,2.5,180,'2026-09-02','Delivered'),
 (1002,'B2','B3',4,10,650,'2026-09-10','Delivered'),
 (1003,'B1','B3',2,1.2,220,'2026-09-25','Returned'),
 (1004,'B2','B1',5,6,400,'2026-10-04','Delivered'),
 (1005,'B3','B2',6,3,260,'2026-10-11','In Transit'),
 (1006,'B2','B1',4,15,900,'2026-10-18','Delivered');
```

## Q1. Shipment with agent name
```sql
SELECT s.ship_id, e.name AS agent, s.charge, s.status
FROM Shipment s INNER JOIN Employee e ON s.agent_id = e.emp_id;
```

## Q2. Self join – employee and supervisor
```sql
SELECT e.name AS employee, s.name AS supervisor
FROM Employee e LEFT JOIN Employee s ON e.supervisor_id = s.emp_id;
-- Manoj shows NULL supervisor (LEFT JOIN keeps him)
```

## Q3. Branches with employees (including branches with none)
```sql
SELECT b.branch_id, b.city, e.name
FROM Branch b LEFT JOIN Employee e ON b.branch_id = e.branch_id;
-- B4 Nashik has NULL employee
```

## Q4. Total charge per status
```sql
SELECT status, SUM(charge) AS total_charge FROM Shipment GROUP BY status;
-- Delivered 2130, Returned 220, In Transit 260
```

## Q5. Agents with more than one shipment
```sql
SELECT e.name, COUNT(*) AS shipments
FROM Shipment s JOIN Employee e ON s.agent_id = e.emp_id
GROUP BY s.agent_id, e.name
HAVING COUNT(*) > 1;
-- Kiran 2, Deepak 2
```

## Q6. Highest-charge shipment
```sql
SELECT * FROM Shipment WHERE charge = (SELECT MAX(charge) FROM Shipment);   -- 1006, 900
```

## Q7. Employees earning above average salary
```sql
SELECT name, salary FROM Employee WHERE salary > (SELECT AVG(salary) FROM Employee);
-- avg = 28833.33 → Manoj, Asha
```

## Q8. Shipments sent from Pune (nested)
```sql
SELECT * FROM Shipment
WHERE from_branch IN (SELECT branch_id FROM Branch WHERE city = 'Pune');
-- 1002, 1004, 1006
```

## Q9. View
```sql
CREATE VIEW delivered_shipments AS
SELECT s.ship_id, e.name AS agent_name, s.charge
FROM Shipment s JOIN Employee e ON s.agent_id = e.emp_id
WHERE s.status = 'Delivered';
SELECT * FROM delivered_shipments;
```

## Q10. Month-wise total charge
```sql
SELECT DATE_FORMAT(ship_date,'%Y-%m') AS month, SUM(charge) AS total_charge
FROM Shipment
GROUP BY DATE_FORMAT(ship_date,'%Y-%m')
ORDER BY month;
-- 2026-09 → 1050, 2026-10 → 1560
```

---
# Slip 2C — Cricket league

## Setup
```sql
CREATE DATABASE IF NOT EXISTS cricketDB;  USE cricketDB;

CREATE TABLE Team (team_id VARCHAR(3) PRIMARY KEY, team_name VARCHAR(30), city VARCHAR(30));
CREATE TABLE Player (player_id INT PRIMARY KEY, name VARCHAR(30), team_id VARCHAR(3), role VARCHAR(15),
                     FOREIGN KEY (team_id) REFERENCES Team(team_id));
CREATE TABLE Performance (match_no INT, player_id INT, runs INT, balls INT, wickets INT,
                          PRIMARY KEY (match_no, player_id),
                          FOREIGN KEY (player_id) REFERENCES Player(player_id));

INSERT INTO Team VALUES ('T1','Sangli Strikers','Sangli'),('T2','Pune Panthers','Pune'),('T3','Kolhapur Kings','Kolhapur');
INSERT INTO Player VALUES (1,'Ishan','T1','Batter'),(2,'Yash','T1','Bowler'),(3,'Mihir','T2','Batter'),
                          (4,'Om','T2','All-rounder'),(5,'Rutuja','T3','Batter'),(6,'Soham','T3','Bowler'),
                          (7,'Parth','T1','All-rounder');
INSERT INTO Performance VALUES
 (1,1,72,48,0),(1,2,5,6,3),(1,3,40,35,0),(1,4,28,20,2),
 (2,1,15,18,0),(2,5,88,55,0),(2,6,2,4,4),
 (3,3,65,40,0),(3,4,10,8,1),(3,6,0,1,2);
```

## Q1. Player, team, runs for every performance
```sql
SELECT p.name, t.team_name, f.runs
FROM Performance f
JOIN Player p ON f.player_id = p.player_id
JOIN Team t   ON p.team_id   = t.team_id;
```

## Q2. Total runs per player
```sql
SELECT p.name, SUM(f.runs) AS total_runs
FROM Player p JOIN Performance f ON p.player_id = f.player_id
GROUP BY p.player_id, p.name;
```

## Q3. Team runs, only teams above 100
```sql
SELECT t.team_name, SUM(f.runs) AS team_runs
FROM Team t JOIN Player p ON t.team_id = p.team_id
            JOIN Performance f ON p.player_id = f.player_id
GROUP BY t.team_id, t.team_name
HAVING SUM(f.runs) > 100;
-- Pune Panthers 143 (Sangli 92, Kolhapur 90 are filtered out)
```

## Q4. Highest single-innings score
```sql
SELECT p.name, f.runs
FROM Performance f JOIN Player p ON f.player_id = p.player_id
WHERE f.runs = (SELECT MAX(runs) FROM Performance);      -- Rutuja, 88
```

## Q5. Players with ≥ 2 wickets in a match
```sql
SELECT DISTINCT p.name
FROM Performance f JOIN Player p ON f.player_id = p.player_id
WHERE f.wickets >= 2;                                    -- Yash, Om, Soham
```

## Q6. Players who did not play (LEFT JOIN … IS NULL)
```sql
SELECT p.name
FROM Player p LEFT JOIN Performance f ON p.player_id = f.player_id
WHERE f.player_id IS NULL;                               -- Parth
```

## Q7. Players who scored more than the average runs (subquery)
```sql
SELECT DISTINCT p.name, f.runs
FROM Performance f JOIN Player p ON f.player_id = p.player_id
WHERE f.runs > (SELECT AVG(runs) FROM Performance);      -- average = 32.5
-- Ishan 72, Mihir 40, Rutuja 88, Mihir 65
```
(Alternative at player level: compare each player's `SUM(runs)` with `AVG` of player totals.)

## Q8. Players per team by role
```sql
SELECT t.team_name, p.role, COUNT(*) AS players
FROM Player p JOIN Team t ON p.team_id = t.team_id
GROUP BY t.team_name, p.role;
```

## Q9. View top_batters
```sql
CREATE VIEW top_batters AS
SELECT p.player_id, p.name, SUM(f.runs) AS total_runs
FROM Player p JOIN Performance f ON p.player_id = f.player_id
GROUP BY p.player_id, p.name
HAVING SUM(f.runs) > 50;
SELECT * FROM top_batters;     -- Ishan 87, Mihir 105, Rutuja 88
```

## Q10. Players ordered by total wickets
```sql
SELECT p.name, COALESCE(SUM(f.wickets),0) AS total_wickets
FROM Player p LEFT JOIN Performance f ON p.player_id = f.player_id
GROUP BY p.player_id, p.name
ORDER BY total_wickets DESC;
-- Soham 6, Yash 3, Om 3, Ishan 0, Mihir 0, Rutuja 0, Parth 0
```

---
### Viva quick points
- WHERE filters rows before grouping; HAVING filters groups after aggregation.
- INNER JOIN = matching rows only; LEFT JOIN = all left rows + matches (NULL otherwise); self join = table joined with itself via alias.
- A view is a stored SELECT (virtual table); it stores no data (unless materialized, not in MySQL).
- Correlated vs non-correlated subquery; IN vs EXISTS.
