# MongoDB: most important Queries, Updates and Aggregration

---

## Queries

MongoDB has its own JSON-like query language

You can perform ad hoc queries on the database using the find or findOne functions

**Query selectors** can be used to match documents

- You can query for ranges, set inclusion, inequalities and more

**Query options** can be used to modify the result of the queries

- sorting, choosing which fields to include etc…

Note:

- Regular expressions can be used for queries and they can take advantage of indexes
- Every MongoDB query is fundamentally a instantiation of a cursor and fetching of that cursor’s result set

---

### Queries: Find method

The "find" method is used to perform queries <!-- .element: class="fragment" -->

Querying returns a subset of documents in a collection <!-- .element: class="fragment" -->

Takes a query document for filtering <!-- .element: class="fragment" -->

Empty query document matches everything in the collection <!-- .element: class="fragment" -->

---

### Queries: Find method

Example in C#:

```C#
FilterDefinition<Player> filter =
    Builders<Player>.Filter.Eq(p => p.Name, name);

Player player =
    await collection.Find(filter).FirstAsync();
```

---

### Query cursor

When you call find, the database is NOT queried immediately <!-- .element: class="fragment" -->

The database returns the results from "find" by using a cursor <!-- .element: class="fragment" -->

Cursor allows control of the eventual output of a query <!-- .element: class="fragment" -->

Almost every method on a cursor object returns the cursor itself <!-- .element: class="fragment" -->

- It's possible chain options in any order <!-- .element: class="fragment" -->

---

### Query cursor example

```C#
SortDefinition<Player> sortDef =
    Builders<Player>.Sort.Ascending("Level");

IFindFluent<Player, Player> cursor =
    collection.Find("")
        .Sort(sortDef)
        .Limit(1)
        .Skip(10);
```

Query is executed when results are requested from the cursor:

```C#
List<Player> players = await cursor.ToListAsync();
```

---

### Query selectors: Comparisons

**\$lt** == lower than (<)

**\$lte** == lower than or equal (<=)

**\$gt** == greater than (>)

**\$gte** == greater than or equal (>=)

Example in C#:

```C#
FilterDefinition<Player> filter =
    Builders<Player>.Filter.Gte("Level", 18)
    & Builders<Player>.Filter.Lte("Level", 30);

List<Player> players =
    await collection.Find(filter).ToListAsync();
```

Note:

