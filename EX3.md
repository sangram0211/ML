# Experiment 3 — Database normalization up to BCNF
Platform: MySQL 8. Slips: **3A Project_Assignment, 3B Course_Registration, 3C Lab_Slot (3NF but not BCNF)**.

---
# Slip 3A — Project_Assignment(EmpID, EmpName, Dept, DeptHead, ProjID, ProjName, Hours)

FDs: EmpID → EmpName, Dept; Dept → DeptHead; ProjID → ProjName; {EmpID, ProjID} → Hours

## Q1. Why not 1NF
Columns `ProjID`, `ProjName` and `Hours` hold **multiple values in one cell** (e.g. E1 has P1, P2 / Billing, Portal / 10, 15). 1NF requires every attribute to be atomic (single-valued).

## Q2. Convert to 1NF (one value per cell)
| EmpID | EmpName | Dept | DeptHead | ProjID | ProjName | Hours |
|---|---|---|---|---|---|---|
| E1 | Ajay | IT | Mr. Rao | P1 | Billing | 10 |
| E1 | Ajay | IT | Mr. Rao | P2 | Portal | 15 |
| E2 | Bhavna | HR | Ms. Iyer | P2 | Portal | 20 |
| E3 | Chetan | IT | Mr. Rao | P1 | Billing | 12 |
| E3 | Chetan | IT | Mr. Rao | P3 | Analytics | 8 |

## Q3. Candidate key
Closure of {EmpID, ProjID}: EmpID → EmpName, Dept; Dept → DeptHead; ProjID → ProjName; {EmpID, ProjID} → Hours ⇒ closure = all attributes. Neither EmpID nor ProjID alone determines everything, so the **candidate key (and primary key) = {EmpID, ProjID}**.

## Q4. Partial dependencies → 2NF
Partial dependencies (part of the key determines a non-key attribute):
- EmpID → EmpName, Dept (and DeptHead through Dept)
- ProjID → ProjName

2NF decomposition:
- **Employee(EmpID, EmpName, Dept, DeptHead)**
- **Project(ProjID, ProjName)**
- **Assignment(EmpID, ProjID, Hours)**

## Q5. Transitive dependency → 3NF
EmpID → Dept → DeptHead is transitive (non-key → non-key). Split Employee:
- **Employee(EmpID, EmpName, Dept)** – Dept FK
- **Department(Dept, DeptHead)**
- Project(ProjID, ProjName)
- Assignment(EmpID, ProjID, Hours)

## Q6. BCNF check
| Table | FD | Determinant is a key? |
|---|---|---|
| Employee | EmpID → EmpName, Dept | Yes (EmpID) |
| Department | Dept → DeptHead | Yes (Dept) |
| Project | ProjID → ProjName | Yes (ProjID) |
| Assignment | {EmpID, ProjID} → Hours | Yes (composite key) |

Every determinant is a superkey ⇒ **all four tables are in BCNF**.

## Q7. Create final tables
```sql
CREATE DATABASE IF NOT EXISTS projDB;  USE projDB;

CREATE TABLE Department (
  dept      VARCHAR(10) PRIMARY KEY,
  dept_head VARCHAR(30)
);
CREATE TABLE Employee (
  emp_id   VARCHAR(5) PRIMARY KEY,
  emp_name VARCHAR(30),
  dept     VARCHAR(10),
  FOREIGN KEY (dept) REFERENCES Department(dept)
);
CREATE TABLE Project (
  proj_id   VARCHAR(5) PRIMARY KEY,
  proj_name VARCHAR(30)
);
CREATE TABLE Assignment (
  emp_id  VARCHAR(5),
  proj_id VARCHAR(5),
  hours   INT,
  PRIMARY KEY (emp_id, proj_id),
  FOREIGN KEY (emp_id)  REFERENCES Employee(emp_id),
  FOREIGN KEY (proj_id) REFERENCES Project(proj_id)
);
```

## Q8. Insert data
```sql
INSERT INTO Department VALUES ('IT','Mr. Rao'),('HR','Ms. Iyer');
INSERT INTO Employee   VALUES ('E1','Ajay','IT'),('E2','Bhavna','HR'),('E3','Chetan','IT');
INSERT INTO Project    VALUES ('P1','Billing'),('P2','Portal'),('P3','Analytics');
INSERT INTO Assignment VALUES ('E1','P1',10),('E1','P2',15),('E2','P2',20),('E3','P1',12),('E3','P3',8);
```

## Q9. Lossless join (rebuild original)
```sql
SELECT e.emp_id, e.emp_name, e.dept, d.dept_head, p.proj_id, p.proj_name, a.hours
FROM Assignment a
JOIN Employee   e ON a.emp_id  = e.emp_id
JOIN Department d ON e.dept    = d.dept
JOIN Project    p ON a.proj_id = p.proj_id;
```
Returns exactly the 5 rows of the 1NF table – no spurious or missing tuples ⇒ lossless.

