# Storage

This file covers the *concepts* behind storage choices. Specific products are in [databases.md](../building-blocks/databases.md), [blob-storage.md](../building-blocks/blob-storage.md), and [search-databases.md](../building-blocks/search-databases.md).

## Choosing a database family

```mermaid
flowchart TD
    Q1{"Do you need relationships,<br/>joins, and strong constraints?"} -- Yes --> REL["Relational (SQL)<br/>e.g. Postgres, MySQL"]
    Q1 -- No --> Q2{"What is the access pattern?"}
    Q2 -- "Lookup by a single key" --> KV["Key-value<br/>e.g. Redis, DynamoDB"]
    Q2 -- "Nested, changing structure" --> DOC["Document<br/>e.g. MongoDB"]
    Q2 -- "Huge writes, known queries per row" --> WIDE["Column-family<br/>e.g. Cassandra"]
```

| Family | Data model | Strength | Trade-off |
|---|---|---|---|
| **Relational** | Tables, rows, columns, foreign keys | Joins, transactions, constraints, flexible queries | Harder to scale writes horizontally; joins get expensive on large tables |
| **Key-value** | Key to value | Very fast, simple, scales horizontally | Only access by key (unless you add indexes) |
| **Document** | JSON-like documents | Flexible schema, nested data | Joins across documents are weak |
| **Column-family** | Wide rows, partitioned by key | Write-heavy workloads at scale | Limited query flexibility; design around queries |
| **Graph** | Nodes and edges | Relationship traversal | Specialized; rarely the default |

Avoid broad claims like "I need NoSQL because of scale" or "I need SQL because of relationships." Both relational and NoSQL databases can handle either. Name the database you pick and the specific feature that solves the problem. For example: "Postgres, because ACID transactions protect data integrity."

Common starting point: for product-style data with relationships, pick a relational database. For high-volume infrastructure data, consider a NoSQL database.

## ACID vs BASE

These describe how strict the guarantees are.

| | ACID | BASE |
|---|---|---|
| **Typical systems** | Relational databases | Many NoSQL and distributed systems |
| **Meaning** | **A**tomicity: all or nothing. **C**onsistency: rules hold after each transaction. **I**solation: concurrent transactions do not interfere. **D**urability: committed data survives crashes. | **B**asically **A**vailable: the system responds even with failures. **S**oft state: data can change over time without input. **E**ventually consistent: replicas converge given time. |
| **Choose when** | Money, inventory, bookings: correctness matters more than availability | Feeds, likes, analytics: availability and speed matter more than instant accuracy |

*(general knowledge: ACID and BASE definitions)*

Transactions are the main ACID feature in practice. Example: create a user and their first post together. Either both exist or neither does.

## Data modeling

### Start with three factors

Tie every schema decision to these:

- **Data volume**: how much data exists, and whether it must be split across stores.
- **Access patterns**: the most important factor. For each endpoint, ask which queries it needs to support.
- **Consistency requirements**: money and inventory need ACID in one database. Feeds and like counts can be eventually consistent and denormalized.

In an interview, say the reason out loud: "Likes can be eventually consistent, so I'll denormalize the like count into the posts table."

### Normalization vs denormalization

```mermaid
flowchart LR
    subgraph N["Normalized"]
        U1[(users: id, name)]
        O1[(orders: id, user_id, product_id)]
        O1 -. "join by user_id" .-> U1
    end
    subgraph D["Denormalized"]
        O2[(orders: id, user_id, username, product_id, product_name)]
    end
```

| | Normalized | Denormalized |
|---|---|---|
| **Storage** | Each fact stored once | Facts copied into multiple places |
| **Reads** | Needs joins | Fast, single-table reads |
| **Writes / updates** | Update once | Update every copy (for example, a username change) |
| **Best for** | Start here; consistency matters | Read-heavy, rarely changing data; hot paths |

Default: start normalized. Denormalize specific hot paths when you can point to a read problem. Do not propose denormalization upfront without a reason.