- Range queries will match only if they have same type as the value to be compared against
- Generally you should not store more than one type for key within the same collection
- (e.g. db.players.find({score: {$gte: 100, $lte: 200}})

---

### Query selectors: Set operators

Each takes a list of one or more values as predicate

**\$all** – returns a document if all of the given values match the search key

**\$in** - returns a document if any of the given values matches the search key

**\$nin** – returns a document if none of the given values matches the search key

```C#
var filter =
    Builders<Player>.Filter.In(
        p => p.Level,
        new[] { 10, 20, 30 }
    );
```

Note:

- When using the set operators, keep in mind that $in and $all can take advantage of indexes, but $nin can’t and thus requires a collection scan. If you use $nin, try to use it in combination with another query term that does use an index.
- Make example of using set

---

### Query selectors: Boolean operators (1)

**\$ne** – not equal

**\$not** – negates the result of another MongoDB operator or regular expression query

**\$exists** – used for querying documents containing a particular key, needed because collections don’t enforce schema

Note:

- \$ne can not take advantage of indexes
- Do not use $not if there exists a negated form for the operator (e.g. $in and \$nin)
- If possible values are scoped to the same key, use \$in instead

---

### Query selectors: Boolean operators (2)

**\$or** – expresses a logical disjunction of two values for two different keys

**\$and** – both selectors match for the document

```C#
var f1 = Builders<Player>.Filter.Eq(p => p.Name, "John");
var f2 = Builders<Player>.Filter.Gt(p => p.Level, 10);
var andFilter = Builders<Player>.Filter.And(f1, f2);
```

Note:

- $and and $or take array of query selectors, where each selector can be arbitrarily complexand may itself contain other query operators.
- MongoDB interprets all query selectors containing more than one key by ANDing the conditions, you should use \$and only when you can’t express an AND in a simpler way.

---

### Query selectors: Javascript

If you can’t express your query with the tools described thus far, you have an option to write some JavaScript

**\$where** operator is used to pass a JavaScript expression

Adds substantial overhead because expressions need to be evaluated within a JavaScript interpreter context

JavaScript expressions can’t use an index

For security, use of "\$where" clauses should be highly restricted or eliminated

Note:

- Using where makes it possible to do JavaScript injection attacks – malicious users can’t write or delete this way but they might be able to read sensitive data

---

### Querying Arrays

Querying for matching value in array uses the same syntax as querying a single value

This, for example, would find a player with Tags “active” and “novice”

```C#
var filter =
    Builders<Player>.Filter.Eq(p => p.Tags, "active");

var player =
    _collection.Find(filter).FirstAsync();
```

**\$size** – allows to query for an array by its size

---

### Querying sub-documents

**\$elemMatch** can be used to perform queries on the subdocuments

```C#
var playersWithWeapons =
    Builders<Player>.Filter.ElemMatch<Item>(
        p => p.Items,
        Builders<Item>.Filter.Eq(
            i => i.ItemType,
            ItemType.Weapon
        )
    );
```

---

### Query cursor options: **Projections**

Query cursor options can be used to constrain the result set

Returns only specific fields of the document

```C#
var projection =
    Builders<Player>.Projection.As<PlayerWithLimitedProps>();

var limitedPlayer =
    await _collection.Find(filter).Project(projection).FirstAsync();
```

---

### Query cursor options: **Sorting**

Can be used for sorting any result in ascending or descending order

Can be used with one or more fields

```C#
SortDefinition<Player> sortDef =
    Builders<Player>.Sort.Ascending("Level");

IFindFluent<Player, Player> cursor =
    collection.Find("").Sort(sortDef);
```

---

### Query cursor options: **Skip and limit**

- Skip documents and limit documents returned by the query
- Beware of skipping large number of documents because it gets ineffective

```C#
IFindFluent<Player, Player> cursor =
    collection.Find("").Limit(1).Skip(10);
```

Note:

- Especially in cases where you have large documents, using a projection will minimize the costs of network latency and deserialization.

---

## Updates

---

### Updates

Documents can be replaced completely

Specific fields can be updated independently

Targeted updates generally give better performance

- Update documents are smaller
- No need to fetch the existing document

---

### Updates

By default update only updates the first document matched by its query selector

- There is an option to do multidocument updates as well

MongoDB supports upserts

- Upsert means doing an insert if the document doesn’t exist and doing update if it already exists

Note:

- It’s best to keep the documents small if you intend to do a lot of updates
- Updates that modify the structure and size of the document lead into rewriting the whole document (if the document size is several MBs, the updates will start getting costly)

---

### Standard update operators

**\$inc** – increment or decrement a numeric value

- Can be used with an upsert

**\$set** – set value of any particular key to any valid BSON type

**\$unset** – remove a key from the document

- For an array, the value is merely set to null

**\$rename** – change the name of a key

- Works with subdocuments as well

---

### Array update operators

**\$push** and **\$pushAll** – append value/list of values to an array

**\$addToSet** – only adds the value if it doesn’t exist in the array

- Can be used in combination with \$each to add multiple values

**\$pop** – Used for removing an item from an array based on position, does not return removed value

**\$pull** and **\$pullAll** – remove value/list of values by the value

Note:

- Since arrays are so central to MongoDB’s document model, there exists update operators just for arrays

---

### Atomic operations

**findAndModify** –command

Any one document can be updated atomically

Document structure in itself makes it possible to fit within a single document many things that might atomic processing

When updating and removing documents a special option called \$atomic can be used to make sure that the whole operation is performed before others read the touched documents

Note:

- Guarantee of consistent reads along a single connection
- Database permits only one writer or multiple readers at once (not both)
- Fetching data from disk yields the execution
- Multidocument operations yield the execution unless explicitly specified not to

---

## Aggregation

Summarizing and reframing the data

Note:

- Queries allow to get data as it’s stored -- Aggregations operations process data records and return computed results
- Aggregation operations group values from multiple documents together, and can perform a variety of operations on the grouped data to return a single result
- Aggregation = crunching

---

### Aggregation

Simplified process of aggregation works like this:

- Group values from multiple documents
- Perform a variety of operations on the values
- Return a single result

Three methods for aggregation:

**Aggregation pipeline**

**Map-reduce**

**Single purpose aggregation methods**

---

### Aggregation example description

Find out what are the most common level for the players

1. Project the levels out of each player document
2. Group the levels by the number, counting the number of occurrences
3. Sort the levels by the occurrence count, descending
4. Limit the results to the first three

---

### Aggregation example

```C#
var levelCounts =
    _collection.Aggregate()
        .Project(p => p.Level)
        .Group(l => l, p => new LevelCount { Id = p.Key, Count = p.Sum() })
        .SortByDescending(l => l.Count)
        .Limit(3);
```

Note:

- Project: The syntax is similar to the field selector used in querying: you can select fields to project by specifying " fieldname" : 1 or exclude fields with " fieldname" : 0. After this operation, each document in the results looks like: {"\_id" : id, "author" : " authorName"}. These resulting documents only exists in memory and are not written to disk anywhere
- Group: This groups the authors by name and increments "count" for each document an author appears in. First, we specify the field we want to group by, which is "author" . This is indicated by the "\_id" : "$author" field. You can picture this as: after the group there will be one result document per author, so "author" becomes the unique identifier("_id" ). The second field means to add 1 to a "count" field for each document in the group. Note that the incoming documents do not have a "count" field; this is a new field created by the "$group"

---

## Aggregation pipeline

Transform and combine documents in a collection

Objects are transformed as they pass through a series of pipeline operators

- Such as filtering, grouping and sortingg

Pipeline operators need not produce one output document for every input document: operators may
also generate new documents or filter out documents

---

## Aggregation pipeline

Using projections you can

- Add computed fields
- Create new virtual sub-objects
- Extract sub-fields into the top-level of results

Pipeline operators can be repeated and combined freely

Note:

Returns result set inline

Operations can be repeated, for example, multiple $project or $group steps.

Supports non-sharded and sharded input collections

Can be indexed and further optimized

---

### Aggregation pipeline operators

**\$match**

Filters documents so that you can run an aggregation on a subset of documents
Can use all of the usual query operators

**\$sort**

Similar as sort with normal querying

---

### Aggregation pipeline operators

**\$limit**

Takes a number, n, and returns the first n resulting documents

**\$skip**

Takes a number, n, and discards the first n documents from the result set

---

### Aggregation pipeline operators

**\$unwind**

Unwinding turns each field of an array into a separate document

```C#
var allItems =
    await _collection.Aggregate()
        .Unwind(p => p.Items)
        .As<Item>()
        .ToListAsync();
```

---

### Aggregation pipeline: **\$group**

Grouping allows you to group documents based on certain fields and combine their values

When you choose a field or fields to group by, you pass it to the \$group function as the group’s "\_id" field

```C#
_collection.Aggregate()
        .Project(p => p.Level)
        .Group(l => l,
            p => new LevelCount {
                Id = p.Key,
                Count = p.Sum()
                }
            )
```

---

### Aggregation pipeline: **\$group**

There are several operators which can be used with **\$group**, such as:

- \$sum
- \$max
- \$min
- \$first
- \$last

---

### Aggregation pipeline: **\$project**

Much more powerful in the pipeline than it is in the “normal” query language

Allows to extract fields from subdocuments, rename fields, and perform operations on them

---

### Aggregation pipeline: \$project

There are many expressions which can be applied when projecting

- Math expressions - $add, $subtract, \$mod…
- Date expressions - $year, $month, \$week…
- String expressions - $subsctr, $concat, \$toLower…
- Logical expressions - $cmp, $eq, \$gt…

---

### Map-reduce

Can solve that is too complex for the aggregation framework

Uses JavaScript as its “query language” so it can express arbitrarily complex logic

Tends to be fairly slow and should not be used for real-time data analysis

Somewhat complex to use

---

### Single purpose aggregation methods: **Count**

```C#
var count =
    await _collection.CountAsync(
        Builder<Player>.Filter.Eq(p => p.Level, 3));
```

Returns the count of documents qualifying the defined condition(s)

---

### Single purpose aggregation methods: **Distinct**

The most simple way of getting distinct values for a key

Works for array keys and single value keys

```C#
var distinctLevels =
    await _collection.DistinctAsync(
        p => p.Level,
        Builders<Player>.Filter.Empty);
```
