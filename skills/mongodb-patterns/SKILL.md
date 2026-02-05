---
name: mongodb-patterns
description: MongoDB schema design, queries, and best practices for production
---

# MongoDB Patterns Skill

## Overview

This skill provides guidelines for designing, querying, and optimizing MongoDB databases for production applications.

## Schema Design

### 1. Document Design Principles

```javascript
// DO: Embed when data is accessed together
// User with embedded addresses
{
  _id: ObjectId("..."),
  email: "user@example.com",
  profile: {
    firstName: "John",
    lastName: "Doe",
    avatar: "https://..."
  },
  addresses: [
    {
      type: "shipping",
      street: "123 Main St",
      city: "New York",
      isDefault: true
    },
    {
      type: "billing",
      street: "456 Oak Ave",
      city: "Boston"
    }
  ],
  preferences: {
    newsletter: true,
    notifications: {
      email: true,
      sms: false
    }
  }
}

// DO: Reference when data grows large or is accessed independently
// Order with referenced user
{
  _id: ObjectId("..."),
  userId: ObjectId("user-id"),
  items: [
    {
      productId: ObjectId("product-1"),
      name: "Widget",  // Denormalized for quick display
      price: 29.99,
      quantity: 2
    }
  ],
  total: 59.98,
  status: "pending",
  createdAt: ISODate("2024-01-15T10:30:00Z")
}
```

### 2. Schema Validation

```javascript
// DO: Use schema validation at collection level
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "createdAt"],
      properties: {
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
          description: "Must be a valid email address"
        },
        age: {
          bsonType: "int",
          minimum: 0,
          maximum: 150
        },
        role: {
          enum: ["user", "admin", "moderator"]
        },
        addresses: {
          bsonType: "array",
          items: {
            bsonType: "object",
            required: ["street", "city"],
            properties: {
              type: {
                enum: ["shipping", "billing"]
              }
            }
          }
        }
      }
    }
  },
  validationLevel: "strict",
  validationAction: "error"
});
```

### 3. Indexing

```javascript
// DO: Single field indexes
db.users.createIndex({ email: 1 }, { unique: true });
db.users.createIndex({ "profile.username": 1 }, { unique: true });

// DO: Compound indexes
db.orders.createIndex({ userId: 1, createdAt: -1 });
db.products.createIndex({ category: 1, price: 1 });

// DO: Multikey indexes for arrays
db.users.createIndex({ "addresses.city": 1 });
db.posts.createIndex({ tags: 1 });

// DO: Text indexes for search
db.products.createIndex(
  { name: "text", description: "text" },
  { weights: { name: 10, description: 5 } }
);

// DO: Hashed indexes for sharding
db.users.createIndex({ _id: "hashed" });

// DO: TTL indexes for data expiration
db.sessions.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 3600 }
);

// DO: Partial indexes
db.orders.createIndex(
  { status: 1 },
  { partialFilterExpression: { status: { $in: ["pending", "processing"] } } }
);

// DO: Sparse indexes
db.users.createIndex(
  { phoneNumber: 1 },
  { sparse: true }
);

// DON'T: Create too many indexes
// Each index slows down writes
// Use explain() to verify index usage
```

## Query Patterns

### 1. CRUD Operations

