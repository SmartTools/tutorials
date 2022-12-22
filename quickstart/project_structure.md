---
layout: page
title: Project Structure
parent: Quickstart
nav_order: 2
---

# Dirs structure

```
|-- Project
    |-- Features
    |   |-- feature-name
    |   |   |-- src/java/project
    |   |   |   |-- actors
    |   |   |   |   |-- ActorName
    |   |   |   |   |   |-- ActorName.java
    |   |   |   |   |   |-- ActorWrapper.java
    |   |   |   |   |   |-- ActorException.java
    |   |   |   |   |   |-- package-info.java
    |   |   |   |-- plugins
    |   |   |   |   |-- ActorNamePlugin
    |   |   |   |   |   |-- ActorNamePlugin.java
    |   |   |   |   |   |-- package-info.java
    |   |   |-- test/java/project
    |   |   |   |-- ...
    |   |   |-- API.md
    |   |   |-- bin.xml
    |   |   |-- config.json
    |   |   |-- pom.xml
    |   |   |-- README.md
    |   |-- ...
    |-- ServerParts
    |   |-- corefeatures
    |   |    |-- features.json
    |   |-- core-pack.json
    |   |-- configuration.json
    |   |-- server.jar
    |-- Makefile
    |-- README.md
    |-- .gitignore
    |-- .editorconfig
    |-- docker-compose.yml ## this is if you would like to use docker.
```

# Configuration files
## .gitignore

Supposing you use git, gitignore must have this lines:

```
# IntelliJ IDEA project files
**/.idea
**/*.iml

# Eclipse project files
**/.project
**/.settings
**/.classpath

# Build files
.class
**/target
**/*.jar
project-distribution/*

# Don't ignore server.jar
!ServerParts/server.jar

ServerParts/corefeatures/*
# Needed for corefeatures downloading
!ServerParts/corefeatures/features.json

ServerParts/core/*
ServerParts/downloads/*
ServerParts/features/*
# Needed for user features downloading
!ServerParts/features/features.json

**/.allure
allure-results/*

**/docs
```

## .env
Some properties for development environment can be stored in .env file like this:
```shell
POSTGRES_DB=postgres_db
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DOCKER_CONTAINER_NAME=your_project_postgresql

FEATURE_ARCHETYPE_VERSION=current-archetype-version
PROJECT_GROUP_ID=com.example.your-project
PROJECT_GROUP_PACKAGE=com.example.your_project
REPO_ID=your_project
TEST_SERVER_HOST=your-project-test-server.lan
ARTIFACTORY_REPO=url-to-your-artifactory-repository
```

## Makefile

To specify the way of building system strictly we will use `Makefile`. Do not use instructions in the readme, because you'll definitely reach instructions-hell.

```make
include .env
export $(shell sed 's/=.*//' .env)

SUDO=$(shell getent group docker | grep -q $$USER || echo sudo)

.PHONY: full_restart
full_restart: restart_db_docker download_core start

.PHONY: start
start: install_features start_server

.PHONY: download_core
download_core:
	cd ./ServerParts; \
	mkdir -p ./core; \
	das dc -sl ./core-pack.json; \
	cd ..

.PHONY: install_feature
install_feature:
	mkdir -p ./project-distribution; \
	bash build-feature.sh ${feature} ${skipTests} \
	mkdir -p ./ServerParts/features;

.PHONY: install_features
install_features:
	mkdir -p ./project-distribution; \
	bash build-features.sh false

.PHONY: start_server
start_server:
	cd ./ServerParts; \
	java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005 -jar ./server.jar; \
	cd ..

###

.PHONY: start_db_docker
start_db_docker:
	$(SUDO) docker-compose up postgresql; \
	cd ..

.PHONY: shutdown_db_docker
shutdown_db_docker:
	$(SUDO) docker-compose down --rmi all; \
	cd ..

.PHONY: restart_db_docker
restart_db_docker: shutdown_db_docker start_db_docker

.PHONY: connect_to_psql_docker
connect_to_psql_docker:
	$(SUDO) docker exec -it ${POSTGRES_DOCKER_CONTAINER_NAME} psql ${POSTGRES_DB} -U ${POSTGRES_USER}

###

.PHONY: checkstyle
checkstyle:
	cd ./Features; \
	mvn checkstyle:check; \
	cd ..

.PHONY: test
test:
	cd ./Features; \
	mvn test; \
	cd ..

###

.PHONY: cf
cf:
	cd Features && mvn archetype:generate            \
      -DarchetypeGroupId=info.smart-tools.common     \
      -DarchetypeArtifactId=feature-archetype        \
      -DarchetypeVersion=${FEATURE_ARCHETYPE_VERSION}\
      -DgroupId=${PROJECT_GROUP_ID}                  \
      -DgroupPackage=${PROJECT_GROUP_PACKAGE}        \
      -DartifactId=${artId}                          \
      -DartifactPackage=${artPack}                   \
      -DrepoId=${REPO_ID}                            \
      -DrepoUrl=${ARTIFACTORY_REPO}                  \
      -Dversion=0.1.0

.PHONY: deploy
deploy:
	rsync -avzr --progress ./ServerParts/ deploy@${TEST_SERVER_HOST}:/home/deploy/toDeploy/backend/

.PHONY: build_docs
build_docs:
	docker-compose run --rm doc-builder build-pages

.PHONY: build_docs_git
build_docs_git:
	docker-compose run --rm doc-builder build-pages-git

```

### Feature compiling
To compile only modified features, this script can be used.

**Important note**: this feature will compile only those feature that can be seen with `git status` command. If feature does not appear as a result of checking status of project's git repository, it won't compile.

```shell
#!/usr/bin/env bash

for dir in Features/*/; do
  {
    git status | grep "${dir}" &&
    cd "${dir}" &&
    mvn clean install \
      -Dmaven.test.skip=$1 \
      -Dbuild.format=jar \
      -Dbuild.unpack=true \
      -Dbuild.includeBaseDirectory=false \
      -Dbuild.exclude=**/** &&
      rm -rf ./target/*sources.jar &&
      rm -rf ./target/*javadoc.jar &&
      mv ./target/*.jar ../../project-distribution &&
      cd -
  }
done
files=(./project-distribution/*.jar)
if [ -f "${files[0]}" ]; then
  mv ./project-distribution/*.jar ./ServerParts/features
fi

```

## .editorconfig

In our projects I recommend to use this editor configuration.

```ini
# EditorConfig is awesome: http://EditorConfig.org

# top-most EditorConfig file
root = true

# Unix-style newlines with a newline ending every file
[*]
end_of_line = lf
insert_final_newline = true
charset = utf-8

# Tab indentation (no size specified)
[Makefile]
indent_style = tab

# Matches the exact files either package.json or .travis.yml
[*.{json,yml,yaml}]
indent_style = space
indent_size = 2

[*.{xml,java}]
indent_style = space
indent_size = 4
```
