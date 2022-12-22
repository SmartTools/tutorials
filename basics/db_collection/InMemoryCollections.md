---
layout: default
title: In-memory Collections
parent: Database Collections
grand_parent: Basics
nav_order: 1
---


# In-memory DB collections

This implementation uses in-memory structures (maps and lists) to temporary save documents.
Use it for testing.

## Plugins

In-memory collections can be used in place of Postgres collections. Just load another set of plugins.

These plugins are required to work with in-memory collections:

* NullConnectionPoolPlugin
* PluginInMemoryDatabase
* PluginInMemoryDBTasks

Plus general plugins which implementation may varies:

* PluginIOCSimpleContainer
* PluginIOCKeys
* IFieldNamePlugin
* IFieldPlugin
* PluginDSObject

## Modules

This is the list of modules (jars) used by in-memory collections:

* plugin.null_connection_pool
* plugin.in_memory_database
* plugin.in_memory_db_tasks
* core.istorage_connection
* core.in_memory_database
* core.in_memory_db_create_collection_task
* core.in_memory_db_insert_task
* core.in_memory_db_upsert_task
* core.in_memory_db_delete_task
* core.in_memory_db_get_by_id_task
* core.in_memory_db_select_task
* core.in_memory_db_count_task
* strategy.uuid_nextid_strategy

## Connection Pool

In-memory database doesn't require connections and connection pools.
However, you can use null connection poll, to keep your code compatible with Postgres collections

```java
IPool pool = IOC.resolve(Keys.getKeyByName("DatabaseConnectionPool"));
```
    
## Full text search

In-memory database uses naive implementation of full text search.
It means the results of the search may differ from the results of the search in Postgres collections.
