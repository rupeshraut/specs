# Effective Modern MongoDB Query Patterns

A comprehensive guide to mastering MongoDB queries, aggregation framework, indexing, and performance optimization.

---

## Table of Contents

1. [MongoDB Fundamentals](#mongodb-fundamentals)
2. [CRUD Operations](#crud-operations)
3. [Query Operators](#query-operators)
4. [Aggregation Framework](#aggregation-framework)
5. [Indexing Strategies](#indexing-strategies)
6. [Query Optimization](#query-optimization)
7. [Spring Data MongoDB](#spring-data-mongodb)
8. [Transactions](#transactions)
9. [Change Streams](#change-streams)
10. [Schema Design Patterns](#schema-design-patterns)
11. [Performance Best Practices](#performance-best-practices)
12. [Common Pitfalls](#common-pitfalls)

---

## MongoDB Fundamentals

### Document Model

```javascript
// MongoDB stores data as BSON documents
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "name": "John Doe",
  "email": "john@example.com",
  "age": 30,
  "address": {
    "street": "123 Main St",
    "city": "New York",
    "zip": "10001"
  },
  "tags": ["premium", "verified"],
  "createdAt": ISODate("2024-01-15T10:30:00Z"),
  "metadata": {
    "lastLogin": ISODate("2024-01-20T14:30:00Z"),
    "loginCount": 42
  }
}

/*
Key Concepts:
- Documents are JSON-like (actually BSON)
- Schema-less (flexible schema)
- Nested documents (embedded documents)
- Arrays (can contain any type)
- Rich data types (ObjectId, Date, Binary, etc.)
*/
```

### Data Types

```javascript
// Common MongoDB Data Types

// ObjectId - 12-byte unique identifier
ObjectId("507f1f77bcf86cd799439011")

// String
"Hello World"

// Number (Int32, Int64, Double, Decimal128)
42
NumberLong("9223372036854775807")
NumberDecimal("123.45")

// Boolean
true

// Date
ISODate("2024-01-15T10:30:00Z")
new Date()

// Array
["tag1", "tag2", "tag3"]

// Embedded Document
{ "street": "123 Main St", "city": "NYC" }

// Null
null

// Binary Data
BinData(0, "base64encodeddata")

// Regular Expression
/pattern/i
```

---

## CRUD Operations

### 1. **Create (Insert)**

```javascript
// Insert single document
db.users.insertOne({
  name: "John Doe",
  email: "john@example.com",
  age: 30,
  createdAt: new Date()
})

// Insert multiple documents
db.users.insertMany([
  { name: "Alice", email: "alice@example.com", age: 25 },
  { name: "Bob", email: "bob@example.com", age: 35 },
  { name: "Charlie", email: "charlie@example.com", age: 28 }
])

// Insert with specific _id
db.users.insertOne({
  _id: "custom-id-123",
  name: "John Doe",
  email: "john@example.com"
})

// Ordered vs Unordered inserts
db.users.insertMany(
  [
    { name: "User1" },
    { name: "User2" },
    { _id: 1 },  // Duplicate, will fail
    { name: "User3" }
  ],
  { ordered: false }  // Continue on error
)
```

### 2. **Read (Find)**

```javascript
// Find all documents
db.users.find()

// Find with filter
db.users.find({ age: 30 })

// Find one document
db.users.findOne({ email: "john@example.com" })

// Find with multiple conditions (AND)
db.users.find({
  age: { $gt: 25 },
  status: "active"
})

// Find with OR
db.users.find({
  $or: [
    { age: { $lt: 25 } },
    { age: { $gt: 60 } }
  ]
})

// Projection - select specific fields
db.users.find(
  { age: { $gt: 25 } },
  { name: 1, email: 1, _id: 0 }  // Include name, email; exclude _id
)

// Limit and skip (pagination)
db.users.find()
  .limit(10)
  .skip(20)

// Sort
db.users.find()
  .sort({ age: -1, name: 1 })  // Age descending, name ascending

// Count
db.users.countDocuments({ age: { $gt: 25 } })
db.users.estimatedDocumentCount()  // Fast, less accurate

// Distinct values
db.users.distinct("city")
db.users.distinct("city", { age: { $gt: 25 } })
```

### 3. **Update**

```javascript
// Update one document
db.users.updateOne(
  { email: "john@example.com" },
  { $set: { age: 31, lastUpdated: new Date() } }
)

// Update multiple documents
db.users.updateMany(
  { status: "pending" },
  { $set: { status: "active" } }
)

// Replace entire document
db.users.replaceOne(
  { email: "john@example.com" },
  {
    name: "John Smith",
    email: "john@example.com",
    age: 31
  }
)

// Upsert - insert if not exists
db.users.updateOne(
  { email: "new@example.com" },
  { 
    $set: { name: "New User", email: "new@example.com" },
    $setOnInsert: { createdAt: new Date() }
  },
  { upsert: true }
)

// Update operators
db.users.updateOne(
  { email: "john@example.com" },
  {
    $set: { name: "John Smith" },           // Set field
    $unset: { middleName: "" },             // Remove field
    $inc: { loginCount: 1 },                // Increment
    $mul: { score: 1.1 },                   // Multiply
    $rename: { "addr": "address" },         // Rename field
    $currentDate: { lastModified: true },   // Set to current date
    $min: { lowScore: 50 },                 // Update if less than
    $max: { highScore: 100 }                // Update if greater than
  }
)

// Array update operators
db.users.updateOne(
  { email: "john@example.com" },
  {
    $push: { tags: "premium" },                    // Add to array
    $pull: { tags: "trial" },                      // Remove from array
    $addToSet: { tags: "verified" },               // Add if not exists
    $pop: { tags: 1 },                             // Remove first (-1) or last (1)
    $pullAll: { tags: ["old", "deprecated"] }      // Remove multiple
  }
)

// Update array elements
db.users.updateOne(
  { "orders.orderId": "ORDER-123" },
  { $set: { "orders.$.status": "shipped" } }  // $ = matched element
)

// Update all array elements
db.users.updateOne(
  { email: "john@example.com" },
  { $inc: { "orders.$[].quantity": 1 } }  // $[] = all elements
)

// Update with array filters
db.users.updateOne(
  { email: "john@example.com" },
  { $set: { "orders.$[elem].status": "cancelled" } },
  { arrayFilters: [{ "elem.amount": { $gt: 100 } }] }
)

// findAndModify - atomic find and update
db.users.findOneAndUpdate(
  { email: "john@example.com" },
  { $inc: { loginCount: 1 } },
  { 
    returnDocument: "after",  // Return updated document
    upsert: true
  }
)
```

### 4. **Delete**

```javascript
// Delete one document
db.users.deleteOne({ email: "john@example.com" })

// Delete multiple documents
db.users.deleteMany({ status: "inactive" })

// Delete all documents in collection
db.users.deleteMany({})

// findAndDelete - atomic find and delete
db.users.findOneAndDelete(
  { email: "john@example.com" },
  { projection: { name: 1, email: 1 } }  // Return these fields
)
```

---

## Query Operators

### 1. **Comparison Operators**

```javascript
// $eq - Equal
db.users.find({ age: { $eq: 30 } })
db.users.find({ age: 30 })  // Same as above

// $ne - Not equal
db.users.find({ status: { $ne: "inactive" } })

// $gt, $gte - Greater than, greater than or equal
db.users.find({ age: { $gt: 25 } })
db.users.find({ age: { $gte: 25 } })

// $lt, $lte - Less than, less than or equal
db.users.find({ age: { $lt: 60 } })
db.users.find({ age: { $lte: 60 } })

// $in - In array
db.users.find({ status: { $in: ["active", "pending"] } })

// $nin - Not in array
db.users.find({ status: { $nin: ["deleted", "banned"] } })

// Range query
db.users.find({ age: { $gte: 25, $lte: 60 } })
```

### 2. **Logical Operators**

```javascript
// $and - All conditions must be true
db.users.find({
  $and: [
    { age: { $gte: 25 } },
    { status: "active" },
    { city: "New York" }
  ]
})

// Implicit AND (more common)
db.users.find({
  age: { $gte: 25 },
  status: "active",
  city: "New York"
})

// $or - At least one condition must be true
db.users.find({
  $or: [
    { age: { $lt: 25 } },
    { age: { $gt: 60 } }
  ]
})

// $nor - All conditions must be false
db.users.find({
  $nor: [
    { status: "deleted" },
    { status: "banned" }
  ]
})

// $not - Negation
db.users.find({
  age: { $not: { $gt: 60 } }
})

// Complex logical combinations
db.users.find({
  $and: [
    {
      $or: [
        { age: { $lt: 25 } },
        { age: { $gt: 60 } }
      ]
    },
    { status: "active" }
  ]
})
```

### 3. **Element Operators**

```javascript
// $exists - Field exists
db.users.find({ middleName: { $exists: true } })
db.users.find({ middleName: { $exists: false } })

// $type - Field type
db.users.find({ age: { $type: "int" } })
db.users.find({ age: { $type: "double" } })
db.users.find({ tags: { $type: "array" } })

// Multiple types
db.users.find({ age: { $type: ["int", "double"] } })
```

### 4. **Array Operators**

```javascript
// $all - Array contains all elements
db.users.find({ tags: { $all: ["premium", "verified"] } })

// $elemMatch - Array element matches all conditions
db.users.find({
  orders: {
    $elemMatch: {
      status: "shipped",
      amount: { $gt: 100 }
    }
  }
})

// $size - Array size
db.users.find({ tags: { $size: 3 } })

// Array element query
db.users.find({ "tags.0": "premium" })  // First element
```

### 5. **String Operators**

```javascript
// $regex - Regular expression
db.users.find({ email: { $regex: /^john.*@example\.com$/ } })
db.users.find({ email: { $regex: "^john", $options: "i" } })  // Case-insensitive

// Text search (requires text index)
db.users.createIndex({ bio: "text" })
db.users.find({ $text: { $search: "mongodb developer" } })

// Text search with score
db.users.find(
  { $text: { $search: "mongodb developer" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })
```

### 6. **Nested Document Queries**

```javascript
// Dot notation
db.users.find({ "address.city": "New York" })

// Nested field with multiple conditions
db.users.find({
  "address.city": "New York",
  "address.zip": { $regex: /^100/ }
})

// Query entire nested document (exact match)
db.users.find({
  address: {
    street: "123 Main St",
    city: "New York",
    zip: "10001"
  }
})
```

---

## Aggregation Framework

### 1. **Basic Aggregation Pipeline**

```javascript
// Simple aggregation
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $group: {
      _id: "$customerId",
      totalAmount: { $sum: "$amount" },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { totalAmount: -1 } },
  { $limit: 10 }
])

// Aggregation stages explained:
// $match - Filter documents (like find)
// $group - Group by field and accumulate
// $sort - Sort results
// $limit - Limit number of results
// $skip - Skip documents
// $project - Reshape documents
// $unwind - Deconstruct array field
// $lookup - Left outer join
// $facet - Multiple pipelines
```

### 2. **$group Accumulators**

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $group: {
      _id: "$customerId",
      
      // Sum
      totalAmount: { $sum: "$amount" },
      count: { $sum: 1 },
      
      // Average
      avgAmount: { $avg: "$amount" },
      
      // Min/Max
      minAmount: { $min: "$amount" },
      maxAmount: { $max: "$amount" },
      
      // First/Last
      firstOrder: { $first: "$orderId" },
      lastOrder: { $last: "$orderId" },
      
      // Push to array
      allOrderIds: { $push: "$orderId" },
      
      // Add to set (unique values)
      uniqueProducts: { $addToSet: "$productId" },
      
      // Standard deviation
      stdDevAmount: { $stdDevPop: "$amount" }
    }
  }
])

// Group by multiple fields
db.orders.aggregate([
  { $group: {
      _id: {
        customerId: "$customerId",
        year: { $year: "$orderDate" },
        month: { $month: "$orderDate" }
      },
      totalAmount: { $sum: "$amount" }
    }
  }
])

// Group by null (aggregate all documents)
db.orders.aggregate([
  { $group: {
      _id: null,
      totalRevenue: { $sum: "$amount" },
      avgOrderValue: { $avg: "$amount" }
    }
  }
])
```

### 3. **$project - Reshaping Documents**

```javascript
db.users.aggregate([
  { $project: {
      name: 1,                           // Include field
      email: 1,
      age: 1,
      _id: 0,                            // Exclude _id
      
      // Computed fields
      ageGroup: {
        $cond: {
          if: { $gte: ["$age", 60] },
          then: "senior",
          else: {
            $cond: {
              if: { $gte: ["$age", 18] },
              then: "adult",
              else: "minor"
            }
          }
        }
      },
      
      // String concatenation
      fullName: {
        $concat: ["$firstName", " ", "$lastName"]
      },
      
      // Array size
      tagCount: { $size: "$tags" },
      
      // Date operations
      year: { $year: "$createdAt" },
      month: { $month: "$createdAt" },
      
      // Nested field extraction
      city: "$address.city"
    }
  }
])

// $addFields - Add fields without removing others
db.users.aggregate([
  { $addFields: {
      fullName: { $concat: ["$firstName", " ", "$lastName"] },
      ageInMonths: { $multiply: ["$age", 12] }
    }
  }
])
```

### 4. **$lookup - Joins**

```javascript
// Basic lookup (left outer join)
db.orders.aggregate([
  { $lookup: {
      from: "customers",               // Collection to join
      localField: "customerId",        // Field in orders
      foreignField: "_id",             // Field in customers
      as: "customerDetails"            // Output array
    }
  }
])

// Unwind lookup result (one-to-one relationship)
db.orders.aggregate([
  { $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
    }
  },
  { $unwind: "$customer" },  // Convert array to object
  { $project: {
      orderId: 1,
      amount: 1,
      customerName: "$customer.name",
      customerEmail: "$customer.email"
    }
  }
])

// Advanced lookup with pipeline
db.orders.aggregate([
  { $lookup: {
      from: "customers",
      let: { customerId: "$customerId" },
      pipeline: [
        { $match: {
            $expr: { $eq: ["$_id", "$$customerId"] }
          }
        },
        { $project: { name: 1, email: 1, _id: 0 } }
      ],
      as: "customer"
    }
  },
  { $unwind: "$customer" }
])

// Multiple lookups
db.orders.aggregate([
  { $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
    }
  },
  { $lookup: {
      from: "products",
      localField: "productId",
      foreignField: "_id",
      as: "product"
    }
  },
  { $unwind: "$customer" },
  { $unwind: "$product" }
])
```

### 5. **$unwind - Array Deconstruction**

```javascript
// Document with array
// { _id: 1, user: "john", tags: ["a", "b", "c"] }

db.users.aggregate([
  { $unwind: "$tags" }
])

// Result:
// { _id: 1, user: "john", tags: "a" }
// { _id: 1, user: "john", tags: "b" }
// { _id: 1, user: "john", tags: "c" }

// Preserve null and empty arrays
db.users.aggregate([
  { $unwind: {
      path: "$tags",
      preserveNullAndEmptyArrays: true
    }
  }
])

// Include array index
db.users.aggregate([
  { $unwind: {
      path: "$tags",
      includeArrayIndex: "tagIndex"
    }
  }
])
```

### 6. **Advanced Aggregation Patterns**

```javascript
// Faceted search - multiple aggregations in one query
db.products.aggregate([
  { $facet: {
      // Price ranges
      priceRanges: [
        { $bucket: {
            groupBy: "$price",
            boundaries: [0, 50, 100, 200, 500],
            default: "other",
            output: { count: { $sum: 1 } }
          }
        }
      ],
      
      // Categories
      categories: [
        { $group: {
            _id: "$category",
            count: { $sum: 1 }
          }
        },
        { $sort: { count: -1 } }
      ],
      
      // Top products
      topProducts: [
        { $sort: { sales: -1 } },
        { $limit: 5 },
        { $project: { name: 1, price: 1, sales: 1 } }
      ]
    }
  }
])

// Time-series aggregation
db.sales.aggregate([
  { $match: {
      date: {
        $gte: ISODate("2024-01-01"),
        $lt: ISODate("2024-02-01")
      }
    }
  },
  { $group: {
      _id: {
        year: { $year: "$date" },
        month: { $month: "$date" },
        day: { $dayOfMonth: "$date" }
      },
      dailyRevenue: { $sum: "$amount" },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1, "_id.day": 1 } }
])

// Running total (window functions - MongoDB 5.0+)
db.sales.aggregate([
  { $setWindowFields: {
      partitionBy: "$customerId",
      sortBy: { date: 1 },
      output: {
        runningTotal: {
          $sum: "$amount",
          window: {
            documents: ["unbounded", "current"]
          }
        }
      }
    }
  }
])

// Moving average
db.metrics.aggregate([
  { $setWindowFields: {
      sortBy: { timestamp: 1 },
      output: {
        movingAvg: {
          $avg: "$value",
          window: {
            documents: [-4, 0]  // Last 5 values including current
          }
        }
      }
    }
  }
])
```

---

## Indexing Strategies

### 1. **Basic Indexes**

```javascript
// Single field index
db.users.createIndex({ email: 1 })  // 1 = ascending, -1 = descending

// Check indexes
db.users.getIndexes()

// Drop index
db.users.dropIndex({ email: 1 })
db.users.dropIndex("email_1")  // By name

// Compound index (multiple fields)
db.users.createIndex({ lastName: 1, firstName: 1 })

// Index options
db.users.createIndex(
  { email: 1 },
  {
    unique: true,                    // Unique constraint
    sparse: true,                    // Only index non-null values
    name: "email_unique_index",      // Custom name
    partialFilterExpression: {       // Partial index
      age: { $gte: 18 }
    }
  }
)

// TTL index (auto-delete after time)
db.sessions.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 3600 }  // Delete after 1 hour
)

// Text index (full-text search)
db.articles.createIndex(
  { title: "text", content: "text" },
  { weights: { title: 10, content: 5 } }  // Title 2x more important
)

// Geospatial index
db.places.createIndex({ location: "2dsphere" })
```

### 2. **Index Strategies**

```javascript
// ESR Rule: Equality, Sort, Range
// Best practice: equality fields first, sort fields next, range fields last

// ❌ BAD
db.users.createIndex({ age: 1, city: 1, lastName: 1 })

// ✅ GOOD
db.users.createIndex({ city: 1, lastName: 1, age: 1 })

// For query:
db.users.find({ 
  city: "New York",      // Equality
  age: { $gt: 25 }       // Range
}).sort({ lastName: 1 }) // Sort

// Covered query - query uses only index (very fast)
db.users.createIndex({ email: 1, name: 1, age: 1 })

db.users.find(
  { email: "john@example.com" },
  { email: 1, name: 1, age: 1, _id: 0 }  // Must exclude _id or include in index
)

// Multikey index (index array values)
db.users.createIndex({ tags: 1 })

// Query can use index
db.users.find({ tags: "premium" })
```

### 3. **Index Performance Analysis**

```javascript
// Explain query execution
db.users.find({ email: "john@example.com" }).explain()

// Execution stats
db.users.find({ email: "john@example.com" }).explain("executionStats")

// All plans
db.users.find({ email: "john@example.com" }).explain("allPlansExecution")

// Read explain output:
/*
Key fields:
- stage: IXSCAN (index scan) = GOOD, COLLSCAN (collection scan) = BAD
- executionTimeMillis: Total execution time
- totalKeysExamined: Keys scanned
- totalDocsExamined: Documents scanned
- nReturned: Documents returned

Ideal ratio: nReturned ≈ totalDocsExamined ≈ totalKeysExamined
*/

// Force index usage (testing)
db.users.find({ email: "john@example.com" }).hint({ email: 1 })

// Analyze index usage
db.users.aggregate([
  { $indexStats: {} }
])
```

---

## Query Optimization

### 1. **Query Performance Patterns**

```javascript
// ❌ BAD: $where operator (executes JavaScript, very slow)
db.users.find({
  $where: "this.age > 25 && this.status == 'active'"
})

// ✅ GOOD: Native operators
db.users.find({
  age: { $gt: 25 },
  status: "active"
})

// ❌ BAD: Regex without anchor
db.users.find({ email: { $regex: /example/ } })

// ✅ GOOD: Regex with anchor (can use index)
db.users.find({ email: { $regex: /^john/ } })

// ❌ BAD: Negation queries (can't use index efficiently)
db.users.find({ status: { $ne: "inactive" } })

// ✅ GOOD: Positive query
db.users.find({ status: { $in: ["active", "pending"] } })

// ❌ BAD: $exists on large collection
db.users.find({ optionalField: { $exists: true } })

// ✅ GOOD: Add field to all documents, query specific value
db.users.updateMany(
  { optionalField: { $exists: false } },
  { $set: { optionalField: null } }
)
db.users.find({ optionalField: { $ne: null } })
```

### 2. **Pagination Best Practices**

```javascript
// ❌ BAD: Skip/limit for large offsets (slow)
db.users.find().skip(10000).limit(10)  // Scans 10,010 documents!

// ✅ GOOD: Range-based pagination (cursor-based)
// First page
db.users.find().sort({ _id: 1 }).limit(10)

// Next page (use last _id from previous page)
db.users.find({ _id: { $gt: lastId } }).sort({ _id: 1 }).limit(10)

// With custom sort field
db.users.createIndex({ createdAt: 1, _id: 1 })

// First page
db.users.find().sort({ createdAt: 1, _id: 1 }).limit(10)

// Next page
db.users.find({
  $or: [
    { createdAt: { $gt: lastCreatedAt } },
    { 
      createdAt: lastCreatedAt,
      _id: { $gt: lastId }
    }
  ]
}).sort({ createdAt: 1, _id: 1 }).limit(10)
```

### 3. **Aggregation Optimization**

```javascript
// Put $match early in pipeline
// ✅ GOOD
db.orders.aggregate([
  { $match: { status: "completed" } },  // Filter first
  { $lookup: { /* ... */ } },           // Then join
  { $group: { /* ... */ } }             // Then group
])

// ❌ BAD
db.orders.aggregate([
  { $lookup: { /* ... */ } },           // Join all documents
  { $match: { status: "completed" } },  // Filter after join
  { $group: { /* ... */ } }
])

// Use $project to reduce document size early
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $project: { customerId: 1, amount: 1 } },  // Keep only needed fields
  { $group: {
      _id: "$customerId",
      total: { $sum: "$amount" }
    }
  }
])

// Use allowDiskUse for large aggregations
db.orders.aggregate(
  [ /* pipeline */ ],
  { allowDiskUse: true }  // Use disk for sorting/grouping
)
```

---

## Spring Data MongoDB

### 1. **Repository Pattern**

```java
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.mongodb.repository.Query;
import org.springframework.data.domain.*;

// Entity
@Document(collection = "users")
public class User {
    @Id
    private String id;
    
    @Indexed(unique = true)
    private String email;
    
    private String name;
    private Integer age;
    private String city;
    
    @Field("created_at")
    private Instant createdAt;
    
    private List<String> tags;
    
    // Getters, setters, constructors
}

// Repository interface
public interface UserRepository extends MongoRepository<User, String> {
    
    // Query method naming convention
    List<User> findByAge(Integer age);
    
    List<User> findByAgeGreaterThan(Integer age);
    
    List<User> findByAgeAndCity(Integer age, String city);
    
    List<User> findByAgeOrCity(Integer age, String city);
    
    List<User> findByAgeBetween(Integer minAge, Integer maxAge);
    
    List<User> findByNameLike(String name);
    
    List<User> findByNameStartingWith(String prefix);
    
    List<User> findByTagsContaining(String tag);
    
    List<User> findByCreatedAtBefore(Instant date);
    
    // Sorting
    List<User> findByAgeOrderByNameAsc(Integer age);
    
    List<User> findByCity(String city, Sort sort);
    
    // Pagination
    Page<User> findByCity(String city, Pageable pageable);
    
    Slice<User> findByAge(Integer age, Pageable pageable);
    
    // Top/First
    List<User> findTop10ByCity(String city);
    
    User findFirstByEmailOrderByCreatedAtDesc(String email);
    
    // Count/Exists
    long countByCity(String city);
    
    boolean existsByEmail(String email);
    
    // Delete
    long deleteByAge(Integer age);
    
    void removeByCity(String city);
}
```

### 2. **Custom Queries**

```java
public interface UserRepository extends MongoRepository<User, String> {
    
    // @Query with JSON
    @Query("{ 'age': { $gte: ?0, $lte: ?1 } }")
    List<User> findByAgeRange(Integer minAge, Integer maxAge);
    
    // @Query with projection
    @Query(value = "{ 'city': ?0 }", fields = "{ 'name': 1, 'email': 1 }")
    List<User> findNameAndEmailByCity(String city);
    
    // @Query with sort
    @Query(value = "{ 'age': { $gt: ?0 } }", sort = "{ 'age': -1 }")
    List<User> findByAgeGreaterThanSorted(Integer age);
    
    // Complex query
    @Query("""
        {
          $or: [
            { age: { $lt: ?0 } },
            { age: { $gt: ?1 } }
          ],
          status: 'active'
        }
        """)
    List<User> findActiveUsersOutsideAgeRange(Integer minAge, Integer maxAge);
    
    // Aggregation
    @Aggregation(pipeline = {
        "{ $match: { 'city': ?0 } }",
        "{ $group: { _id: '$age', count: { $sum: 1 } } }",
        "{ $sort: { count: -1 } }"
    })
    List<AgeGroupCount> getAgeDistribution(String city);
}

// Result DTO for aggregation
public record AgeGroupCount(Integer age, Long count) {}
```

### 3. **MongoTemplate (Advanced Queries)**

```java
@Service
public class UserService {
    
    private final MongoTemplate mongoTemplate;
    
    public UserService(MongoTemplate mongoTemplate) {
        this.mongoTemplate = mongoTemplate;
    }
    
    // Query with Criteria
    public List<User> findUsers(String city, Integer minAge) {
        Query query = new Query();
        
        query.addCriteria(
            Criteria.where("city").is(city)
                .and("age").gte(minAge)
        );
        
        return mongoTemplate.find(query, User.class);
    }
    
    // Complex criteria
    public List<User> findUsersWithComplexCriteria() {
        Criteria criteria = new Criteria().orOperator(
            Criteria.where("age").lt(25),
            Criteria.where("age").gt(60)
        ).and("status").is("active");
        
        Query query = new Query(criteria);
        query.with(Sort.by(Sort.Direction.DESC, "age"));
        query.limit(10);
        
        return mongoTemplate.find(query, User.class);
    }
    
    // Aggregation with MongoTemplate
    public List<AgeGroupCount> getAgeDistribution(String city) {
        Aggregation aggregation = Aggregation.newAggregation(
            Aggregation.match(Criteria.where("city").is(city)),
            Aggregation.group("age").count().as("count"),
            Aggregation.sort(Sort.Direction.DESC, "count")
        );
        
        AggregationResults<AgeGroupCount> results = mongoTemplate.aggregate(
            aggregation,
            "users",
            AgeGroupCount.class
        );
        
        return results.getMappedResults();
    }
    
    // Update with MongoTemplate
    public void updateUserAge(String email, Integer newAge) {
        Query query = new Query(Criteria.where("email").is(email));
        Update update = new Update().set("age", newAge);
        
        mongoTemplate.updateFirst(query, update, User.class);
    }
    
    // Upsert
    public void upsertUser(User user) {
        Query query = new Query(Criteria.where("email").is(user.getEmail()));
        
        Update update = new Update()
            .set("name", user.getName())
            .set("age", user.getAge())
            .setOnInsert("createdAt", Instant.now());
        
        mongoTemplate.upsert(query, update, User.class);
    }
    
    // FindAndModify
    public User findAndUpdateLoginCount(String email) {
        Query query = new Query(Criteria.where("email").is(email));
        Update update = new Update().inc("loginCount", 1);
        
        FindAndModifyOptions options = new FindAndModifyOptions()
            .returnNew(true)
            .upsert(false);
        
        return mongoTemplate.findAndModify(
            query,
            update,
            options,
            User.class
        );
    }
    
    // Bulk operations
    public void bulkUpdateUsers(List<User> users) {
        BulkOperations bulkOps = mongoTemplate.bulkOps(
            BulkOperations.BulkMode.UNORDERED,
            User.class
        );
        
        for (User user : users) {
            Query query = new Query(Criteria.where("email").is(user.getEmail()));
            Update update = new Update()
                .set("name", user.getName())
                .set("age", user.getAge());
            
            bulkOps.updateOne(query, update);
        }
        
        BulkWriteResult result = bulkOps.execute();
        System.out.println("Modified: " + result.getModifiedCount());
    }
    
    // Geo queries
    public List<Place> findNearby(double longitude, double latitude, double maxDistance) {
        Point location = new Point(longitude, latitude);
        
        NearQuery nearQuery = NearQuery.near(location)
            .maxDistance(new Distance(maxDistance, Metrics.KILOMETERS));
        
        GeoResults<Place> results = mongoTemplate.geoNear(
            nearQuery,
            Place.class
        );
        
        return results.getContent()
            .stream()
            .map(GeoResult::getContent)
            .collect(Collectors.toList());
    }
}
```

### 4. **Reactive MongoDB**

```java
// Reactive repository
public interface ReactiveUserRepository extends ReactiveMongoRepository<User, String> {
    
    Flux<User> findByCity(String city);
    
    Mono<User> findByEmail(String email);
    
    Flux<User> findByAgeGreaterThan(Integer age);
}

// Reactive service
@Service
public class ReactiveUserService {
    
    private final ReactiveUserRepository userRepository;
    
    public Mono<User> createUser(User user) {
        user.setCreatedAt(Instant.now());
        return userRepository.save(user);
    }
    
    public Flux<User> findUsersByCity(String city) {
        return userRepository.findByCity(city);
    }
    
    public Mono<User> updateUser(String id, User updates) {
        return userRepository.findById(id)
            .flatMap(existingUser -> {
                existingUser.setName(updates.getName());
                existingUser.setAge(updates.getAge());
                return userRepository.save(existingUser);
            });
    }
}
```

---

## Transactions

### 1. **Multi-Document Transactions**

```javascript
// Start a session
const session = db.getMongo().startSession()

// Start transaction
session.startTransaction()

try {
  // Operations within transaction
  db.accounts.updateOne(
    { _id: "account1" },
    { $inc: { balance: -100 } },
    { session }
  )
  
  db.accounts.updateOne(
    { _id: "account2" },
    { $inc: { balance: 100 } },
    { session }
  )
  
  // Commit transaction
  session.commitTransaction()
} catch (error) {
  // Abort on error
  session.abortTransaction()
  throw error
} finally {
  session.endSession()
}
```

### 2. **Spring Data MongoDB Transactions**

```java
@Service
public class TransferService {
    
    private final MongoTemplate mongoTemplate;
    private final MongoTransactionManager transactionManager;
    
    @Transactional
    public void transfer(String fromAccount, String toAccount, BigDecimal amount) {
        Query fromQuery = new Query(Criteria.where("_id").is(fromAccount));
        Update debit = new Update().inc("balance", amount.negate());
        mongoTemplate.updateFirst(fromQuery, debit, Account.class);
        
        Query toQuery = new Query(Criteria.where("_id").is(toAccount));
        Update credit = new Update().inc("balance", amount);
        mongoTemplate.updateFirst(toQuery, credit, Account.class);
        
        // If exception thrown, transaction rolls back automatically
    }
    
    // Programmatic transaction
    public void transferProgrammatic(String fromAccount, String toAccount, BigDecimal amount) {
        TransactionTemplate transactionTemplate = new TransactionTemplate(transactionManager);
        
        transactionTemplate.execute(status -> {
            try {
                // Operations
                Query fromQuery = new Query(Criteria.where("_id").is(fromAccount));
                Update debit = new Update().inc("balance", amount.negate());
                mongoTemplate.updateFirst(fromQuery, debit, Account.class);
                
                Query toQuery = new Query(Criteria.where("_id").is(toAccount));
                Update credit = new Update().inc("balance", amount);
                mongoTemplate.updateFirst(toQuery, credit, Account.class);
                
                return null;
            } catch (Exception e) {
                status.setRollbackOnly();
                throw e;
            }
        });
    }
}
```

---

## Change Streams

### 1. **Basic Change Streams**

```javascript
// Watch all changes
const changeStream = db.users.watch()

changeStream.on("change", (change) => {
  console.log("Change detected:", change)
})

// Watch specific operations
db.users.watch([
  { $match: { operationType: "insert" } }
])

// Watch specific fields
db.users.watch([
  { $match: { "updateDescription.updatedFields.status": { $exists: true } } }
])
```

### 2. **Spring Data MongoDB Change Streams**

```java
@Service
public class UserChangeListener {
    
    private final ReactiveMongoTemplate mongoTemplate;
    
    @PostConstruct
    public void watchUserChanges() {
        ChangeStreamOptions options = ChangeStreamOptions.builder()
            .filter(Aggregation.newAggregation(
                Aggregation.match(Criteria.where("operationType").in("insert", "update"))
            ))
            .build();
        
        Flux<ChangeStreamEvent<User>> changeStream = mongoTemplate.changeStream(
            "users",
            options,
            User.class
        );
        
        changeStream.subscribe(event -> {
            System.out.println("Operation: " + event.getOperationType());
            System.out.println("Document: " + event.getBody());
            
            if (event.getOperationType() == OperationType.INSERT) {
                User user = event.getBody();
                sendWelcomeEmail(user);
            }
        });
    }
}
```

---

## Schema Design Patterns

### 1. **Embedding vs Referencing**

```javascript
// ✅ GOOD: Embedding (one-to-few, data read together)
{
  _id: ObjectId("..."),
  name: "John Doe",
  email: "john@example.com",
  address: {
    street: "123 Main St",
    city: "New York",
    zip: "10001"
  },
  phoneNumbers: [
    { type: "home", number: "555-1234" },
    { type: "work", number: "555-5678" }
  ]
}

// ✅ GOOD: Referencing (one-to-many, data changes frequently)
// User document
{
  _id: ObjectId("user123"),
  name: "John Doe",
  email: "john@example.com"
}

// Order documents (many)
{
  _id: ObjectId("order1"),
  userId: ObjectId("user123"),
  items: [...],
  total: 99.99
}
```

### 2. **Extended Reference Pattern**

```javascript
// Denormalize frequently accessed fields
{
  _id: ObjectId("order1"),
  userId: ObjectId("user123"),
  
  // Denormalized user info (for display)
  userInfo: {
    name: "John Doe",
    email: "john@example.com"
  },
  
  items: [...],
  total: 99.99
}
```

### 3. **Bucket Pattern**

```javascript
// Instead of one document per measurement
// ❌ BAD
{ sensor: "A", temp: 20, time: ISODate("2024-01-15T10:00:00Z") }
{ sensor: "A", temp: 21, time: ISODate("2024-01-15T10:01:00Z") }
// ... thousands of documents

// ✅ GOOD: Bucket pattern
{
  sensor: "A",
  date: ISODate("2024-01-15"),
  measurements: [
    { temp: 20, time: ISODate("2024-01-15T10:00:00Z") },
    { temp: 21, time: ISODate("2024-01-15T10:01:00Z") },
    // ... up to reasonable size
  ]
}
```

---

## Performance Best Practices

### ✅ DO

1. **Create indexes** for frequent queries
2. **Use projections** to return only needed fields
3. **Use covered queries** (query uses only index)
4. **Use range-based pagination** (not skip/limit)
5. **Put $match early** in aggregation pipeline
6. **Use compound indexes** following ESR rule
7. **Monitor slow queries** (enable profiling)
8. **Use connection pooling** (default in drivers)
9. **Batch operations** when possible
10. **Use appropriate read preference** (primary, secondary)

### ❌ DON'T

1. **Don't use $where** (executes JavaScript)
2. **Don't retrieve entire documents** if not needed
3. **Don't use skip for large offsets** (use range queries)
4. **Don't create too many indexes** (impacts writes)
5. **Don't use regex without anchor** on large collections
6. **Don't use $exists** without index
7. **Don't fetch and loop** (use aggregation)
8. **Don't ignore explain plans**
9. **Don't use transactions unnecessarily** (overhead)
10. **Don't store large arrays** (> 1000 elements)

---

## Common Pitfalls

```javascript
// ❌ PITFALL 1: N+1 queries
// Bad: Fetch users then fetch orders for each
users.forEach(user => {
  const orders = db.orders.find({ userId: user._id })
})

// ✅ GOOD: Use $lookup (aggregation join)
db.users.aggregate([
  { $lookup: {
      from: "orders",
      localField: "_id",
      foreignField: "userId",
      as: "orders"
    }
  }
])

// ❌ PITFALL 2: Growing arrays
// Bad: Unbounded array growth
db.users.updateOne(
  { _id: userId },
  { $push: { events: event } }  // Array grows forever!
)

// ✅ GOOD: Limit array size
db.users.updateOne(
  { _id: userId },
  {
    $push: {
      events: {
        $each: [event],
        $slice: -100  // Keep only last 100
      }
    }
  }
)

// ❌ PITFALL 3: Case-sensitive queries
db.users.find({ email: "John@Example.com" })  // Won't match "john@example.com"

// ✅ GOOD: Case-insensitive index
db.users.createIndex(
  { email: 1 },
  { collation: { locale: "en", strength: 2 } }
)

// Or use regex
db.users.find({ email: { $regex: /^john@example\.com$/i } })

// ❌ PITFALL 4: Large documents in memory
db.users.find().toArray()  // Loads everything in memory!

// ✅ GOOD: Use cursor
const cursor = db.users.find()
while (cursor.hasNext()) {
  const user = cursor.next()
  process(user)
}
```

---

## Conclusion

**Key Takeaways:**

1. **Use appropriate queries** - Match, aggregate, or find based on need
2. **Index strategically** - Follow ESR rule, analyze with explain
3. **Optimize aggregations** - $match early, reduce document size
4. **Design schema wisely** - Embed vs reference based on access patterns
5. **Monitor performance** - Enable profiling, analyze slow queries
6. **Use Spring Data** - Leverage repositories for common patterns
7. **Understand trade-offs** - Denormalization vs consistency

**MongoDB Strengths:**
- Flexible schema
- Rich query language
- Powerful aggregation
- Horizontal scaling
- Real-time analytics

**Remember**: MongoDB is not a silver bullet. Choose the right tool for your use case!
