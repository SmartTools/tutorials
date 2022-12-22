---
layout: page
title: Development, deployment
parent: User Guide
nav_order: 6
---

# Development and deployment
## Overview
Unlike other frameworks like Spring (Java) or Flask (Python), SmartActors is developed with modular development in mind. What it means is developers are able to develop features for the project independently. This, in return, puts several restrictions on development and deployment processes.

## Development
### Feature development
Although it may sound redundant, the most important thing in feature development is initial planning. Everything depends on how features are planned. Here are the questions you need to answer before developing a new feature:

* What is this feature for?
* What chains does it contain?
* Does it contain external chain?
* Can this feature be reused?

### Chain drafting
The good way to work on chains is to draft it before writing anything in `config.json`. To draft a chain, you need to answer these questions:

* What this chain should do?
* How many steps it contain?
* What actors are used in this chain?
* Are there any actors that already exists and can be used in the chain?

By answering these questions you can avoid many pitfalls like you suddenly need a new actor, old one isn't good enough, etc.

### Versioning
SmartActors allows to use different versions of each feature independently. The version of the feature is always present in the feature's name like this: `info.smart-tools:feature-name:0.1.0`. One way it could be used if there're multiple clients that aren't updated as frequently as the SmartActors application. By storing multiple versions of each feature we can avoid breaking functionality of these clients.

### Documentation
Due to the fact that developers can work independently on project features, every part of the project must be documented. For example:
* You've created a new actor? Add JavaDoc for it
* You've added a new field in wrapper? Add JavaDoc for it
* You've updated plugin so that it takes params for actors from other place? Update JavaDoc for it
* You've created the new external chain? Add API.md file for it.

By writing documentation now, you'll save time not only for your future self, but also for other developers, who may work with your feature or actor. Do not put it aside as an afterthought, do it as soon as possible.

The good way to force documentation on developers is to use both CheckStyle plugin for IntelliJ IDEA (or something similar in other IDEs) and force JavaDoc check in CI/CD configuration.

## Deployment
### Always working server
SmartActors server is developed in the way to work 24/7 without any shutdown. The only reason to restart SmartActors server is when the server (i.e. the physical or virtual machine, where SmartActors server is hosted) is rebooted or shut down for maintenance or other reasons.

It puts several restriction on the way server administrator can operate with application written with SmartActors. For example, as it was said in guide for [stateful actors](stateful_actors), server administrator must have the way to modify internal state of the server with control handlers.

### Deploying features to the server
Due to the fact that we can't simply stop server to add new feature or update existing ones, we need to do it in real time. This can be achieved with feature `RemoteManagement` and feature deployment should be done in 2 steps:

1. Upload feature to `features` directory on the server
2. Call chain `load-feature-from-file`

This way the feature will be loaded by the server and it will be available for usage.

More on feature loading can be found in [this document](../developing_server/Reloading)
