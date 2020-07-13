---
layout: page
title: Das User's Guide
description: Das User's Guide
group: das
---

# Das User's Guide

## Conventions

### Naming

ProjectName, FeatureName, ActorName and PluginName must be written in the UpperCamelCase and contains only letters and digits.

For the specific project each feature must have a unique name.
For the specific feature each actor, plugin must have a unique name.

### Project structure

TBD

### Versioning

TBD

## Installation and server creation

For das installation following pre-installed application are needed:

* open-jdk-8
* maven 3+

[Download](https://repository.smart-tools.info/artifactory/smartactors_development_tools_dev/info/smart_tools/smartactors/das/0.6.0/das-0.6.0.deb) and install the debian package :

```console
$ sudo dpkg -i das-0.6.0.deb
```

Check das:

```console
$ das
```

The result should be:

```console
Distributed Actor System. Design, assembly and deploy tools.
Version 0.6.0.
```

Use the command `cs` (create server) to create the server. This command supports the following arguments:

* `aid` — the artifact id of the server (default value: servers.server2)
* `g` — the group id of the server (default value: info.smart_tools.smartactors)
* `v` — the version of the server (default value: RELEASE)
* `sn` — the server name (default value: server)
* `rid` — the server repository id (default value: archiva.servers)
* `rurl` — the url of the server repository (default value: http://archiva.smart-tools.info/repository/servers/)
* `path` — the directory where to create the server (default value: current directory)

Run:

```console
$ das cs -v 0.6.0 -path ~/servers -sn MyServer -rid server -rurl https://repository.smart-tools.info/artifactory/smartactors_servers_dev/
```

Go to the `~/servers/MyServer/`
and check the created server structure. It should be the following:

```
|-- MyServer
    |-- core
    |   | -- ..
    |-- corefeatures
    |   | -- ..
    |-- features
    |   | -- ..
     -- server-0.6.0.jar (version of jar file may differ)
```

Use the command `dc` (download core) to download the core pack. This command supports the following arguments:

* `aid` — the artifact id of the server core pack (default value: core-pack)
* `g` — the group id of the server core-pack (default value: info.smart_tools.smartactors)
* `v` — the version of the server core-pack (default value: RELEASE)
* `rid` — the core-pack repository id (default value: archiva.smartactors-features)
* `rurl` — the url of the core-pack repository (default value: http://archiva.smart-tools.info/repository/smartactors-features/)
* `path` — the server location (default value: current directory)
* `sl` — location of a custom core-pack json

Run in server directory:

```console
$ das dc -v 0.6.0 -sl core-pack.json
```

Go to the `~/servers/MyServer/core` and check the directory core isn’t empty and contains some directories like a `base-0.6.0`, `ioc-0.6.0`, etc...

The server is installed.

Use the following command to run the server:

```console
$ java -jar ~/servers/MyServer/server-0.6.0.jar
```

The result should be:

```console
[OK] Stage 1: server core has been loaded successful.
```


## Deprecated
### Check new [quickstart_guide](../quickstart/how_to_start.md)

## Simple project: 'Hello, world!' in the console

Use the command `cp` (create project) to create the project. This command supports the following arguments:

* `pn` — the project name (default value: MyProject)
* `g` — the group id of the project (default value: com.my-project)
* `v` — the version of the project (default value: 0.1.0-SNAPSHOT)
* `path` — the directory where to create the project (default value: current directory)

Run:

```console
$ das cp -path ~/projects/
```

The result should be:

```console
Distributed Actor System. Design, assembly and deploy tools.
Version 0.6.0.
Creating project ...
Project has been created successful.
```

Go to the `~/projects/` and look for the directory `MyProject` (default project name). This directory must contains two files: `das.data` and `pom.xml`.

Use the command `cf` (create feature) to create the feature. This command supports the following arguments:

* `fn` — the feature name (mandatory argument)
* `g` — the group id of the feature (default value: the group id of the owner project)
* `v` — the version of the feature (default value: the version of the owner project)

In the project directory run:

```console
$ das cf -fn NewFeature
```

The result should be:

```console
Distributed Actor System. Design, assembly and deploy tools.
Version 0.6.0.
Creating feature ...
Feature has been created successful.
```

Go to the `~/projects/MyProject/NewFeature`. This directory must contains two files: `config.json`, `pom.xml` and the directory `NewFeatureDistribution`.

Use the command `ca` (create actor) to create the actor. This command supports the following arguments:

* `an` — the actor name (mandatory argument)
* `fn` — the name of the owner feature (mandatory argument)
* `v` — the version of the actor (default value: the version of the owner feature)

In the project directory run:

```console
$ das ca -an NewActor -fn NewFeature
```

The result should be:

```console
Distributed Actor System. Design, assembly and deploy tools.
Version 0.6.0.
Creating actor ...
Actor has been created successful.
```

Go to the `~/projects/MyProject/NewFeature`. This directory must contain a new directory `NewActor` with the following content:

* pom.xml
* src/main/java/com/my_project/new_feature/new_actor/NewActor.java
* src/main/java/com/my_project/new_feature
/new_actor/exception/NewActorException.template
* src/main/java/com/my_project/new_feature
/new_actor/wrapper/NewActorWrapper.template
* src/test/java/com/my_project/new_feature/new_actor/NewActorTest.java

Open the project into an IDE (like Intellij IDEA) and edit the file `NewFeature/NewActor/src/main/java/com/my_project/new_feature/new_actor/NewActor.java`:

```java
package com.my_project.new_feature.new_actor;

import com.my_project.new_feature.new_actor.wrapper.NewActorWrapper;

public class NewActor {

   public NewActor() {
   }

   public void handler(final NewActorWrapper wrapper)
           throws Exception {
       System.out.println(wrapper.getText());
   }
}
```

and then edit the file `NewFeature/config.json`:

```json
{
 "featureName": "com.my-project:new-feature",
 "afterFeatures": [],
 "objects": [
   {
     "name": "new-actor",
     "kind": "stateless_actor",
     "dependency": "NewActor"
   }
 ],
 "maps": [
   {
     "id": "print-to-console",
     "steps": [
       {
         "target": "new-actor",
         "handler": "handler",
         "wrapper": {
           "in_getText": "message/text"
         }
       }
     ],
     "exceptional": [
     ]
   }
 ],
 "onFeatureLoading": [
   {
     "chain": "print-to-console",
     "messages": [
       {
         "text": "Hello, world!"
       }
     ]
   }
 ]
}
```

Rename `NewFeature/NewActor/src/main/java/com/my_project/new_feature/new_actor/wrapper/NewActorWrapper.template` into the `NewFeature/NewActor/src/main/java/com/my_project/new_feature/new_actor/wrapper/NewActorWrapper.java` and update it:

```java
package com.my_project.new_feature.new_actor.wrapper;

import info.smart_tools.smartactors.iobject.iobject.exception.ReadValueException;

public interface NewActorWrapper {

   /**
    * Get the text from the text field of the message
    * @return the text
    * @throws ReadValueException if any errors occurred on reading data
    */
   String getText()
           throws ReadValueException;
}
```

Use the command `cpl` (create plugin) to create the plugin. This command supports the following arguments:

* `pln` — the plugin name (mandatory argument)
* `fn` — the name of the owner feature (mandatory argument)
* `v` — the version of the plugin (default value: the version of the owner feature)

In the project directory run:

```console
$ das cpl -pln NewActor -fn NewFeature
```

The result should be:

```
Distributed Actor System. Design, assembly and deploy tools.
Version 0.6.0.
Creating plugin ...
Plugin has been created successful.
```

Go to the `~/projects/MyProject/NewFeature`. This directory must contain a new directory `NewActorPlugin` with following content:

* pom.xml
* src/main/java/com/my_project/new_feature/new_actor_plugin/NewActorPlugin.java
* src/test/java/com/my_project/new_feature
/new_actor_plugin/NewActorPluginTest.java

Open the project in an IDE (like Intellij IDEA) and edit the file `NewFeature/NewActor/src/main/java/com/my_project/new_feature
/new_actor_plugin/NewActorPlugin.java`.

```java
package com.my_project.new_feature.new_actor_plugin;

import com.my_project.new_feature.new_actor.NewActor;
import info.smart_tools.smartactors.base.exception.invalid_argument_exception.InvalidArgumentException;
import info.smart_tools.smartactors.base.interfaces.iaction.exception.FunctionExecutionException;
import info.smart_tools.smartactors.base.strategy.apply_function_to_arguments.ApplyFunctionToArgumentsStrategy;
import info.smart_tools.smartactors.feature_loading_system.bootstrap_plugin.BootstrapPlugin;
import info.smart_tools.smartactors.feature_loading_system.interfaces.ibootstrap.IBootstrap;
import info.smart_tools.smartactors.ioc.iioccontainer.exception.RegistrationException;
import info.smart_tools.smartactors.ioc.iioccontainer.exception.ResolutionException;
import info.smart_tools.smartactors.ioc.ioc.IOC;
import info.smart_tools.smartactors.ioc.key_tools.Keys;

public class NewActorPlugin extends BootstrapPlugin {

    /**
    * The constructor.
    *
    * @param bootstrap    the bootstrap
    */
   public NewActorPlugin(final IBootstrap bootstrap) {
           super(bootstrap);
   }

   @Item("my-plugin")
   public void doSomeThing()
           throws ResolutionException, RegistrationException, InvalidArgumentException {
       IOC.register(
               Keys.getKeyByName("NewActor"),
               new ApplyFunctionToArgumentsStrategy(
                       a -> {
                           try {
                               return new NewActor();
                           } catch (Exception e) {
                               throw new FunctionExecutionException(e);
                           }
                       }
               )
       );
   }
}
```

then add to the dependencies section of the file `NewFeature/NewActorPlugin.pom`:

```xml
<dependency>
   <groupId>com.my-project</groupId>
   <artifactId>new-feature.new-actor</artifactId>
   <version>0.1.0</version>
</dependency>
```

Use the command `make` to compile and assemble the project. This command is used without arguments.

In the project directory run:

```console
$ das make
```

Each assembled feature from the project is placed into the project directory `project-distribution` like a zipped file.

Copy the assembled feature `project-distribution/new-feature-0.1.0-archive.zip` to the `~/servers/MyServer/features/` and run the server.

The result should be:

```console
[OK] Stage 1: server core has been loaded successful.


[INFO] Start unzipping feature - 'new-feature'.
[OK] -------------- Feature 'new-feature' has been unzipped successful.
[INFO] Start loading feature - 'new-feature'.
[OK] -------------- Feature - 'com.my-project:new-feature' has been loaded successful.
Hello, world!


[INFO] Feature group has been loaded: [
com.my-project:new-feature - (OK)]
```

## Additional core features

TBD