## Q10. Total hours per employee
```sql
SELECT e.emp_id, e.emp_name, SUM(a.hours) AS total_hours
FROM Employee e JOIN Assignment a ON e.emp_id = a.emp_id
GROUP BY e.emp_id, e.emp_name;
-- E1 Ajay 25, E2 Bhavna 20, E3 Chetan 20
```

---
# Slip 3B — Course_Registration(RegNo, StudName, CourseCode, CourseName, Instructor, InstrRoom, Credits)

FDs: RegNo → StudName; CourseCode → CourseName, Instructor, Credits; Instructor → InstrRoom

## Q1. Primary key
{RegNo, CourseCode}: together they determine every attribute (see Q3); no proper subset does.

## Q2. Anomalies
- **Insert anomaly:** a new course (e.g. CS304) cannot be stored until some student registers for it (RegNo would be NULL – part of the key).
- **Update anomaly:** changing Prof. Shah's room from A-101 must be done in every row where he appears (rows 1 and 3); updating one row leaves inconsistent data.
- **Delete anomaly:** deleting Isha's only registration (RegNo 23) also deletes all information about course CS303 / Prof. Bose.

## Q3. Closure of {RegNo, CourseCode}
{RegNo, CourseCode}⁺ = {RegNo, CourseCode} → + StudName (RegNo→StudName) → + CourseName, Instructor, Credits (CourseCode→…) → + InstrRoom (Instructor→InstrRoom)
= {RegNo, StudName, CourseCode, CourseName, Instructor, InstrRoom, Credits} = all attributes, so it is a key.

## Q4. Classify FDs
| FD | Type |
|---|---|
| RegNo → StudName | Partial (part of key → non-key) |
| CourseCode → CourseName, Instructor, Credits | Partial |
| Instructor → InstrRoom | Transitive (CourseCode → Instructor → InstrRoom) |
| {RegNo, CourseCode} → all | Full (key) |

## Q5. 2NF decomposition (remove partial dependencies)
- Student(**RegNo**, StudName)
- Course(**CourseCode**, CourseName, Instructor, InstrRoom, Credits)
- Registration(**RegNo, CourseCode**)

## Q6. 3NF decomposition (remove transitive dependency)
- Student(**RegNo**, StudName)
- Instructor(**Instructor**, InstrRoom)
- Course(**CourseCode**, CourseName, Instructor → Instructor, Credits)
- Registration(**RegNo → Student, CourseCode → Course**)

## Q7. BCNF check
Student: RegNo is key. Instructor: Instructor is key. Course: CourseCode is key (only FD). Registration: only trivial FDs (all-key). Every determinant is a superkey ⇒ **all tables are in BCNF**.

## Q8. Create and insert
```sql
CREATE DATABASE IF NOT EXISTS courseDB;  USE courseDB;

CREATE TABLE Student (reg_no INT PRIMARY KEY, stud_name VARCHAR(30));
CREATE TABLE Instructor (instructor VARCHAR(30) PRIMARY KEY, instr_room VARCHAR(10));
CREATE TABLE Course (
  course_code VARCHAR(10) PRIMARY KEY, course_name VARCHAR(30),
  instructor VARCHAR(30), credits INT,
  FOREIGN KEY (instructor) REFERENCES Instructor(instructor));
CREATE TABLE Registration (
  reg_no INT, course_code VARCHAR(10),
  PRIMARY KEY (reg_no, course_code),
  FOREIGN KEY (reg_no) REFERENCES Student(reg_no),
  FOREIGN KEY (course_code) REFERENCES Course(course_code));

INSERT INTO Student VALUES (21,'Gauri'),(22,'Harsh'),(23,'Isha');
INSERT INTO Instructor VALUES ('Prof. Shah','A-101'),('Prof. Nair','A-205'),('Prof. Bose','B-110');
INSERT INTO Course VALUES ('CS301','DBMS','Prof. Shah',4),('CS302','OS','Prof. Nair',3),('CS303','CN','Prof. Bose',3);
INSERT INTO Registration VALUES (21,'CS301'),(21,'CS302'),(22,'CS301'),(23,'CS303');
```

## Q9. Students taught by Prof. Shah
```sql
SELECT s.stud_name
FROM Student s
JOIN Registration r ON s.reg_no = r.reg_no
JOIN Course c       ON r.course_code = c.course_code
WHERE c.instructor = 'Prof. Shah';      -- Gauri, Harsh
```