Valid reasons to denormalize: analytics over data that rarely changes, event logs and audit trails (a snapshot at a point in time), and read-heavy data. A common alternative is to keep a normalized source of truth and put a cache such as Redis in front, holding precomputed joins or aggregates.

### NoSQL data modeling

Design around access patterns. In DynamoDB-style stores, the partition key decides where data lives:

- Query "all posts for user X": partition key `user_id`. That query is a fast single-partition lookup.
- Query "all posts that mention hashtag Y": not supported well. It needs a scan or another table keyed by hashtag.

List the queries first, then design the keys.

### Entities, keys, and constraints

- **Primary keys**: use system-generated IDs (`user_id`, `post_id`), not business data such as email. They stay stable when business rules change.
- **Relationships**: one-to-many is a foreign key on the many side (`posts.user_id`). Many-to-many (users like posts) needs a join table (`likes(user_id, post_id)`). One-to-one is rare; the two tables are often better merged.
- **Foreign keys** prevent orphaned rows, but every insert and update pays for the check. At very large scale some companies drop them and enforce integrity in the application. Name this trade-off if you bring up foreign keys.
- **Constraints** (NOT NULL, UNIQUE, CHECK) enforce rules in the database, such as unique emails or positive prices, at some write cost.

## Indexes

An index is an extra structure that speeds up lookups at the cost of storage and slower writes.

```mermaid
flowchart LR
    Q["WHERE email = 'x@y.com'"] --> NOIDX["No index:<br/>scan all 10M rows"]
    Q --> IDX["Index on email:<br/>B-tree lookup, a few steps"]
```

| Index type | Good for |
|---|---|
| **B-tree** (default) | Exact lookups and range queries (dates, prices) |
| **Hash** | Exact lookups only; no ranges |
| **Composite** (multi-column) | Queries on several fields, such as city plus date |
| **Full-text** | Word search. See [search-databases.md](../building-blocks/search-databases.md) |
| **Geospatial** | "Find places within 5 miles." See [proximity-services.md](../patterns/proximity-services.md) |

Tip: list the fields your queries filter on and propose indexes on those. For "find user by email", index `email`.

External search indexes are usually fed from the primary database through **change data capture (CDC)**. They lag slightly behind, which is acceptable for search.

## Sharding for data modeling

For the mechanics, see [scalability](./05-scalability.md). For the schema:

- **Shard by the primary access pattern.** If you mostly query posts by user, shard by `user_id`.
- **Avoid cross-shard queries.** A timeline that merges posts from many followed users, sharded by `user_id`, has to query many shards.
- **Avoid time-range shards for write-heavy data.** All current writes hit the newest range, creating a hot shard. Time-range partitioning suits archives and analytics.
- **The shard key is hard to change.** It affects every query, so decide it early.

## Blob storage

Do not store large files (images, videos) in a relational database. Store the file in object storage and keep a URL in the database. Details: [blob-storage.md](../building-blocks/blob-storage.md).

## Common mistakes

- Choosing a database by brand instead of by access pattern.
- Sharding before trying indexes, replicas, and caches.
- Using a database as a queue or as blob storage.
- Forgetting that joins become a bottleneck on large tables.
- Using a business value such as email as the primary key.
- Denormalizing before finding a read problem.
- Using time-range sharding for a write-heavy table.
- Choosing a graph database for relationships a relational database can handle.

## Example decision

Users and orders need transactions, so use Postgres. Index `user_id` on orders for the "my orders" query. Product search goes to Elasticsearch, fed by CDC, so search results can be a few seconds stale.

## Related

- [Databases](../building-blocks/databases.md)
- [Scalability](./05-scalability.md)
- [CAP and consistency](./07-cap-and-consistency.md)
- [Search databases](../building-blocks/search-databases.md)

## Sources

- Hello Interview, [Core Concepts](https://www.hellointerview.com/learn/system-design/in-a-hurry/core-concepts)
- Hello Interview, [Core Concepts: Data Modeling](https://www.hellointerview.com/learn/system-design/core-concepts/data-modeling)
- Hello Interview, [Key Technologies: Core Database](https://www.hellointerview.com/learn/system-design/in-a-hurry/key-technologies)
