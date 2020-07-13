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
	bash build-features.sh false \
	mkdir -p ./ServerParts/features;

.PHONY: install_features_without_tests
install_features_without_tests:
	mkdir -p ./project-distribution; \
	bash build-features.sh true \
	mkdir -p ./ServerParts/features;

.PHONY: install_only_modified_features
install_only_modified_features:
	mkdir -p ./project-distribution; \
	bash build-only-modified-features.sh false

.PHONY: install_only_modified_features_without_tests
install_only_modified_features_without_tests:
	mkdir -p ./project-distribution; \
	bash build-only-modified-features.sh true

.PHONY: start_server
start_server:
	cd ./ServerParts; \
	java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005 -jar ./server.jar; \
	cd ..

.PHONY: start_server_without_logs
start_server_without_logs:
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
	cd Features && mvn archetype:generate                                       \
      -DarchetypeGroupId=info.smart-tools.common                                \
      -DarchetypeArtifactId=feature-archetype                                   \
      -DarchetypeVersion=${FEATURE_ARCHETYPE_VERSION}                           \
      -DgroupId=${PROJECT_GROUP_ID}                                             \
      -DgroupPackage=${PROJECT_GROUP_PACKAGE}                                   \
      -DartifactId=${artId}                                                     \
      -DartifactPackage=${artPack}                                              \
      -DrepoId=${REPO_ID}                                                       \
      -DrepoUrl=${ARTIFACTORY_REPO}                                             \
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
