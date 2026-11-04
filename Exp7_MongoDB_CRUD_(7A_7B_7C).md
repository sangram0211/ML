# Experiment 7 — MongoDB database and CRUD operations
Platform: MongoDB 7 with `mongosh` (or Compass → MONGOSH tab). Slips: **7A Food delivery, 7B Library, 7C IoT sensor readings**.

Start the shell with `mongosh`. `use <db>` creates the database lazily (it appears after the first insert).

---
# Slip 7A — Food delivery

## Q1. Create foodDB, insertMany
```js
use foodDB
db.restaurants.insertMany([
  { _id: "R1", name: "Spice Hub",   city: "Sangli", cuisine: "North Indian", rating: 4.3,
    menu: [ { item: "Paneer Tikka", price: 220 }, { item: "Dal Makhani", price: 180 } ] },
  { _id: "R2", name: "Dosa Corner", city: "Pune",   cuisine: "South Indian", rating: 4.6,
    menu: [ { item: "Masala Dosa", price: 90 }, { item: "Idli", price: 60 } ] },
  { _id: "R3", name: "Pizza Point", city: "Sangli", cuisine: "Italian",      rating: 3.9,
    menu: [ { item: "Margherita", price: 250 } ] }
])
```

## Q2. insertOne (own document)
```js
db.restaurants.insertOne({
  _id: "R4", name: "Biryani House", city: "Kolhapur", cuisine: "Mughlai", rating: 3.8,
  menu: [ { item: "Chicken Biryani", price: 280 } ]
})
```

## Q3. Display all
```js
db.restaurants.find()
db.restaurants.find().pretty()
```

## Q4. Restaurants in Sangli
```js
db.restaurants.find({ city: "Sangli" })        // Spice Hub, Pizza Point
```

## Q5. Rating above 4 – name and rating only
```js
db.restaurants.find({ rating: { $gt: 4 } }, { _id: 0, name: 1, rating: 1 })
// { name: 'Spice Hub', rating: 4.3 }, { name: 'Dosa Corner', rating: 4.6 }
```

## Q6. Sort by rating, highest first
```js
db.restaurants.find().sort({ rating: -1 })
```

## Q7. updateOne with $set
```js
db.restaurants.updateOne({ name: "Pizza Point" }, { $set: { rating: 4.1 } })
```

## Q8. $push a menu item
```js
db.restaurants.updateOne(
  { name: "Spice Hub" },
  { $push: { menu: { item: "Veg Biryani", price: 200 } } }
)
db.restaurants.find({ name: "Spice Hub" }, { _id: 0, name: 1, menu: 1 })
```

## Q9. updateMany – add field
```js
db.restaurants.updateMany({}, { $set: { is_open: true } })
```

## Q10. Delete rating below 4 and count
```js
db.restaurants.deleteMany({ rating: { $lt: 4 } })     // deletes Biryani House (3.8); Pizza Point is now 4.1
db.restaurants.countDocuments()                       // 3
```
(If your own document has rating ≥ 4, nothing is deleted and the count is 4.)

---
# Slip 7B — Library

## Q1. Create libraryDB and insert books and members
```js
use libraryDB
db.books.insertMany([
  { _id: 1, title: "Database Systems", author: "Korth",            genre: "Computer",  copies: 5, year: 2019 },
  { _id: 2, title: "Wings of Fire",    author: "A. P. J. Abdul Kalam", genre: "Biography", copies: 3, year: 1999 },
  { _id: 3, title: "Python Basics",    author: "Rao",              genre: "Computer",  copies: 0, year: 2022 }
])
db.members.insertMany([
  { _id: "M1", name: "Rutuja",
    borrowed: [ { book_id: 1, date: ISODate("2026-10-05") } ] },
  { _id: "M2", name: "Siddhesh",
    borrowed: [ { book_id: 1, date: ISODate("2026-10-12") }, { book_id: 2, date: ISODate("2026-10-15") } ] }
])
```

## Q2. All books
```js
db.books.find()
```

## Q3. Computer books
```js
db.books.find({ genre: "Computer" })
```

