---
layout: page
title: Plugins
parent: "Developing Server"
nav_order: 1
---

# Plugins

## Overview

Plugin is a set of functionality, i.e. Java classes packed into one or multiple JAR files.
The Server loads and initializes such JARs.

## Loading and initialization steps

Loading and initializing are two separate steps of the process.

At the first step a part of the server, the [`PluginLoader`](../apidocs/info/smart_tools/smartactors/core/plugin_loader_from_jar/PluginLoader.html) scans the JAR files for implementations of [`IPlugin`](../apidocs/info/smart_tools/smartactors/core/iplugin/IPlugin.html).
It uses [`PluginCreator`](../apidocs/info/smart_tools/smartactors/core/plugin_creator/PluginCreator.html) to instantiate the Plugin by passing the instance of `IBootstrap` to it's constructor.
The `load()` method of each Plugin is called here.

In it's `load()` method the Plugin adds it's own [`IBootstrapItem`](../apidocs/info/smart_tools/smartactors/core/ibootstrap_item/IBootstrapItem.html) into the [`IBootstrap`](../apidocs/info/smart_tools/smartactors/core/ibootstrap/IBootstrap.html) known to him.
The `IBootstrapItem` gives the name of the part of the functionality provided by the plugin and allows to declare dependencies from the other bootstrap items using the `after()` and `before()` methods.
Finally the sequence of the bootstrap items according to the dependencies is constructed and their `executeProcess()` is called.
This is the second step of the initialization.

![Plugins and Bootstrap](http://www.plantuml.com/plantuml/img/IyxFBSZFIyqhKGXEBIfBBU9AXWi4v9IcP-OX2JZbvvSKbnGb5c0Jyon9pUNYWXYYaA-hQwUWfAK4DKF1IY4dFp6b65KUhXKedLgHcbnQaWfK0TMX1JC1nGAWoeAY_BBC591AX7wSYfFpSt9IaqkG5ODba6s7AarAJSilIYMiBZ6j11XA0000)

## Sample plugin

Let create a sample plugin.

```java
public class SamplePlugin implements IPlugin
```

### Bootstrap reference    

It must contain a reference to `Bootstrap` provided during initialization process.

```java
private final IBootstrap<IBootstrapItem<String>> bootstrap;
```

The `Bootstrap` must be injected into the plugin's constructor.

```java
public SamplePlugin(final IBootstrap<IBootstrapItem<String>> bootstrap) {
    this.bootstrap = bootstrap;
}
```

### Load method    

You have to define the `load()` method.

```java
@Override
public void load() throws PluginException {}
```

In the method at the first you should declare the `BootstrapItem` provided by your plugin.

```java
IBootstrapItem<String> item = new BootstrapItem("SamplePlugin");
```

Here you declare that your item must be initialized after initialization of `'IOC'` item.

```java
item.after("IOC");
```

### Initialization callback    

Then you define your plugin initialization code.
For example, let register some new strategy in IOC.

```java
item.process(() -> {
    try {
        IKey key = Keys.getKeyByName("new SampleClass");
        IOC.register(
            key,
            // it's the initialization action of our plugin
            new CreateNewInstanceStrategy(
                (args) -> new SampleClass((String) args[0])
            )
        );
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
    System.out.println("SamplePlugin initialized");
});
```

### Registering the callback    

Finally you should add your bootstrap item into `Bootstrap`:

```java
bootstrap.add(item);
```

## How plugins are loaded

On the server side the loading of the plugin may looks as the following.

### Bootstrap    

The server initializes the `Bootstrap` to handle the initialization sequence.

```java
IBootstrap bootstrap = new Bootstrap();
```

### PluginCreator

Also it uses `PluginCreator` to find and call the correct constructor of `IPlugin`.

```java
IPluginCreator creator = new PluginCreator();
```

### PluginLoaderVisitor    

A `IPluginLoaderVisitor` is necessary to track plugins loading process.

```java
IPluginLoaderVisitor<String> visitor = new SamplePluginVisitor();
```

### ModuleManager    

Because the plugin jars can be located in different directories and can be added dynamically, the special object named 'module' needs to be created to store each plugin inside the server.

```java
ModuleManager.addModule(pluginId, pluginName, pluginVersion);
```

### PluginLoader

Then this 'module' for plugin must be setup as current and `PluginLoader` is created.
The second constructor's parameter is the code to load each plugin.

```java
ModuleManager.setCurrentModule(ModuleManager.getModuleById(pluginId));
IPluginLoader<Collection<IPath>> pluginLoader = new PluginLoader(
    ModuleManager.getCurrentClassLoader(),
    (t) -> {
        try {
            IPlugin plugin = creator.create(t, bootstrap);
            plugin.load();
        } catch (Exception e) {
            throw new RuntimeException("Could not create instance of IPlugin", e);
        }
    },
    visitor
);
```

### Loading    

The server scans the collection of jar files for `IPlugin` instances.

```java
Collection<IPath> fileCollection = new ArrayList<>();
fileCollection.add(new Path("libs/SamplePlugin.jar"));
fileCollection.add(new Path("libs/IocPlugin.jar"));
pluginLoader.loadPlugin(fileCollection);
```

### Initialization    

And then calls the initialization code of `BootstrapItem`s in the sequence according to their dependencies.

```java
bootstrap.start();
```

## Initialization order    

In this example, when we have two plugins: 'SamplePlugin' following _after_ 'IOC' — the loading and initialization sequence is like this:

1. `SamplePlugin.load()`, registers bootstrap item
2. `IOCPlugin.load()`, registers bootstrap item
3. initialization code of bootstrap item of 'IOC' plugin
4. initialization code of bootstrap item of 'SamplePlugin' because it asked to be after 'IOC'

Note the order of first two steps is not defined, while the order of execution of bootstrap items is defined by their dependencies.

## Code of the example

You can check the full source codes of this example [here](../xref/info/smart_tools/smartactors/core/examples/plugin/package-summary.html).
