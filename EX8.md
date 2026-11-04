# Experiment 8 — MongoDB database for student records (reports with aggregation)
Platform: MongoDB 7 with `mongosh`. Slips: **8A Attendance report, 8B Result analysis, 8C Report card**.

Tip: in `mongosh` you can paste multi-line statements directly. Reusable pipeline pieces are stored in `const` variables.

---
# Slip 8A — Attendance report (database `attendanceDB`)
Defaulter = attendance below 75 %.

## Q1. Create database, collections, insert data
```js
use attendanceDB

db.students.insertMany([
  { _id: "S01", name: "Aditi Patil",   gender: "F", dept: "AIML", semester: 3, contact: { phone: "9822200001", city: "Sangli" } },
  { _id: "S02", name: "Rohit Kamble",  gender: "M", dept: "AIML", semester: 3, contact: { phone: "9822200002", city: "Miraj" } },
  { _id: "S03", name: "Sneha Mali",    gender: "F", dept: "CSE",  semester: 5, contact: { phone: "9822200003", city: "Kolhapur" } },
  { _id: "S04", name: "Yash Shinde",   gender: "M", dept: "CSE",  semester: 5, contact: { phone: "9822200004", city: "Sangli" } },
  { _id: "S05", name: "Pranav Joshi",  gender: "M", dept: "AIML", semester: 3, contact: { phone: "9822200005", city: "Satara" } }
])

db.courses.insertMany([
  { _id: "C11", name: "DBMS",             credits: 4, faculty: "Dr. Patil", semester: 3 },
  { _id: "C12", name: "DAA",              credits: 4, faculty: "Dr. Kale",  semester: 3 },
  { _id: "C21", name: "Computer Networks", credits: 3, faculty: "Dr. Joshi", semester: 5 }
])

db.attendance.insertMany([
  { student_id: "S01", course_id: "C11", total: 40, attended: 37 },
  { student_id: "S01", course_id: "C12", total: 38, attended: 30 },
  { student_id: "S02", course_id: "C11", total: 40, attended: 26 },
  { student_id: "S02", course_id: "C12", total: 38, attended: 33 },
  { student_id: "S05", course_id: "C11", total: 40, attended: 31 },
  { student_id: "S05", course_id: "C12", total: 38, attended: 24 },
  { student_id: "S03", course_id: "C21", total: 36, attended: 30 },
  { student_id: "S04", course_id: "C21", total: 36, attended: 25 }
])
```

## Q2. All AIML students
```js
db.students.find({ dept: "AIML" })
```

## Q3. Students from Sangli – name and phone only
```js
db.students.find({ "contact.city": "Sangli" }, { _id: 0, name: 1, "contact.phone": 1 })
```

## Q4. Attendance percentage for every record
```js
const pctStage = { $addFields: { pct: { $round: [ { $multiply: [ { $divide: ["$attended", "$total"] }, 100 ] }, 2 ] } } }

db.attendance.aggregate([
  pctStage,
  { $project: { _id: 0, student_id: 1, course_id: 1, total: 1, attended: 1, pct: 1 } }
])
// S01 C11 92.5, S01 C12 78.95, S02 C11 65, S02 C12 86.84, S05 C11 77.5, S05 C12 63.16, S03 C21 83.33, S04 C21 69.44
```

## Q5. Defaulters with student and course name ($lookup)
```js
db.attendance.aggregate([
  pctStage,
  { $match: { pct: { $lt: 75 } } },
  { $lookup: { from: "students", localField: "student_id", foreignField: "_id", as: "s" } },
  { $lookup: { from: "courses",  localField: "course_id",  foreignField: "_id", as: "c" } },
  { $unwind: "$s" }, { $unwind: "$c" },
  { $project: { _id: 0, student: "$s.name", course: "$c.name", pct: 1 } }
])
// Rohit Kamble – DBMS 65, Pranav Joshi – DAA 63.16, Yash Shinde – Computer Networks 69.44
```

## Q6. Course-wise average attendance percentage
```js
db.attendance.aggregate([
  pctStage,
  { $group: { _id: "$course_id", avg_pct: { $avg: "$pct" } } },
  { $lookup: { from: "courses", localField: "_id", foreignField: "_id", as: "c" } },
  { $unwind: "$c" },
  { $project: { _id: 0, course: "$c.name", avg_pct: { $round: ["$avg_pct", 2] } } }
])
// DBMS 78.33, DAA 76.32, Computer Networks 76.39
```