```javascript
// DO: Create with validation
const newUser = {
  email: "user@example.com",
  profile: {
    firstName: "John",
    lastName: "Doe"
  },
  createdAt: new Date()
};

try {
  const result = await db.users.insertOne(newUser);
  console.log(`Created user with id: ${result.insertedId}`);
} catch (error) {
  if (error.code === 11000) {
    console.error("Duplicate email");
  }
}

// DO: Read with projection
const user = await db.users.findOne(
  { _id: ObjectId("...") },
  { projection: { email: 1, "profile.firstName": 1 } }
);

// DO: Find with sorting and pagination
const users = await db.users
  .find({ isActive: true })
  .sort({ createdAt: -1 })
  .skip(20)
  .limit(10)
  .toArray();

// DO: Update with operators
await db.users.updateOne(
  { _id: ObjectId("...") },
  {
    $set: { "profile.lastName": "Smith", updatedAt: new Date() },
    $inc: { loginCount: 1 },
    $push: { 
      loginHistory: { 
        $each: [{ timestamp: new Date(), ip: "192.168.1.1" }],
        $slice: -10 // Keep only last 10
      }
    },
    $addToSet: { tags: "premium" } // Add if not exists
  }
);

// DO: Update with aggregation pipeline
await db.orders.updateOne(
  { _id: ObjectId("...") },
  [
    {
      $set: {
        total: {
          $sum: {
            $map: {
              input: "$items",
              as: "item",
              in: { $multiply: ["$$item.price", "$$item.quantity"] }
            }
          }
        }
      }
    }
  ]
);

// DO: Delete with care
await db.sessions.deleteMany({
  createdAt: { $lt: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000) }
});

// DO: Find and modify (atomic)
const updatedUser = await db.users.findOneAndUpdate(
  { _id: ObjectId("...") },
  { $inc: { credits: -10 } },
  { 
    returnDocument: "after",
    projection: { credits: 1 }
  }
);
```

### 2. Aggregation Pipeline

```javascript
// DO: Use aggregation for complex queries
// User statistics with order counts
const userStats = await db.users.aggregate([
  // Match stage
  { $match: { isActive: true } },
  
  // Lookup (join) with orders
  {
    $lookup: {
      from: "orders",
      localField: "_id",
      foreignField: "userId",
      as: "orders"
    }
  },
  
  // Project to shape output
  {
    $project: {
      email: 1,
      fullName: { 
        $concat: ["$profile.firstName", " ", "$profile.lastName"] 
      },
      orderCount: { $size: "$orders" },
      totalSpent: {
        $sum: "$orders.total"
      }
    }
  },
  
  // Sort and limit
  { $sort: { totalSpent: -1 } },
  { $limit: 10 }
]).toArray();

// DO: Use unwind for array processing
const productStats = await db.orders.aggregate([
  { $unwind: "$items" },
  {
    $group: {
      _id: "$items.productId",
      totalQuantity: { $sum: "$items.quantity" },
      totalRevenue: {
        $sum: { $multiply: ["$items.price", "$items.quantity"] }
      },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { totalRevenue: -1 } }
]).toArray();

// DO: Use facet for multiple aggregations
const dashboardData = await db.orders.aggregate([
  {
    $facet: {
      totalStats: [
        {
          $group: {
            _id: null,
            totalRevenue: { $sum: "$total" },
            orderCount: { $sum: 1 },
            avgOrderValue: { $avg: "$total" }
          }
        }
      ],
      dailyStats: [
        {
          $group: {
            _id: { 
              $dateToString: { format: "%Y-%m-%d", date: "$createdAt" } 
            },
            revenue: { $sum: "$total" },
            count: { $sum: 1 }
          }
        },
        { $sort: { _id: -1 } },
        { $limit: 30 }
      ],
      statusBreakdown: [
        { $group: { _id: "$status", count: { $sum: 1 } } }
      ]
    }
  }
]).toArray();
```

### 3. Transactions

```javascript
// DO: Use transactions for multi-document operations
const session = client.startSession();

try {
  await session.withTransaction(async () => {
    // Deduct from user balance
    await db.users.updateOne(
      { _id: userId, balance: { $gte: amount } },
      { $inc: { balance: -amount } },
      { session }
    );
    
    // Create order
    const orderResult = await db.orders.insertOne(orderData, { session });
    
    // Update inventory
    for (const item of items) {
      await db.products.updateOne(
        { 
          _id: item.productId,
          stock: { $gte: item.quantity }
        },
        { $inc: { stock: -item.quantity } },
        { session }
      );
    }
    
    // Create audit log
    await db.auditLogs.insertOne({
      action: "order_created",
      orderId: orderResult.insertedId,
      userId,
      timestamp: new Date()
    }, { session });
  });
  
  console.log("Transaction committed");
} catch (error) {
  console.error("Transaction failed:", error);
  throw error;
} finally {
  await session.endSession();
}
```

