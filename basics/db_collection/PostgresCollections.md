---
layout: default
title: PostgreSQL Collections
parent: Database Collections
grand_parent: Basics
nav_order: 2
---

# PostgreSQL DB collections

## Plugins

These plugins are required to work with Postgres DB collections:

* PostgresConnectionPoolPlugin
* PostgresDBTasksPlugin

Plus general plugins which implementation may varies:

* PluginIOCSimpleContainer
* PluginIOCKeys
* IFieldNamePlugin
* IFieldPlugin
* PluginDSObject

## Modules

This is the list of modules (jars) used by Postgres DB collections:

* plugin.postgres_connection_pool
* plugin.postgres_db_tasks
* core.postgres_connection
* core.istorage_connection
* core.postgres_schema
* core.postgres_create_task
* core.postgres_insert_task
* core.postgres_upsert_task
* core.postgres_delete_task
* core.postgres_getbyid_task
* core.postgres_search_task
* core.postgres_count_task
* strategy.uuid_nextid_strategy

Plus Postgres JDBC driver:

* postgresql-9.4-1206-jdbc42.jar
* jna-3.0.9.jar

## Connection pool

Resolving of DatabaseConnectionPool requires ConnectionOptions.

```java
ConnectionOptions connectionOptions = new SomeConnectionOptions();
IPool pool = IOC.resolve(Keys.getKeyByName("DatabaseConnectionPool"), connectionOptions);
```
    
Connection options contains host, port of the server, database name, username and password.
    
## Collection

Each collection corresponds to the table in the PostgreSQL database.

If the collection has no fulltext index it is created as this:

```sql
CREATE TABLE collection (document jsonb NOT NULL);
CREATE UNIQUE INDEX collection_pkey ON collection USING BTREE ((document#>'{collectionID}'));
```

## Indexes

### Ordered index

For each field with ordered index this is created:

```sql
CREATE INDEX ON collection USING BTREE ((document#>'{field}'));
```
    
### Datetime index

For each field with datetime index this is created:

```sql
CREATE INDEX ON collection USING BTREE ((parse_timestamp_immutable(document#>'{field}')));
```
    
### Tags index

For each field with tags index this is created:

```sql
CREATE INDEX ON collection USING GIN ((document#>'{field}'));
```

### Full text index        
        
If the collection has fulltext index it is created as this:

```sql
CREATE TABLE collection (document jsonb NOT NULL, fulltext tsvector);
CREATE UNIQUE INDEX collection_pkey ON collection USING BTREE ((document#>'{collectionID}'));
CREATE INDEX ON collection USING GIN (fulltext);
```
    
Also the trigger and the function `collection_fulltext_update_trigger()` are created.

This means fulltext search operation works only when the fulltext index was created
and looks only to data of indexed fields.

## Search

Search queries start from `SELECT document FROM collection`.

In most cases the access to the field is done using path in the document.

```sql
WHERE (document#>'{field,nested}') = to_json(?)::jsonb
``` 
    
### Datetime fields

For date-from and date-to search additional conversion is performed.

```sql
WHERE (parse_timestamp_immutable(document#>'{date-from}') >= (?)::timestamp)
```
    
### Tags

For tags search the operator `?` is used.

```sql
WHERE ((document#>'{tags}')??(?))
```
    
### Full text search

The full text search is done against `fulltext` column using operator `@@`.

```sql
WHERE fulltext@@(to_tsquery('russian',?))
```
    
## Count

Count queries start from `SELECT COUNT(*) FROM collection`.
Then the same WHERE clause is used as for Search queries.