## Q7. 2 more C12 lectures held; S02 attended both
```js
// everyone's total for C12 goes up by 2 (two lectures were held)
db.attendance.updateMany({ course_id: "C12" }, { $inc: { total: 2 } })
// S02 attended both
db.attendance.updateOne({ student_id: "S02", course_id: "C12" }, { $inc: { attended: 2 } })

db.attendance.find({ course_id: "C12" })       // totals 40; S02 attended 35
```

## Q8. Add student S06 with attendance in C11 and C12
```js
db.students.insertOne({
  _id: "S06", name: "Tanmay Kulkarni", gender: "M", dept: "AIML", semester: 3,
  contact: { phone: "9822200006", city: "Sangli" }
})
db.attendance.insertMany([
  { student_id: "S06", course_id: "C11", total: 40, attended: 34 },
  { student_id: "S06", course_id: "C12", total: 40, attended: 28 }
])
```

## Q9. Number of defaulters in each department
```js
db.attendance.aggregate([
  pctStage,
  { $match: { pct: { $lt: 75 } } },
  { $lookup: { from: "students", localField: "student_id", foreignField: "_id", as: "s" } },
  { $unwind: "$s" },
  { $group: { _id: "$s.dept", defaulters: { $addToSet: "$student_id" } } },   // distinct students
  { $project: { _id: 0, dept: "$_id", defaulter_count: { $size: "$defaulters" } } }
])
// After Q7/Q8: AIML 3 (S02, S05, S06), CSE 1 (S04)
```

## Q10. Delete S04's attendance
```js
db.attendance.deleteMany({ student_id: "S04" })
db.attendance.countDocuments({ student_id: "S04" })      // 0
```

---
# Slip 8B — Result analysis (database `resultDB`)
Pass mark: total ≥ 40.

## Q1. Create, insert, display sorted by total
```js
use resultDB

db.students.insertMany([
  { _id: "R01", name: "Kiran Desai",    gender: "M", dept: "AIML", semester: 5 },
  { _id: "R02", name: "Lata Pawar",     gender: "F", dept: "AIML", semester: 5 },
  { _id: "R03", name: "Manish Gore",    gender: "M", dept: "AIML", semester: 5 },
  { _id: "R04", name: "Nisha Kulkarni", gender: "F", dept: "AIML", semester: 5 }
])
db.courses.insertMany([
  { _id: "M51", name: "Machine Learning", credits: 4, faculty: "Dr. Kale" },
  { _id: "M52", name: "Deep Learning",    credits: 4, faculty: "Dr. Rao" },
  { _id: "M53", name: "Data Mining",      credits: 3, faculty: "Dr. Shah" }
])
db.exams.insertMany([
  { student_id: "R01", course_id: "M51", ise: 28, ese: 40, total: 68 },
  { student_id: "R01", course_id: "M52", ise: 15, ese: 18, total: 33 },
  { student_id: "R02", course_id: "M51", ise: 37, ese: 55, total: 92 },
  { student_id: "R02", course_id: "M52", ise: 33, ese: 47, total: 80 },
  { student_id: "R02", course_id: "M53", ise: 0,  ese: 0,  total: 0 },
  { student_id: "R03", course_id: "M51", ise: 22, ese: 30, total: 52 },
  { student_id: "R03", course_id: "M53", ise: 30, ese: 45, total: 75 },
  { student_id: "R04", course_id: "M52", ise: 38, ese: 54, total: 92 },
  { student_id: "R04", course_id: "M53", ise: 20, ese: 15, total: 35 }
])

db.exams.find().sort({ total: -1 })
```

## Q2. Students who failed in any course
```js
db.exams.aggregate([
  { $match: { total: { $lt: 40 } } },
  { $lookup: { from: "students", localField: "student_id", foreignField: "_id", as: "s" } },
  { $unwind: "$s" },
  { $group: { _id: "$s.name", failed_courses: { $push: "$course_id" } } }
])
// Kiran Desai [M52], Lata Pawar [M53], Nisha Kulkarni [M53]
```

## Q3. Course-wise average, highest, lowest
```js
db.exams.aggregate([
  { $group: { _id: "$course_id", avg_total: { $avg: "$total" }, highest: { $max: "$total" }, lowest: { $min: "$total" } } },
  { $sort: { _id: 1 } }
])
// M51 avg 70.67 (92/52), M52 avg 68.33 (92/33), M53 avg 36.67 (75/0)
```

