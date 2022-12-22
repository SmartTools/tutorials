---
layout: page
title: "Version Management"
description: "How to use Feature Versioning"
parent: "Developing Server"
nav_order: 4
---


# Version Management

## Overview

This part describes how to enable/disable feature versioning, how to use feature versioning
in message processing and clarifies the scope and module context switching concepts.
 
## How to enable/disable Feature Versioning

The Feature Versioning is included by default when you download server's core using `das dc`
command. You may make sure of this by checking presence of modules `version-management` and
`version-management-plugin` in `core` directory of your Smartactors server.
Then on server startup Feature Versioning is enabled by default.

To disable Feature Versioning on your server you may just remove modules 
`version-management` and `version-management-plugin` from `core` server directory and
then restart the server.

If Feature Versioning is disabled then server does not differentiate feature, chain 
and actor versions and overwrites them on [`Feature Reloading`](./Reloading). 
Also IOC registrations become not isolated by features dependency hierarchy.
In case of [`Feature Reloading`](./Reloading) without Feature Versioning 
the chains and actors of feature are always overwritten by newly loaded chains and actors 
with same names, as far as IOC registrations.

## Setup Feature Version

You may setup version of your feature in feature name in `config.json` file using `:` 
separator like in example below:

```json
{
  "featureName": "com.my-project:original-feature:0.4.3",
  "afterFeatures": ["com.my-project:side-feature"]
}
```

Feature with no version in its name is considered as having lowest version. You may 
load to server and use several versions of feature at the same time.

All chains and actors of feature are registered with the version of their feature.  

All dependent features are connected to the highest version of base feature (from which 
are already loaded) while dependent feature loading. You may setup in feature dependencies
in `config.json` file the exact version of base feature on which dependent feature depends
on. 

```json
{
  "featureName": "com.my-project:original-feature:0.4.3",
  "afterFeatures": ["com.my-project:side-feature:0.3.5"]
}
```

In that case dependent feature will be connected to indicated version of base feature
if indicated version of base feature is already loaded to server. Otherwise dependent
feature will be connected to the highest version of base feature loaded and server throws
warning about version inconsistency.

## Using versioning for message processing

If versioning is enabled on your server and several feature versions have been loaded then,
on each feature chain call, chain version selection is preformed. The default behaviour is
to select the highest version of the chain. If you want to select the version of chain 
depending on message context then you need to register in feature plugin the strategy to
resolve chain version from message. 

### How to register chain version resolution strategy

Please see below the example how to register such strategy in feature plugin:

```java
IFieldName versionFN = IOC.resolve(
    Keys.getKeyByName("info.smart_tools.smartactors.iobject.ifield_name.IFieldName"), 
    "msgVersion"
);
IOC.resolve(
    Keys.getKeyByName(
        "register_message_version_strategy",
        "chain-name",
        new ApplyFunctionToArgumentsStrategy(args -> {
            IObject message = (IObject) args[0];
            Object version = message.getValue(versionFN);
            return version;
        })
    )
);
```

The registration strategy which called by `register_message_version_strategy` key must 
receive the chain name and chain versioning strategy as parameters. Chain version 
strategy receives the message as IObject parameter and must return the version of chain
to call in `X:Y:Z` format. In the example above the version is taken directly from message
field "msgVersion", but you may use any other way. The example of selecting version 
depending on presence of some field in message:
 
```java
IFieldName someParamFN = IOC.resolve(
    Keys.getKeyByName("info.smart_tools.smartactors.iobject.ifield_name.IFieldName"), 
    "someParameter"
);
IOC.resolve(
    Keys.getKeyByName(
        "register_message_version_strategy",
        "chain-name",
        new ApplyFunctionToArgumentsStrategy(args -> {
            IObject message = (IObject) args[0];
            Object someParam = message.getValue(someParamFN);
            if (someParam == null) { // parameter is not present
                return "1:5:2"; 
            } else { // parameter is present
                return "1:6:8";
            }
        })
    )
);
```

### Version strategy processing

Each version of feature may register its own versioning strategy. If several version
strategies are registered then they are stored in the list ordered by corresponding feature
version. 

When chain selection process is performed on the message processing startup then all version 
strategies are performed one by one starting from highest version of registered version 
strategies until the version is returned. 

If the strategy has returned version then process stops, the chain with 
corresponding version is selected and the rest of strategies is not performed. 
Otherwise, if strategy returns `null` then next strategy will be performed. 

