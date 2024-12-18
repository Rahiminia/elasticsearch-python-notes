# ElasticSearch in Python

## Table of Contents 
  - [Concepts](#concepts)
  - [Connecting to Elasticsearch server in Python](#connecting-to-elasticsearch-server-in-python)
  - [Creating index](#creating-index)
  - [Data types](#data-types)
  - [Basic Operations](#basic-operations)
  - [Bulk API](#bulk-api)
  - [Search API](#search-api)
  - [SQL Search API](#sql-search-api)
  - [Aggregations](#aggregations)
  - [Filter context](#filter-context)
  - [Embedding vectors in Elasticsearch](#embedding-vectors-in-elasticsearch)
  - [Pagination](#pagination)
  - [Ingest pipelines](#ingest-pipelines)
  - [Ingest processors](#ingest-processors)
  - [Index Lifecycle Management (ILM)](#index-lifecycle-management-ilm)
  - [Analyzers](#analyzers)

### I highly recommend checking out [This Video](https://youtu.be/a4HBKEda_F8) by FreeCodeCamp. This notebook is based on their tutorial

## Concepts

ElasticSearch is a distributed, RESTful search & analytics engine with document-oriented model which uses inverted index.

**inverted index**: Maps words to the actual document location where those words occur. unlike databases which are forward indexed.

### Compared to SQL

| SQL | Elastic|
|-|-|
| Table | Index |
|Column| Field|
|Row| Document|

**Index**: a set of documents  
every index has settings for shards & replicas.
- **Shard**: dividing the document into several fragments.  
- **Replicas**: *ReadOnly* copies of the index.  
    *Replicas help system availability and allowing parallel search.*

**Cluster**: Multiple nodes running Elasticsearch

**Mapping**: When inserting documents into indeces, Elasticsearch tries to infer data types for each field. this process is called mapping. 
mapping is done automatically but we can also set it manually.

## Connecting to Elasticsearch server in Python
```python
from elasticsearch import Elasticsearch

es = Elasticsearch(url)
```

## Creating index

1. Simplest way

    In this method the **mappings** which define the structure of documents are infered automatically.

    ```python
    es.indices.create(index='index_name')
    ```
2. Specify the number of replicas and shards
    ```python
    es.indices.create(
        index='name',
        settings={
            'index': {
                'number_of_shards': 1,
                'number_of_replicas': 2,
            }
        }
    )
    ```
3. Specify the mappings *(code in next section)*

## Data types

### Common types
- Binary: Accepts a binary value as a **Base64** encoded string
- Boolean: True/False
- Numbers: long, integer, byte, short, etc.
- Dates
- Keyword: IDs, email addresses, status codes, zip codes, etc.  
    *keyword fields are not analyzed and we can't search part-of-text for them.*
### Objects (JSON)
- Object
- Flattened
    - efficient for deeply nested JSON object
    - Hierarchical structure is nt preserved
- Nested
    - good for array of objects
    - maintains relationship between the object fields

### Text search types
- Text
    - used for full-text content  
    *text fields are analyzed when indexed so we can search part-of-text with queries.*
- Completion
- Search as you type
- Annotated text

### Spatial data types
- Geo point (lat, lng)
- Geo shape (e.g. list of points in a shape)

### Dense vector
- Stores vectors of numeric values.
- **Dense** means we have few or no zero elements.
- This type **does not** support aggregations or sorting
- Nested vectors are **not** supported
- Use **KNN search** to retrieve nearest vectors
- Max size is 4096
- Elasticsearch does not automatically infer the mapping for dense vectors

> [!NOTE]  
> By default, Elasticseach creates two data types for **strings**. the **text** type and the **keyword** type.  
> with **text** we can search for individual words in text (*full-text search*), but with **keyword** the whole search phrase should be present (*keyword search*)

#### Set mapping manually:
```python
mapping = {
    'properties': {
        'id': {
            'type': 'integer',
        },
        'title': {
            'type': 'text',
            'fields': {
                'keyword': {
                    'type': 'keyword',
                    'ignore_above': 256,
                },
            },
        },
        'text': {
            'type': 'text',
            'fields': {
                'keyword': {
                    'type': 'keyword',
                    'ignore_above': 256,
                },
            },
        },
    }
}

es.indeces.create(index='name', mappings=mapping)
```

## Basic Operations

### Inserting documents

```python
documents = [
    {
        'title': 'title1',
        'text': 'this is a test document',
        'num': 1,
    },
    {
        'title': 'title2',
        'text': 'this is a test document',
        'num': 2,
    },
]

for doc in documents:
    es.index(index='index_name', body=doc)
```

### Deleting documents

```python
es.delete(index='index_name', id=0)
```
*if **id** doesn't exist, the operation throws an error.*

### Getting documents

```python
es.get(index='index_name', id=0)
```
*if **id** doesn't exist, the operation throws an error.*

### Counting documents

```python
res = es.count(index='index_name')
print(res['count'])

query = {
    'range': {
        'id': {
            'gt': 0,
        },
    },
}
res = es.count(index='index_name', query=query)
print(res['count'])
```

### Updating documents
1. If document exists

    the update follows these steps:
    1. Get the document
    2. Update it (using script)
    3. Re-index the result

    ```python
    # update fields using script
    es.update(
        index='index_name',
        id=id,
        script={
            'source': 'ctx._source.title = params.title',
            'params': {
                'title': 'New title',
            },
        }
    )

    # add new field using doc
    es.update(
        index='index_name',
        id=id,
        doc={'new_field' :'value'}
    )

    # remove fields using script
    es.update(
        index='index_name',
        id=id,
        script={
            'source': 'ctx._source.remove("field")'
        }
    )
    ```
2. If document doesn't exist

    the update operation can create the document if **doc_as_upsert** is set to **True**

    ```python
    res = es.update(
        index='index_name',
        id='new_id',
        doc={
            'book_id': 1,
            'book_name': 'A book',
        },
        doc_as_upsert=True,
    )


### Exists API
1. Check index
    ```python
    res = es.indices.exists(index='index_name')
    print(res.body) # True/False
    ```
2. Check document (with specific id)
    ```python
    res = es.exists(index='index_name', id=doc_id)
    print(res.body)
    ```

## Bulk API

The bulk API performs multiple operations in one API call. this increases indexing speed.

The bulk API consists of two parts:
- Action  
    *action can be one of these:*
    - index (creates or updates document)
    - create (creates document if doesn't exist)
    - update (updates document if already exists)
    - delete
- Source (the data)

```python
res = es.bulk(
    operations=[
        # action (index)
        {
            'index': {
                '_index': 'index_name',
                '_id': 1,
            },
        },
        # source
        {
            'field1': 'value1',
        },
        # action (update)
        {
            'update': {
                '_index': 'index_name',
                '_id': 2,
            },
        },
        # source
        {
            'field1': 'value2',
        },
        # action (delete doesn't require a source)
        {
            'delete': {
                '_index': 'index_name',
                '_id': 3,
            }
        },
    ]
)

# cheking for operation errors
if res.body['errors']:
    print('Error occured')
```

## Search API

The search API offers several arguments:
- index: where to search
- q: used for simple searches. uses **Lucene** syntax
- query: used for complex, structured queries. uses **DSL**
- timeout: max time to wait for search
- size: number of results (default is 10)
- from: search starting point (for pagination)

### Simple query
```python
# simple query:
es.search(
    index='index*',
    body={
        'q': {'match_all': {}}
    }
)

# this call also does the same:
es.search(index='index*')
```

### DSL
DSL (Domain-Specific Language) consists of two types of clauses:
- Leaf clauses
    - **match**  
    full-text search. returns documents that match a provided text, number, date or bool. filed must be of type **text**
    - **term**  
    return documents that contain an exact term. field must be of type **keyword** or **numeric**
    - **range**  
    return documents that contain values within a provided range (gt, lt, gte, ...)

- Compound clauses (bool)
    - combines multiple queries using these boolean logics:  
        - **must**: all queries are mandatory
        - **should**: queries are optional
        - **must_not**: all queries must be false
        - **filter**: filters documents based on given criteria  
    - field must be of type **text**

```python
# Leaf
es.search(
    index='index*',
    body={
        'query': {
            'match': {
                'field_name': 'value',
            },
        },
        'size': 5,
        'from': 10,
    }
)

# Compound
es.search(
    index='index_name',
    body={
        'query': {
            'bool': {
                'must': [
                    'match': {
                        'field_name': 'value',
                    },
                    'range': {
                        'field_name': {
                            'gt': 0,
                            'lte': 10,
                        }
                    }
                ],
                'filter': [
                    'term': {
                        'field_name': 'value',
                    },
                ],
            },
        },
    }
)
```

## SQL Search API

As an alternative to DSL queries, we can also use SQL queries to search documents in Elasticsearch.

The SQL API supports following parameters:
- delimiter
- cursor
- format
- filter
- fetch_size
- etc.
  
Supported formats are:
- txt
- csv
- json
- yaml
- binary
- etc.

```python
res = es.sql.query(
    format='txt',
    query='SELECT * FROM index ORDER BY field DESC LIMIT lim',
    filter={
        'range': {
            'field': {
                'gte': 10,
            },
        },
    },
    fetch_size=5,
)
```

We can also convert an SQL query to DSL using **translate**:

```python
es.sql.translate(
    query='SELECT * FROM index',
)
```

## Aggregations

**Aggregation** performs calculation on the data (avg, min, max, count, sum)

```python
res = es.search(
    index='index',
    body={
        'query': {
            'match_all': {}
        },
        'aggs': {
            'avg_agg': {
                'avg': {
                    'field': 'age',
                }
            }
        }
    }
)

print(res['aggregations']['avg_agg']['value'])
```

## Filter context

When searching in elasticsearch, we can either use **Query context** or **Filter context**

- Query: (*How well does this document match this query clause?*)  
Elasticsearch generates a **Score** for each result

- Filter: (*Does this document match this query clause?*)  
  Elasticsearch answers with **Yes/No**

Filters execute **faster** and require **less processing** compared to queries.

## Embedding vectors in Elasticsearch
**Embedding** transforms text into numerical vectors.  
deep learning models are used to embed documents. these models preserve the meaning of the text.

The **dense vector** data type can be used to store embedding vectors in Elasticsearch.  
and we can use **kNN algorithm** to search for similar vectors.
```python
# creating a field called 'embedding' of type dense_vector
es.indices.create(
    index='index_name',
    mappings={
        'properties': {
            'embedding': {
                'type': 'dense_vector',
            }
        }
    }
)

# retrieving documents similar to 'embedded_query' vector
res = es.search(
    knn={
        'field': 'embedding',
        'query_vector': embedded_query,
        'num_candidates': 50, # selected documents to perform knn on
        'k': 10, # k parameter of knn
    }
)

print(res['hits']['hits'])
```

## Pagination

In elasticsearch, there are two ways for pagination:
1. **from/size**  
   commonly used for small datasets  
   it is limited to 10k max size  
   high memory usage

2. **search_after**  
   document must have a sortable field (Numeric, Date)  
   efficient for large datasets

```python
# from/size example
es.search(
    index='index_name',
    body={
        'from': 0, # for next pages, set from to 10, 20, ...
        'size': 10,
        'sort': [
            {'field1': 'desc'},
        ],
    }
)

# search_after example
# we use the sort field of last doc in previous search to pass as search_after parameter
last_doc = res['hits']['hits'][-1]['sort']

es.search(
    index='index_name',
    body={
        'size': 10,
        'sort': [
            {'field1': 'desc'},
        ],
        'search_after': last_doc,
    }
)
```

## Ingest pipelines

With pipelines, we can perform transformations on data before indexing

Some common transformations: (remove fields, lowercase text, remove HTML tags, etc.)

*Different pipeline processors are discused in next section*

```python
# creating the pipeline
es.ingest.put_pipeline(
    id: 'pipeline_name',
    description: 'desc',
    processors=[
        {
            'set': {
                'description': 'desc',
                'field': 'field_name',
                'value': 'val',
            },
        },
        {
            'lowercase': {
                'field': 'field_name',
            },
        }
    ]
)

# using the pipeline
es.bulk(operations=ops, pipeline='pipeline_name')
es.index(index='index_name', document=doc, pipeline='pipeline_name')
```
We can also test a pipeline (with specified id) on a mock dataset using **simulate**:
```python
res = es.ingest.simulate(
    id: 'pipeline_name',
    docs=[
        {
            '_index': 'index',
            '_id': 'id1',
            '_source': {
                'key': 'val'.
            }
        },
        {
            '_index': 'index',
            '_id': 'id2',
            '_source': {
                'key': 'val'.
            }
        }
    ]
)
```

Pipelines can fail. we can either **ignore** or **handle** failures.
if we ignore the failure, the pipeline will skip the failed steps.

```python
es.ingest.put_pipeline(
    id: 'id',
    processors=[
        {
            'rename': {
                'description': 'desc',
                'field': 'field_name',
                'ignore_failure': False, # handling failure with on_failure
                'on_failure': [
                    {
                        'set': {
                            'field': 'error',
                            'value': "can't rename",
                            'ignore_failure': True,  # ignoring failure
                        }
                    }
                ]
            }
        }
    ]
)
```

## Ingest processors

Processors are organized into 5 categories:

- Data enrichment (append, inference, attachment, etc.)
- Data filtering (drop, remove)
- Array/JSON handling (foreach, json, sort)
- Pipeline handling (fail, pipeline)
- Data transformation (convert, rename, set, lowercase/uppercase, trim, split, etc.)

[More info in the official documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/processors.html)

## Index Lifecycle Management (ILM)

ILM automates the **rollover** and **management** of indices.  
It helps with storage optimization, automated data retention, efficient management if index size, etc.

**Rollover** is a process where the current index becomes *ReadOnly* and new documents are passed to a new index.

ILM has 4 phases:
- Hot phase: Index is being frequently updated
- Warm phase: Less frequently accessed data
- Cold phase: Archived data for long-term storage
- Delete phase: Data older than a threshold will be deleted

```python
policy = {
    'phases': {
        'hot': {
            'actions': {
                'rollover': {
                    'max_age': '30d',
                }
            }
        },
        'delete': {
            'min_age': '90d',
            'actions': {
                'delete': {}
            }
        }
    }
}

es.ilm.put_lifecycle(name='policy_name', policy=policy)
```

## Analyzers

Analyzers process text during indexing and searching. they tranfer text into tokens.

Analyzers consist of 3 components:
- Character filters (Optional)  
Some common filters: ['html_strip', 'mapping']

- Tokenizers (Only one)  
Some common tokenizers: ['standard', 'lowercase', 'whitespace']

- Token filters (Optional)  
Some common filters: ['apostrophe', 'decimal_digit', 'reverse', 'synonym_filter']

Elasticsearch provides ready-make options for processing text in various ways.  
Each builtin analyzer is designed for specific type of data

Here are some common analyzers:
- **Standard analyzer**:
    - No character filter
    - Standard Tokenizer
    - Lowercase filter & Stop filter
- **Simple analyzer**:
    - No character filter
    - Standard Tokenizer
    - No token filter
- **Whitespace analyzer**:
    - No character filter
    - Whitespace Tokenizer
    - No token filter
- **Stop analyzer**:
    - No character filter
    - Lowercase Tokenizer
    - Stop filter

```python
# using ready analyzers
res = es.indices.analyze(
    analyzer='standard',
    text='text to analyze'
)
tokens = res.body['tokens']

# defining custom analyzer
settings = {
    'settings': {
        'analysis': {
            'char_filter': { # defining character filter
                'ampersand_replace': {
                    'type': 'mapping',
                    'mappings': ['& => and'],
                },
            },
            'filter': { # defining token filter
                'synonym_filter': {
                    'type': 'synonym',
                    'synonyms': [
                        'car, vehicle',
                        'tv, television',
                    ]
                }
            }
            'analyzer': { # defining custom analyzer
                'custom_analyzer': {
                    'type': 'custom',
                    'char_filter': ['html_strip', 'ampersand_replace'],
                    'tokenizer': 'standard',
                    'filter': ['lowercase', 'synonym_filter']
                }
            }
        }
    },
    'mappings': {
        'properties': {
            'text_field': {
                'type': 'text',
                'analyzer': 'cusotm_analyzer', # using analyzer on a field
            }
        }
    }
}

es.indices.create(index='index-name', body=settings)
```