## Q4. Course-wise pass percentage
```js
db.exams.aggregate([
  { $group: { _id: "$course_id",
              appeared: { $sum: 1 },
              passed:   { $sum: { $cond: [ { $gte: ["$total", 40] }, 1, 0 ] } } } },
  { $project: { appeared: 1, passed: 1,
                pass_percentage: { $round: [ { $multiply: [ { $divide: ["$passed", "$appeared"] }, 100 ] }, 2 ] } } },
  { $sort: { _id: 1 } }
])
// M51 100, M52 66.67, M53 33.33   (the absent student R02/M53 is counted as failed)
```

## Q5. Topper of each course with student name
```js
db.exams.aggregate([
  { $sort: { total: -1 } },
  { $group: { _id: "$course_id", top_total: { $first: "$total" }, student_id: { $first: "$student_id" } } },
  { $lookup: { from: "students", localField: "student_id", foreignField: "_id", as: "s" } },
  { $unwind: "$s" },
  { $project: { _id: 0, course: "$_id", topper: "$s.name", total: "$top_total" } },
  { $sort: { course: 1 } }
])
// M51 Lata Pawar 92, M52 Nisha Kulkarni 92, M53 Manish Gore 75
```

## Q6. Add grade field (pipeline update)
```js
const gradeExpr = { $switch: {
  branches: [
    { case: { $gte: ["$total", 90] }, then: "O" },
    { case: { $gte: ["$total", 75] }, then: "A" },
    { case: { $gte: ["$total", 60] }, then: "B" },
    { case: { $gte: ["$total", 40] }, then: "C" }
  ],
  default: "F"
} }

db.exams.updateMany({}, [ { $set: { grade: gradeExpr } } ])
db.exams.find({}, { _id: 0, student_id: 1, course_id: 1, total: 1, grade: 1 })
// 68 B, 33 F, 92 O, 80 A, 0 F, 52 C, 75 A, 92 O, 35 F
```

## Q7. Students per grade in each course
```js
db.exams.aggregate([
  { $group: { _id: { course: "$course_id", grade: "$grade" }, students: { $sum: 1 } } },
  { $sort: { "_id.course": 1, "_id.grade": 1 } }
])
```

## Q8. Faculty-wise average marks
```js
db.exams.aggregate([
  { $lookup: { from: "courses", localField: "course_id", foreignField: "_id", as: "c" } },
  { $unwind: "$c" },
  { $group: { _id: "$c.faculty", avg_marks: { $avg: "$total" } } },
  { $project: { _id: 0, faculty: "$_id", avg_marks: { $round: ["$avg_marks", 2] } } }
])
// Dr. Kale 70.67, Dr. Rao 68.33, Dr. Shah 36.67
```

## Q9. Revaluation – R01 ESE in M52 + 5
```js
// before
db.exams.findOne({ student_id: "R01", course_id: "M52" })        // ese 18, total 33, grade F

db.exams.updateOne(
  { student_id: "R01", course_id: "M52" },
  [ { $set: { ese: { $add: ["$ese", 5] } } },
    { $set: { total: { $add: ["$ise", "$ese"] } } },
    { $set: { grade: gradeExpr } } ]
)

// after
db.exams.findOne({ student_id: "R01", course_id: "M52" })        // ese 23, total 38, grade F (still below 40)
```

## Q10. Delete absent records (total = 0)
```js
db.exams.deleteMany({ total: 0 })                 // removes R02 / M53
db.exams.countDocuments()                         // 8
```

---
# Slip 8C — Report card (database `reportDB`)
Pass mark: total ≥ 40. Defaulter: attendance < 75 %. Grade points O=10, A=9, B=8, C=6, F=0.

