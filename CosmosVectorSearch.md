## Introduction to Cosmos DB Vector Search

1. documents == items
2. we can add another field to item(json) -> embedding. means in a same item we can also store embedding inside same container.

### Configure Vector Index on Container

```python
from azure.cosmos import CosmosClient, PartitionKey
from azure.identity import DefaultAzureCredential

client = CosmosClient(
    "https://myaccount.documents.azure.com:443/",
    "credential=DefaultCredential()"
)
db = client.get_database_client("ai-db")

# create container with vector policy
container = db.create_container_if_not_exists(
    id="documents",
    partition_key=PartitionKey("/category"),
    vector_embedding_policy={
        "vectorEmbeddings" : [{
            "path": "/embedding",
            "dataType" : "float32",
            "dimensions" : 1536,
            "distanceFunction":"cosine"
        }]
    },
    indexing_policy={
        "vectorIndexes":[{
            "path":"/embedding",
            "type":"diskANN"
        }]
    } 
)
```