---
layout: page
title: Reloading
description: "How to use Features Reloading"
parent: "Developing Server"
nav_order: 3
---


# Feature Reloading

## Overview

This part describes the functionality of reloading features without server restart. 
Feature may be loaded to server again even if it has already been loaded. But newly 
loaded feature must have version different from previously loaded. Same version of 
feature cannot be loaded again.

If same feature is reloaded with new version and [`Feature Versioning`](./VersionManagement) 
is disabled then all chains within this feature are 
re-registered under the same names. Actors within reloaded feature are re-registered too. 
Previously loaded feature with another version is still in memory and all its dependent 
features are connected to this previously loaded feature. 
You may reload dependent features too in order 
to reconnect them to reloaded base feature. To keep correct hierarchy you may setup
exact version of base feature in the dependent feature dependencies. In case if no
version of base feature is present in feature dependencies then the highest version 
from loaded base features is used to connect dependent feature to.

Anyway in case if [`Feature Versioning`](./VersionManagement) is disabled and
any chain of base feature is called from chain of dependent feature then 
chain from last loaded base feature is chosen and called. But if chain from dependent
feature directly calls actor belonging to base feature then actor is called from base
feature which dependent feature connected to. That is if dependent feature connected
to previously loaded base feature and its chain calls actor from base feature 
then actor from previously loaded base feature will be called even if base feature 
was reloaded.

<!-- If feature with same name is reloaded but with another version then chains and actors 
are not overwritten and exist in memory in all versions. -->

If [`Feature Versioning`](./VersionManagement) is enabled and on chain call version 
of called chain is not defined directly then special strategy is 
used to determine which chain version must be called basing on chain name and processing 
message. This strategy may be registered for any chain in corresponding feature plugin. 
See [corresponding section](./VersionManagement) of this tutorial to understand how to 
register this strategy. If no version strategies registered for chain then chain with 
maximum version is chosen on chain call by default.

There are three ways to initiate feature reloading: putting the file with feature into 
`features`  subdirectory of Smartactors server directory, calling of special chain of 
RemoteManagement feature with path to feature file in the file system, calling of special 
chain of RemoteManagement feature with description of feature and description of remote 
repository from where feature can be downloaded.

## Loading/reloading feature using `features` subdirectory on Smartactors server.

When Smartactors server is running you may just put the file with the feature you want to 
load into `features` subdirectory. Then server catch the file and load feature 
immediately. Note that file with feature must not have name which already present in 
`features` directory. If you want to reload feature which already present in `features` 
directory then just rename its file to different name (best way is to include version
into feature file name).

## Loading/reloading feature from file on the server

To use this way please make sure that remote-management feature is present in 
`corefeatures` directory of your Smartactors server. If so then to reload feature 
you may call chain `load-feature-from-file` with message which contains parameter 
`featureLocation` with full feature file path in value. Please see below the sample 
of message to call this chain through routing:

```json
{
    "messageMapId" : "load-feature-from-file",
    "featureLocation" : "/home/user1/sm_server/features/my_feature.jar"
}
```

Note that `load-feature-from-file` chain has no external access, therefore it can be 
used only from inside of server. To call it from outside you may create your own 
externally accessible chain which calls this chain (you chain may contain some 
authentication).

## Loading/reloading features from remote repository

To use this way please make sure that remote-management feature is present in 
`corefeatures` directory. If so then to reload features you may call chain 
`load-features-from-repository` with message which contains the description of features 
and repositories. Please see below the example of message to call this chain through 
routing:

```json
{
    "messageMapId" : "load-features-from-repository",
    "repositories" : [
        {
            "repositoryId" : "archiva.smartactors-features",
            "type" : "default",
            "url" : "http://archiva.smart-tools.info/repository/smartactors-features/"
        }
    ],
    "features": [
        {
            "group": "info.smart_tools.smartactors",
            "name": "endpoint",
            "version": "0.6.0"
        },
        {
            "group": "info.smart_tools.smartactors",
            "name": "endpoint-plugins",
            "version": "0.6.0"
        }
    ]
}
```

Format of `repositories` and `features` json variables is same as in `features.json` 
file usually stored in `corefeatures` server directory.

Note that `load-features-from-repository` chain has no external access, therefore it 
can be used only from inside of server. To call it from outside you may create your 
own externally accessible chain which calls this chain (you chain may contain some 
authentication).
