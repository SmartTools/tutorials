---
layout: page
title: PostgreSQL Setup
parent: Quickstart
nav_order: 3
---

# PostgreSQL Setup

## Dependencies

To start using database, especially PostgreSQL in your project you have to add this features in the `corefeatures/features.json` file:

- database
- database-plugins
- database-postgresql
- database-postgresql-plugins
- database-postgresql-connection-options
- database-service-starter
- database-postgresql-create-collection-if-not-exists

For example, your `corefeatures/features.json` could look like:

```json
{
  "repositories": [
    {
      "repositoryId": "smartactors_core_and_core_features",
      "type": "default",
      "url": "https://repository.smart-tools.info/artifactory/smartactors_core_and_core_features/"
    },
    {
      "repositoryId": "smartactors_common_features",
      "type": "default",
      "url": "https://repository.smart-tools.info/artifactory/smartactors_common_features/"
    }
  ],
  "features": [
    {
      "group":"info.smart_tools.smartactors",
      "name": "database-service-starter",
      "version": "0.6.0"
    },
    {
      "group": "info.smart_tools.smartactors",
      "name": "database",
      "version": "0.6.0"
    },
    {
      "group": "info.smart_tools.smartactors",
      "name": "database-plugins",
      "version": "0.6.0"
    },
    {
      "group": "info.smart_tools.smartactors",
      "name": "database-postgresql",
      "version": "0.6.0"
    },
    {
      "group": "info.smart_tools.smartactors",
      "name": "database-postgresql-plugins",
      "version": "0.6.0"
    },
    {
      "group": "info.smart_tools.smartactors",
      "name": "database-postgresql-create-collection-if-not-exists",
      "version": "0.6.0"
    }
  ]
}
```

## Configurations

Create your custom feature, for example `SetupPostgresConnectionOptions`. We will not write some code in this feature but add actions to the `config.json`:

```json
{
  "featureName": "com.my-project:setup-postgres-connection-options",
  "afterFeatures": [
    "info.smart_tools.smartactors:database-postgresql-plugins"
  ],
  "database": [
    {
      "key": "PostgresConnectionOptions",
      "type": "PostgresConnectionOptionsStrategy",
      "config": {
        "url": "jdbc:postgresql://localhost:5432/example",
        "username": "example",
        "password": "example",
        "maxConnections": 20
      }
    }
  ]
}
```

Here we can see a new `database` section. It'll be read by the `database-service-starter` and using some `PostgresConnectionOptionsStrategy` register a connectionOptions with a name specified in `key` field. In our case it's `PostgresConnectionOptions`. You can specify as many new connection options as you need.

## Create collections

In our framework we don't need any migrations but we have to create collections in a DB. For this purposes we can use `database-postgresql-create-collection-if-not-exists` feature which allows us to specify collection names and create them on our feature being loaded.

To use it, create a new feature, for example `CreateCollections` and fill the `config.json` with this:

```json
{
  "featureName": "com.my-project:create-collections",
  "afterFeatures": [
    "com.my-project:setup-postgres-connection-options",
    "info.smart_tools.smartactors:database-postgresql-create-collection-if-not-exists"
  ],
  "onFeatureLoading": [
    {
      "chain": "createCollections",
      "messages": [
        {
          "collectionName": "example_collection_1",
          "connectionOptionsRegistrationName": "PostgresConnectionOptions"
        }
      ]
    },
    {
      "chain": "createCollections",
      "messages": [
        {
          "collectionName": "example_collection_2",
          "connectionOptionsRegistrationName": "PostgresConnectionOptions"
        }
      ]
    }
  ]
}
```

You can see, that this feature allows us to send messages to the some inner chain that creates collections. **Important** thing is that this feature **MUST** depends on our previous feature that creates connection options `com.my-project:setup-postgres-connection-options`.

You can specify as many messages in `messages` section as many collections you need.

Best practises: add this messages in every feature, that need this collection. Do not create a single feature that creates everything. Here we do this just for education purposes.

### Warning!!

Do not combine `onFeatureLoading` and `database` sections, because there is no garantee of execution order. Always create a feature with connection options and a separate feature for working with this connection options.

## How to use it?

You have registered options and created collections. How to use them?

To do database tasks you need to have an `IPool` object, to get it use this snippet:

```java
final ConnectionOptions options = IOC.resolve(Keys.getKeyByName("PostgresConnectionOptions")); // you can get this key from message for example!!
final IPool pool = IOC.resolve(Keys.getKeyByName("PostgresConnectionPool"), options);
String collectionName = "example_collection_1";
try (PoolGuard guard = new PoolGuard(pool)) {
    ITask task = IOC.resolve(
            Keys.getKeyByName(SOME_TASK_ID),
            guard.getObject(),
            collectionName,
            message.getOptions()
    );
    task.execute();
} catch (TaskExecutionException | ResolutionException | PoolGuardException e) {
    throw new CreateCollectionActorException(e);
}
```

## Example

You can dive into code here: [Example](https://github.com/SmartTools/tutorials/tree/master/src/postgresql_setup)