## Q4. Published after 2000 with copies > 0
```js
db.books.find({ year: { $gt: 2000 }, copies: { $gt: 0 } })     // Database Systems only
```

## Q5. Titles and authors, alphabetical by title
```js
db.books.find({}, { _id: 0, title: 1, author: 1 }).sort({ title: 1 })
```

## Q6. Members who borrowed book 1
```js
db.members.find({ "borrowed.book_id": 1 })                     // Rutuja, Siddhesh
```

## Q7. $inc copies of Python Basics by 2
```js
db.books.updateOne({ title: "Python Basics" }, { $inc: { copies: 2 } })    // 0 → 2
```

## Q8. $push borrow record for Rutuja
```js
db.members.updateOne(
  { name: "Rutuja" },
  { $push: { borrowed: { book_id: 2, date: ISODate("2026-11-04") } } }
)
```

## Q9. $pull book 1 from Siddhesh
```js
db.members.updateOne({ name: "Siddhesh" }, { $pull: { borrowed: { book_id: 1 } } })
db.members.find({ name: "Siddhesh" })          // only book 2 remains
```

## Q10. Delete 'Wings of Fire'
```js
db.books.deleteOne({ title: "Wings of Fire" })
db.books.find()                                // Database Systems, Python Basics
```

---
# Slip 7C — IoT sensor readings

## Q1. Create iotDB and insert
```js
use iotDB
db.readings.insertMany([
  { sensor_id: "T1", location: "Lab-1", type: "temperature", value: 28.5, time: ISODate("2026-11-01T09:00:00Z") },
  { sensor_id: "T1", location: "Lab-1", type: "temperature", value: 31.2, time: ISODate("2026-11-01T12:00:00Z") },
  { sensor_id: "H1", location: "Lab-1", type: "humidity",    value: 62,   time: ISODate("2026-11-01T09:00:00Z") },
  { sensor_id: "T2", location: "Lab-2", type: "temperature", value: 26.8, time: ISODate("2026-11-01T09:00:00Z") },
  { sensor_id: "H2", location: "Lab-2", type: "humidity",    value: 70,   time: ISODate("2026-11-01T12:00:00Z") }
])
```

## Q2. All readings
```js
db.readings.find()
```

## Q3. Temperature readings only
```js
db.readings.find({ type: "temperature" })
```

## Q4. Temperature above 30
```js
db.readings.find({ type: "temperature", value: { $gt: 30 } })      // T1 31.2
```

## Q5. Lab-2 readings sorted by time, sensor_id and value only
```js
db.readings.find({ location: "Lab-2" }, { _id: 0, sensor_id: 1, value: 1 }).sort({ time: 1 })
```

## Q6. countDocuments for Lab-1
```js
db.readings.countDocuments({ location: "Lab-1" })                  // 3
```

## Q7. updateMany – add unit
```js
db.readings.updateMany({ type: "temperature" }, { $set: { unit: "C" } })
db.readings.updateMany({ type: "humidity" },    { $set: { unit: "%" } })
```

## Q8. Update value of H2 to 68
```js
db.readings.updateOne({ sensor_id: "H2" }, { $set: { value: 68 } })
```

## Q9. $rename value → reading
```js
db.readings.updateMany({}, { $rename: { value: "reading" } })
db.readings.find()
```

## Q10. Delete readings before 10:00 and count
```js
db.readings.deleteMany({ time: { $lt: ISODate("2026-11-01T10:00:00Z") } })   // deletes the three 09:00 readings
db.readings.countDocuments()                                                 // 2
```

---
### Viva quick points
- MongoDB = document (BSON/JSON-like) NoSQL database; database → collections → documents; `_id` is the mandatory primary key (auto `ObjectId` if omitted).
- CRUD: `insertOne/insertMany`, `find/findOne`, `updateOne/updateMany/replaceOne`, `deleteOne/deleteMany`.
- Operators: `$gt $lt $gte $lte $in $and $or`; update operators `$set $inc $push $pull $rename $unset`.
- Embedded documents/arrays are queried with dot notation (`"borrowed.book_id"`).