## Q10. Room change needs one UPDATE
```sql
UPDATE Instructor SET instr_room = 'A-110' WHERE instructor = 'Prof. Shah';   -- 1 row affected
SELECT s.stud_name, c.course_name, i.instr_room
FROM Student s JOIN Registration r ON s.reg_no=r.reg_no
JOIN Course c ON r.course_code=c.course_code
JOIN Instructor i ON c.instructor=i.instructor;
```
The room is stored once, so all students see the new room – the update anomaly is gone.

---
# Slip 3C — Lab_Slot(Student, Course, LabInstructor)

FDs: {Student, Course} → LabInstructor; LabInstructor → Course

## Q1. Candidate keys
- {Student, Course}: closure = Student, Course, LabInstructor ✔
- {Student, LabInstructor}: LabInstructor → Course, so closure = Student, LabInstructor, Course ✔
- Neither Student, Course nor LabInstructor alone is a key.

**Candidate keys: {Student, Course} and {Student, LabInstructor}.**

## Q2. Prime / non-prime attributes
Prime (part of some candidate key): Student, Course, LabInstructor. Non-prime: **none**.

## Q3. Table is in 3NF
For each FD X → A, 3NF needs X to be a superkey **or** A to be prime.
- {Student, Course} → LabInstructor: X is a superkey ✔
- LabInstructor → Course: X is not a superkey, but Course is prime ✔
So the relation is in 3NF (no non-prime attribute exists, hence no partial/transitive dependency).

## Q4. Why not BCNF
BCNF needs the determinant of every FD to be a superkey. In LabInstructor → Course, LabInstructor is **not** a superkey (it does not determine Student). Hence the table violates BCNF.

## Q5. Update anomaly
Mr. Joshi moves from Python Lab to DBMS Lab. His course is stored in two rows: (Kavya, Python Lab, Mr. Joshi) and (Lalit, Python Lab, Mr. Joshi). Both rows must be changed; if only Kavya's row is updated, Mr. Joshi appears under two different courses, violating LabInstructor → Course. Kavya would also end up with two DBMS Lab rows (one with Ms. Rane, one with Mr. Joshi).

## Q6. BCNF decomposition
Decompose on the violating FD LabInstructor → Course:
- **R1(LabInstructor, Course)** — key: LabInstructor
- **R2(Student, LabInstructor)** — key: {Student, LabInstructor}

| R1 LabInstructor | Course |
|---|---|
| Mr. Joshi | Python Lab |
| Ms. Rane | DBMS Lab |
| Mr. Patil | Python Lab |

| R2 Student | LabInstructor |
|---|---|
| Kavya | Mr. Joshi |
| Kavya | Ms. Rane |
| Lalit | Mr. Joshi |
| Mansi | Mr. Patil |
| Mansi | Ms. Rane |

## Q7. Lossless?
R1 ∩ R2 = {LabInstructor}, and LabInstructor → Course makes it a key of R1 (R1 ∩ R2 → R1). Hence the decomposition is **lossless**; joining R1 and R2 on LabInstructor returns exactly the original 5 rows.

## Q8. Lost dependency
{Student, Course} → LabInstructor is **not preserved** – no single table contains Student, Course and LabInstructor together. This is the usual trade-off: BCNF decomposition is always lossless but may not be dependency-preserving (the constraint "a student has only one instructor per course" must now be enforced by a trigger/assertion).

## Q9. Create BCNF tables
```sql
CREATE DATABASE IF NOT EXISTS labDB;  USE labDB;

CREATE TABLE Instructor_Course (
  lab_instructor VARCHAR(30) PRIMARY KEY,
  course         VARCHAR(30)
);
CREATE TABLE Student_Instructor (
  student        VARCHAR(30),
  lab_instructor VARCHAR(30),
  PRIMARY KEY (student, lab_instructor),
  FOREIGN KEY (lab_instructor) REFERENCES Instructor_Course(lab_instructor)
);

INSERT INTO Instructor_Course VALUES ('Mr. Joshi','Python Lab'),('Ms. Rane','DBMS Lab'),('Mr. Patil','Python Lab');
INSERT INTO Student_Instructor VALUES ('Kavya','Mr. Joshi'),('Kavya','Ms. Rane'),
                                      ('Lalit','Mr. Joshi'),('Mansi','Mr. Patil'),('Mansi','Ms. Rane');
```

## Q10. Join to rebuild
```sql
SELECT si.student, ic.course, si.lab_instructor
FROM Student_Instructor si
JOIN Instructor_Course ic ON si.lab_instructor = ic.lab_instructor
ORDER BY si.student;
```

---
### Viva quick points
- 1NF atomic values; 2NF no partial dependency; 3NF no transitive dependency (or RHS prime); BCNF every determinant is a superkey.
- BCNF is stricter than 3NF; 3NF decomposition is lossless **and** dependency-preserving, BCNF is lossless but may lose dependencies.
- Lossless test for a binary decomposition: R1 ∩ R2 → R1 or R1 ∩ R2 → R2.