## Q1. Create, insert, list each student's courses with marks
```js
use reportDB

db.students.insertMany([
  { _id: "G01", name: "Om Chavan",      gender: "M", dept: "CSE", semester: 3, contact: { phone: "9833300001", city: "Sangli" } },
  { _id: "G02", name: "Pooja Rane",     gender: "F", dept: "CSE", semester: 3, contact: { phone: "9833300002", city: "Pune" } },
  { _id: "G03", name: "Rahul Salunkhe", gender: "M", dept: "CSE", semester: 3, contact: { phone: "9833300003", city: "Satara" } }
])
db.courses.insertMany([
  { _id: "D31", name: "Data Structures",    credits: 4, faculty: "Dr. Joshi" },
  { _id: "D32", name: "OOP with Java",      credits: 4, faculty: "Dr. Naik" },
  { _id: "D33", name: "Digital Electronics", credits: 3, faculty: "Dr. Kulkarni" }
])
db.attendance.insertMany([
  { student_id: "G01", course_id: "D31", total: 40, attended: 36 },
  { student_id: "G01", course_id: "D32", total: 40, attended: 34 },
  { student_id: "G01", course_id: "D33", total: 30, attended: 27 },
  { student_id: "G02", course_id: "D31", total: 40, attended: 29 },
  { student_id: "G02", course_id: "D32", total: 40, attended: 38 },
  { student_id: "G02", course_id: "D33", total: 30, attended: 28 },
  { student_id: "G03", course_id: "D31", total: 40, attended: 33 },
  { student_id: "G03", course_id: "D32", total: 40, attended: 31 },
  { student_id: "G03", course_id: "D33", total: 30, attended: 20 }
])
db.exams.insertMany([
  { student_id: "G01", course_id: "D31", ise: 35, ese: 55, total: 90 },
  { student_id: "G01", course_id: "D32", ise: 30, ese: 48, total: 78 },
  { student_id: "G01", course_id: "D33", ise: 28, ese: 40, total: 68 },
  { student_id: "G02", course_id: "D31", ise: 20, ese: 25, total: 45 },
  { student_id: "G02", course_id: "D32", ise: 36, ese: 52, total: 88 },
  { student_id: "G02", course_id: "D33", ise: 32, ese: 44, total: 76 },
  { student_id: "G03", course_id: "D31", ise: 18, ese: 16, total: 34 },
  { student_id: "G03", course_id: "D32", ise: 25, ese: 35, total: 60 },
  { student_id: "G03", course_id: "D33", ise: 27, ese: 38, total: 65 }
])

db.exams.aggregate([
  { $lookup: { from: "students", localField: "student_id", foreignField: "_id", as: "s" } },
  { $lookup: { from: "courses",  localField: "course_id",  foreignField: "_id", as: "c" } },
  { $unwind: "$s" }, { $unwind: "$c" },
  { $project: { _id: 0, student: "$s.name", course: "$c.name", ise: 1, ese: 1, total: 1 } },
  { $sort: { student: 1 } }
])
```

## Q2. One aggregation joining students, courses, exams and attendance
```js
const joined = [
  { $lookup: { from: "students", localField: "student_id", foreignField: "_id", as: "stu" } },
  { $lookup: { from: "courses",  localField: "course_id",  foreignField: "_id", as: "crs" } },
  { $lookup: {
      from: "attendance",
      let: { s: "$student_id", c: "$course_id" },
      pipeline: [ { $match: { $expr: { $and: [ { $eq: ["$student_id", "$$s"] }, { $eq: ["$course_id", "$$c"] } ] } } } ],
      as: "att" } },
  { $unwind: "$stu" }, { $unwind: "$crs" }, { $unwind: "$att" }
]

db.exams.aggregate(joined)          // each exam record now carries student, course and attendance data
```

## Q3. Name, course, marks, attendance %, grade
```js
const withGrade = [
  { $addFields: {
      att_pct: { $round: [ { $multiply: [ { $divide: ["$att.attended", "$att.total"] }, 100 ] }, 2 ] },
      grade: { $switch: { branches: [
        { case: { $gte: ["$total", 90] }, then: "O" },
        { case: { $gte: ["$total", 75] }, then: "A" },
        { case: { $gte: ["$total", 60] }, then: "B" },
        { case: { $gte: ["$total", 40] }, then: "C" } ], default: "F" } } } },
  { $addFields: { grade_point: { $switch: { branches: [
        { case: { $eq: ["$grade", "O"] }, then: 10 },
        { case: { $eq: ["$grade", "A"] }, then: 9 },
        { case: { $eq: ["$grade", "B"] }, then: 8 },
        { case: { $eq: ["$grade", "C"] }, then: 6 } ], default: 0 } } } }
]

db.exams.aggregate([
  ...joined, ...withGrade,
  { $project: { _id: 0, student: "$stu.name", course: "$crs.name", total: 1, att_pct: 1, grade: 1 } },
  { $sort: { student: 1 } }
])
// Om: DS 90 O 90.0 | OOP 78 A 85.0 | DE 68 B 90.0
// Pooja: DS 45 C 72.5 | OOP 88 A 95.0 | DE 76 A 93.33
// Rahul: DS 34 F 82.5 | OOP 60 B 77.5 | DE 65 B 66.67
```

## Q4. SGPA = Σ(grade point × credits) / Σ credits
```js
db.exams.aggregate([
  ...joined, ...withGrade,
  { $group: { _id: "$student_id", name: { $first: "$stu.name" },
              points:  { $sum: { $multiply: ["$grade_point", "$crs.credits"] } },
              credits: { $sum: "$crs.credits" } } },
  { $project: { name: 1, sgpa: { $round: [ { $divide: ["$points", "$credits"] }, 2 ] } } },
  { $sort: { _id: 1 } }
])
// G01 Om Chavan 9.09 (100/11), G02 Pooja Rane 7.91 (87/11), G03 Rahul Salunkhe 5.09 (56/11)
```