If all of these strategies returned `null` then the highest version of loaded chain will 
be chosen for message processing. If strategy returned some version, but chain with this 
version is not present on the server then exception will be thrown and message will not be 
processed.

## Scope and module context 

By default on each chain call the scope and module context switching is performed. That is
the chain is performed in the scope and module context of feature where the chain is contained.
But this is not right for actors, they always are performed in the scope and module context of
the caller.

For chains it means that if you registered something in IOC in dependent feature and call the
chain from base feature then this chain and its actors will not be affected by your 
IOC registrations because scope and module context are switched on chain call.

If your actor is executed in some module (feature) context then it may see all IOC registrations
done in this module context and upward on all module contexts of the modules which this module
depends on. 

If the IOC registration is performed in some module context then it hides all IOC registrations 
on this key upward on module dependencies.

If your actor tries to resolve some key from IOC and this key was registered in module which is
not in dependencies of module context of actor execution then this key will not be resolved and
corresponding exception `ResolutionException` will be thrown.

So check careful the feature dependencies of your features in order to keep the dependency 
hierarchy integrity.

### Scope and module context management on chain calling

If you want to override the default behavior of system of switching scope and module context
on chain call then you may use `scopeSwitching` parameter in chain call descriptions in
`config.json` file. Please see below the `config.json` examples of preventing scope and module context switching
in cases where it is possible. Note that default value for this parameter is always `true`.

Calling chain from chain directly:

```json
{
  "featureName": "com.my-project:dependent-feature",
  "afterFeatures": ["com.my-project:original-feature"],
  "objects": [],
  "maps": [
    {
      "id": "dependent-chain",
      "externalAccess": true,
      "steps": [
        {
          "chain":"base-chain",
          "scopeSwitching": false,
          "target":"constant_chain_call_receiver"
        }
      ],
    "exceptional": []
    }
  ]
}
```

Same story if you organize `conditional_chain_call_receiver` from 
`condition chain choice strategy`.

Similar behaviour if you create routing step in chain:

```json
{
  "featureName": "com.my-project:endpoint-config-feature",
  "afterFeatures": ["info.smart_tools.smartactors:http-endpoint-plugins"],
  "objects": [
    {
      "name": "router",
      "kind": "raw",
      "dependency": "info.smart_tools.smartactors.message_processing.chain_call_receiver.ChainCallReceiver",
      "strategyDependency": "chain choice strategy"
    }
  ],
  "maps": [
    {
      "id": "routing_chain",
      "steps": [
        {
          "target": "router",
          "scopeSwitching": false
        }
      ],
      "exceptional": []
    }
  ]
}
```

You may prevent scope and module context switching on exceptional chain call by using same
parameter in exceptional block description:

```json
{
  "featureName": "com.my-project:dependent-feature",
  "afterFeatures": ["com.my-project:original-feature"],
  "objects": [],
  "maps": [
    {
      "id": "dependent-chain",
      "externalAccess": true,
      "steps": [
        {
          "target": "some-actor",
          "handler": "someAction"
        }
      ],
      "exceptional": [
        {
          "class": "com.my-project.exceptions.MyException",
          "chain": "error-handling-chain",
          "scopeSwitching": false,
          "after": "break"
        }
      ]
    }
  ]
}
```

You may use same parameter in `[Endpoint]` and `[onFeatureLoading]` sections of `config.json` 
file. In that case corresponding chain will be performed in the scope and module context of 
feature which contains this `[Endpoint]` and/or `[onFeatureLoading]` section:   

```json
{
  "featureName": "com.my-project:endpoint-config-feature",
  "afterFeatures": ["info.smart_tools.smartactors:http-endpoint-plugins"],
  "endpoints": [
    {
      "name": "mainHttpEp",
      "type": "http",
      "port": 9909,
      "startChain": "routing_chain",
      "scopeSwitching": false,
      "maxContentLength": 4098,
      "stackDepth": 5
    }
  ],
  "onFeatureLoading": [
    {
      "chain": "init-system-params-in-memory",
      "scopeSwitching": false,
      "messages": [{}]
    }
  ]
}
```

Finally, you may setup additional boolean parameter when you send message to MessageBus:

```java
MessageBus.send(message, false)
```
    
or

```java
MessageBus.send(message, chainName, false)
```

and so on.
