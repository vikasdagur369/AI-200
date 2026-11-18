# Azure CosmosDB -> schemaless NoSQL database, automatic indexing.

PARTITION KEY deterimines performance more than anything.

## Resource Hierarchy :-

1. Account :- top level resource, unique DNS. hold upto 500 databases.
2. Database :- Database = Groups related containers + Can share throughput (RUs).
3. Container :- Stores items + Partition key + Scalable + No fixed schema
4. Item :- individual JSON document. Must have id + partition key value to be uniquely addressable via point read.

---

## Partition Key Design — The Most Important Decision :-

1. High cardinality :- distinct values spread data across partitions evenly. e.g -> userId, tenantId, productId etc.

2. Query-aligned :- If we query by userId, make it the partition key. same partition queries cost fewer RUs and are faster.

3. Immutable :- cant change partion key later.

`Exam tip: Choose a partition key that distributes data evenly and matches your common query patterns.`

---

## Request Units (RUs) — Cost Model :-

You can check how many RUs an operation consumed using:

1. x-ms-request-charge – HTTP response header that tells you the RU cost.
2. response.request_charge – SDK property that gives you the RU cost.

---

## Throughput Modes :-

1. Manual Throughput :- Example: Your application consistently receives 100 requests per second. Manual throughput is suitable.

2. Autoscale Throughput :- Example: Your AI application receives:

- Morning: 100 requests
- Afternoon: 1,000 requests
- Night: 50 requests
  Autoscale adjusts throughput according to demand.

Manual
Steady and predictable workload.
Autoscale
Bursty and unpredictable workload.

---

## Item System Properties

1. `_etag` — Version identifier

- Every time an item is updated, its ETag changes.
- Used for optimistic concurrency control.
- if_match ensures you update an item only if its version hasn't changed.
  Example: Two users try to update the same item. The first update succeeds. The second user's old ETag no longer matches, so the update is rejected (HTTP 412).

2. `_ts` — Last modified timestamp

- Stores the last modification time.
- Represented as a Unix timestamp (seconds).
- Useful for auditing and tracking when an item was last modified.

3. `_rid` — Resource ID

- Unique internal identifier assigned by Cosmos DB.
- Automatically generated and read-only.

4. `_self` — Resource link

- Internal URI/link used by Cosmos DB to identify a resource.
- Automatically generated and read-only.

Exam Tip: Remember \_etag for concurrency, \_ts for modification time, and \_rid / \_self for internal identification.

---

## Implement the Cosmos DB NoSQL SDK

```python
from azure.cosmos import cosmosClient
from azure.identity import DefaultAzureCredential

client = cosmosClient(
    "https://myaccount.documents.azure.com:443/",
    credential=DefaultAzureCredentials()
)

db = client.get_database_client("aidata")
container = db.get_container_client("products")
```

## Authentication Options :-

1. Account Keys — Simple authentication

- Cosmos DB provides primary and secondary account keys.
- Anyone with the key can access the database with broad permissions.
- Easy to configure, but risky if leaked.
- Best for development/testing, avoid in production.
  Example: Your application uses an account key stored in a configuration file to connect to Cosmos DB.

2. Entra ID (RBAC) — Role-based authentication

Exam Tip: Production + Managed Identity = Entra ID (RBAC).

Remember: Contributor (Role) = Read/Write, Reader (Role) = Read-only.

Role Permission
Cosmos DB Built-in Data Contributor Read + Write
Cosmos DB Built-in Data Reader Read-only

---

## CRUD operations :-

```python
# Create - fails if id already exists
container.create_item(body={"id":"123","categoryId":"electronics", "name": "speaker"})

# If the item doesn't exist → Insert. If the item already exists → Replace it.
container.upsert_item(body={...})

# Read, id and partition key both are required
container.read_item(
    item="123",
    partition_key="electronics"
)

# Delete
 container.delete_item(
    item="123",
    partition_key="electronics"
 )

# completly replace
container.replace_item(
    item=item["id"],
    body=item,
    if_match=item["_etag"]
)
```

Remember:

- replace_item() = Replace the existing item with updated data.
- if_match + \_etag = Prevent overwriting someone else's newer changes.

---

## Idempotent Container & db creation

```python
from azure.cosmos import PartitionKey, ThroughputProperties

db = client.create_database_if_not_exists(id="aidata")

container = db.create_container_if_not_exists(
    id="products",
    partition_key=PartitionKey(path="/categoryId"),
    offer_throughput= ThroughputProperties(auto_scale_max_throughput=4000)
)
```