## Q5. Class topper by SGPA
```js
db.exams.aggregate([
  ...joined, ...withGrade,
  { $group: { _id: "$student_id", name: { $first: "$stu.name" },
              points: { $sum: { $multiply: ["$grade_point", "$crs.credits"] } },
              credits: { $sum: "$crs.credits" } } },
  { $project: { name: 1, sgpa: { $round: [ { $divide: ["$points", "$credits"] }, 2 ] } } },
  { $sort: { sgpa: -1 } },
  { $limit: 1 }
])
// Om Chavan – 9.09
```

## Q6–Q8. Complete report-card pipeline (result, remark, SGPA)
```js
const reportPipeline = [
  ...joined, ...withGrade,
  { $group: {
      _id: "$student_id",
      name: { $first: "$stu.name" }, dept: { $first: "$stu.dept" }, semester: { $first: "$stu.semester" },
      courses: { $push: { course: "$crs.name", credits: "$crs.credits", ise: "$ise", ese: "$ese",
                          total: "$total", attendance_pct: "$att_pct", grade: "$grade" } },
      points:  { $sum: { $multiply: ["$grade_point", "$crs.credits"] } },
      credits: { $sum: "$crs.credits" },
      min_total: { $min: "$total" },
      min_att:   { $min: "$att_pct" } } },
  { $project: {
      name: 1, dept: 1, semester: 1, courses: 1,
      sgpa:   { $round: [ { $divide: ["$points", "$credits"] }, 2 ] },
      result: { $cond: [ { $lt: ["$min_total", 40] }, "Fail", "Pass" ] },             // Q7
      remark: { $cond: [ { $lt: ["$min_att", 75] }, "Detained", "Eligible" ] } } },    // Q8
  { $sort: { _id: 1 } }
]
```

### Q6. Report card of G02 only
```js
db.exams.aggregate([ ...reportPipeline, { $match: { _id: "G02" } } ])
// Pooja Rane: SGPA 7.91, result Pass, remark Detained (DS attendance 72.5 %)
```

### Q7. Pass / Fail result
```js
db.exams.aggregate([ ...reportPipeline, { $project: { name: 1, result: 1 } } ])
// G01 Pass, G02 Pass, G03 Fail (Data Structures 34 < 40)
```

### Q8. 'Detained' remark where attendance < 75 % in any course
```js
db.exams.aggregate([ ...reportPipeline, { $match: { remark: "Detained" } }, { $project: { name: 1, remark: 1 } } ])
// G02 Pooja Rane (72.5 % in DS), G03 Rahul Salunkhe (66.67 % in DE)
```

## Q9. Save the report in report_cards using $out
```js
db.exams.aggregate([ ...reportPipeline, { $out: "report_cards" } ])
db.report_cards.find().pretty()
db.report_cards.countDocuments()          // 3
```

## Q10. Export report_cards to CSV
**Option 1 – mongoexport** (run in the normal terminal, not inside mongosh; arrays such as `courses` cannot be exported to CSV, so list flat fields):
```bash
mongoexport --db reportDB --collection report_cards --type=csv \
  --fields _id,name,dept,semester,sgpa,result,remark --out report_cards.csv
```

**Option 2 – Python (flattens one row per course)**
```python
import csv
from pymongo import MongoClient          # pip install pymongo

col = MongoClient("mongodb://localhost:27017")["reportDB"]["report_cards"]
with open("report_cards.csv", "w", newline="") as f:
    w = csv.writer(f)
    w.writerow(["student_id", "name", "course", "credits", "total", "attendance_pct",
                "grade", "sgpa", "result", "remark"])
    for d in col.find():
        for c in d["courses"]:
            w.writerow([d["_id"], d["name"], c["course"], c["credits"], c["total"],
                        c["attendance_pct"], c["grade"], d["sgpa"], d["result"], d["remark"]])
print("report_cards.csv written")
```

---
### Viva quick points
- Aggregation pipeline stages: `$match $group $project $sort $limit $lookup $unwind $addFields $out`.
- `$lookup` = left outer join between collections; `$unwind` flattens arrays; `$group` with `$sum/$avg/$min/$max/$first/$push`.
- SGPA = Σ(credit × grade point) / Σ credits. Embedded documents (`contact.city`) vs referencing (`student_id`).