## Advanced Patterns

### 1. Change Streams

```javascript
// DO: Use change streams for real-time updates
const changeStream = db.orders.watch([
  { 
    $match: { 
      "fullDocument.status": "completed" 
    } 
  }
]);

changeStream.on("change", (change) => {
  console.log("Order completed:", change.fullDocument);
  // Send notification, update cache, etc.
});

// DO: Resume after interruption
const resumeToken = await loadLastToken(); // From persistent storage
const changeStream = db.orders.watch([], {
  resumeAfter: resumeToken
});

changeStream.on("change", async (change) => {
  await saveToken(change._id); // Save token for resume
  // Process change
});
```

### 2. Full-Text Search

```javascript
// DO: Create text index
db.articles.createIndex({ title: "text", content: "text" });

// DO: Search with text score
const results = await db.articles
  .find(
    { $text: { $search: "mongodb optimization" } },
    { 
      score: { $meta: "textScore" },
      projection: { title: 1, score: { $meta: "textScore" } }
    }
  )
  .sort({ score: { $meta: "textScore" } })
  .limit(10)
  .toArray();

// DO: Atlas Search (if using MongoDB Atlas)
const searchResults = await db.articles.aggregate([
  {
    $search: {
      index: "default",
      text: {
        query: "mongodb",
        path: ["title", "content"],
        fuzzy: { maxEdits: 1 }
      }
    }
  },
  { $limit: 10 }
]).toArray();
```

### 3. GridFS for Large Files

```javascript
// DO: Use GridFS for files >16MB
const { GridFSBucket } = require('mongodb');
const bucket = new GridFSBucket(db);

// Upload file
const uploadStream = bucket.openUploadStream("report.pdf", {
  metadata: { userId: "user-123", type: "report" }
});
fs.createReadStream("./report.pdf").pipe(uploadStream);

// Download file
const downloadStream = bucket.openDownloadStreamByName("report.pdf");
downloadStream.pipe(fs.createWriteStream("./downloaded.pdf"));

// Query files
const files = await bucket.find({ "metadata.userId": "user-123" }).toArray();
```

## Performance Optimization

### 1. Query Optimization

```javascript
// DO: Use explain for query analysis
const explanation = await db.users.find({ email: "test@example.com" }).explain("executionStats");
console.log(explanation.executionStats.totalDocsExamined);

// DO: Covered queries (projection includes only indexed fields)
const user = await db.users.findOne(
  { email: "test@example.com" },
  { projection: { _id: 0, email: 1 } } // Only indexed fields
);

// DO: Use hint when necessary
const results = await db.orders
  .find({ status: "pending" })
  .hint({ status: 1, createdAt: -1 })
  .toArray();
```

### 2. Connection Management

```javascript
// DO: Use connection pooling
const client = new MongoClient(uri, {
  maxPoolSize: 100,
  minPoolSize: 10,
  maxIdleTimeMS: 30000,
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000,
});

// DO: Handle connection events
client.on("serverOpening", () => console.log("MongoDB connection opened"));
client.on("serverClosed", () => console.log("MongoDB connection closed"));
client.on("error", (err) => console.error("MongoDB error:", err));
```

### 3. Monitoring

```javascript
// DO: Enable profiling for slow queries
db.setProfilingLevel(1, { slowms: 100 });

// View slow queries
const slowQueries = await db.system.profile.find().sort({ ts: -1 }).limit(10).toArray();

// DO: Check index usage
const indexStats = await db.users.aggregate([
  { $indexStats: {} }
]).toArray();

// DO: Monitor collection stats
const stats = await db.users.stats();
console.log(`Document count: ${stats.count}`);
console.log(`Index size: ${stats.totalIndexSize}`);
console.log(`Avg document size: ${stats.avgObjSize}`);
```

## When to Use

Use this skill when:
- Designing MongoDB schemas
- Writing MongoDB queries
- Optimizing query performance
- Implementing transactions
- Working with aggregation pipelines
- Setting up full-text search
- Designing for scalability
