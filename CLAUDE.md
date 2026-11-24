# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

ThingsBoard (`4.4.1-SNAPSHOT`) — a multi-module Maven IoT platform. Backend is Java 25 on Spring Boot
4.1.x; the UI is Angular 20 in `ui-ngx`. There is no Maven wrapper; use a system Maven 3.9.x.

## Build commands

```bash
# Backend only, fastest: skips UI build and every packaging step (deb/rpm/zip/boot jar)
mvn clean install -T6 -DskipTests -Dskip.yarn=true

# Same without -T, as documented in CONTRIBUTING
mvn clean install -DskipTests -Dpkg.skip=true

# Compile only the modules you touched, and their upstream dependencies
mvn -pl <module> -am -DskipTests -Dskip.yarn=true test-compile
```

- `-Dskip.yarn=true` skips the Angular build (`ui-ngx`, `msa/web-ui`, `msa/web-report`) **and** all
  packaging (root `skip-pkg-without-yarn` profile).
- `-Dpkg.skip=true` skips only packaging (boot jar, deb, rpm, zip) and still builds the UI. Use the
  finer flags `-Dpkg.skip.bootjar|deb|rpm|zip=true` if needed; see CONTRIBUTING.md for the matrix.
- `-Dskip.ui.build=true` skips only the UI modules (but leaves `msa/js-executor` yarn work running).
- The runnable artifact is `application/target/thingsboard-4.4.1-SNAPSHOT-boot.jar`; main class is
  `org.thingsboard.server.ThingsboardServerApplication`.

### Generated protobuf sources

Proto classes are **not checked in**. `protobuf-maven-plugin` runs `protoc` plus `protoc-gen-grpc-java`
into `target/generated-sources/protobuf/{java,grpc-java}` during `generate-sources`. If compilation
fails with `package org.thingsboard.server.common.msg.gen does not exist`,
`org.thingsboard.server.gen.agent.v1`, `MsgProtos`, `QueueProtos`, etc., generation has not run:

```bash
./build_proto.sh     # the project's own script: mvn clean compile for every proto-bearing module
```

or equivalently:

```bash
mvn -pl common/proto,common/message,common/agent-api,common/cluster-api,common/edge-api,common/transport/coap,common/transport/mqtt -am compile
```

Notes:
- `protobuf.version` is pinned to 3.25.5 even though Spring Boot manages 4.x; do not bump it casually.
- The plugin artifacts (`protoc`, `protoc-gen-grpc-java`) must be downloadable. Maven does not read
  `HTTP_PROXY`/`HTTPS_PROXY` env vars — proxy settings must be in `~/.m2/settings.xml`. A failed
  download leaves a `.lastUpdated` marker in `~/.m2`; rerun with `-U` to retry.
- In IntelliJ, Maven → "Reload All Maven Projects" usually registers the generated dirs. If it does not,
  `common/message`, `common/agent-api`, `common/proto` and `common/edge-api` lack a
  `build-helper-maven-plugin` declaration, so mark `target/generated-sources/protobuf/java` (and
  `protobuf/grpc-java` for `agent-api`) as a Generated Sources Root manually.

## Tests

Install all modules first (`mvn install -DskipTests -Dskip.yarn=true`) if running tests module-scoped.

```bash
# One module
mvn -pl <module> test

# Single test class / method
mvn -pl application -Dtest='DeviceControllerTest#testSaveDevice' test
# With -am, add -DfailIfNoTests=false so upstream modules without that test don't fail

# Fast compile check before a commit
mvn -pl <module> -am -DskipTests -Dskip.yarn=true test-compile
```

- `application` tests: `AbstractWebTest` / `AbstractControllerTest` (controller and integration tests),
  `AbstractIntegrationTest`. `application/src/test/resources/application-test.properties` disables
  transports, edge, agents and integrations for fast context startup; individual tests re-enable what
  they need with `@TestPropertySource`.
- `dao` tests: `AbstractDaoServiceTest`, `AbstractJpaDaoTest`, `AbstractServiceTest`.
- Many `dao`/`application` tests start PostgreSQL/Cassandra/brokers via Testcontainers and need Docker.
  If containers fail with "unsupported Docker API version", set `"min-api-version": "1.32"` in the Docker
  daemon config (see CONTRIBUTING.md).
- The full-suite commands with parallelism/flake settings are in CONTRIBUTING.md ("Running tests").
- Black-box tests live in `msa/black-box-tests`; they need locally built Docker images. See
  `msa/black-box-tests/README.md` before using them.

## UI

`ui-ngx/structure.md` is a detailed navigation guide (routes → components, path aliases, entity groups).
The Maven build provisions Node/Yarn through `frontend-maven-plugin`; for UI-only work:

```bash
cd ui-ngx
yarn install
yarn start       # dev server with live reload
yarn build:prod
yarn lint
```

## License headers

Every source file needs the project SPDX header; do not add personal copyright/author lines.

```bash
mvn license:format                       # Mycila, formats headers
./check-license-headers.sh --fix --since origin/master   # fork-specific provenance checker
```

CI runs `check-license-headers.yml` and `check-configuration-files.yml` (the latter runs
`tools/src/main/python/check_yml_file.py` on `thingsboard.yml` and the per-service `tb-*.yml` files).

## Architecture

### Reactor layout

| Module | Contents |
|---|---|
| `common` | Shared models, messaging, queues, caches, protobuf/gRPC contracts, transport APIs |
| `dao` | Persistence: entity services, SQL/JPA, NoSQL/Cassandra, timeseries |
| `rule-engine` | `rule-engine-api` (node SPI) and `rule-engine-components` (built-in nodes) |
| `transport` | HTTP, MQTT, CoAP, LwM2M, SNMP transport servers |
| `integration` | Connectors (HTTP, MQTT, Kafka, Pub/Sub, AWS, Azure, OPC-UA, …) and the integration executor |
| `application` | Monolith: Spring Boot app assembling all of the above |
| `msa` | Microservice packaging: one Docker image per service + black-box tests |
| `edqs` | Entity Data Query Service (standalone/cluster query service) |
| `report`, `monitoring`, `migrator` | Standalone services |
| `ui-ngx` | Angular UI |
| `packaging` | Gradle plugin run from Maven for deb/rpm/zip artifacts |
| `tools`, `rest-client`, `netty-mqtt` | Tooling, REST client, standalone MQTT client |

### Message flow

Transports never touch the database. A device message enters a transport server
(`transport/*`, shared code in `common/transport/transport-api`), is encoded as a
`TbProtoQueueMsg<TransportApiRequestMsg>` and sent through the queue abstraction
(`common/queue`). The core service consumes it in `DefaultTransportApiService`
(`application/service/transport`) and dispatches an actor message. Rule-engine output, RPC, telemetry
writes and responses travel back over the same queue.

`common/message` defines `TbMsg`, the immutable rule-engine message; its metadata/payload protobuf is
`common/message/src/main/proto/tbmsg.proto`. `common/proto` holds queue/transport/integration gRPC and
protobuf contracts.

### Actor model and rule engine

The core is an actor system (`common/actor`, `DefaultTbActorSystem`). Actor hierarchy in
`application/actors`:

```
AppActor
 └─ TenantActor (per tenant)
     ├─ RuleChainActor (per rule chain)
     │   └─ RuleNodeActor (per node instance)
     ├─ DeviceActor (per device session / RPC state)
     └─ CalculatedFieldActor
```

Actors are addressable cluster-wide (`common/cluster-api`, actor RPC over the queue), which is what lets
the monolith and the `msa` services scale horizontally. Rule-chain topology comes from the DB; each node
runs as a `RuleNodeActor` executing an implementation of `TbNode`
(`rule-engine/rule-engine-api`): `init(TbContext, TbNodeConfiguration)`, `onMsg(TbContext, TbMsg)`,
optional `destroy()`/`upgrade()`. Built-in nodes live in `rule-engine/rule-engine-components` and are
annotated with `@RuleNode`; `AnnotationComponentDiscoveryService`
(`application/service/component`) discovers them and builds the component descriptors exposed by the
API/UI. To add a rule node: implement `TbNode` + configuration in `rule-engine-components`, add
`@RuleNode`, register any new dependency in the module's pom.

### Persistence

`common/data` holds entities and value objects. `dao` pairs each entity with a service
(`dao/entity`, `dao/device`, …) over JPA repositories (`dao/sql`) and optionally NoSQL implementations
(`dao/nosql` for Cassandra, `dao/sqlts` for SQL timeseries). The timeseries backend is selected at
runtime with `DATABASE_TS_TYPE` (`sql`, `timescale`, `cassandra`; hybrid mixes them). Entity/attribute
caches are in `common/cache` (Caffeine in-memory, Redis when configured). DAO Spring configuration
classes live in `dao/config`.

### Queues

`common/queue` abstracts producers/consumers (`TbQueueProducer`/`TbQueueConsumer`) over in-memory queues
(monolith) and Kafka (`queue/kafka`, `queue/provider`). Queue topics, partitions and consumer groups are
configured in `thingsboard.yml`; per-service queue factories in `queue/provider` define which topics each
service (core, transport, rule engine, integration executor, report, EDQS, version control) produces and
consumes.

### Configuration

- Monolith: `application/src/main/resources/thingsboard.yml`. Values use `${ENV_VAR:default}` placeholders,
  so the same build is configured by environment variables (DB URL, `DATABASE_TS_TYPE`, queue type,
  transports, etc.).
- Microservices: additional per-service files, e.g. `transport/*/src/main/resources/tb-*-transport.yml`,
  `integration/executor/src/main/resources/tb-integration-executor.yml`,
  `msa/vc-executor/src/main/resources/tb-vc-executor.yml`. The CI yml checker validates them.
- MSA images in `msa/*` run the same `application` code with a service-specific Spring entry point and
  config; the `application` module remains the monolith.

## Conventions

- Lombok is used throughout (constructor injection, `@Slf4j`, `@Data`, `@Builder`); see `lombok.config`.
  `lombok.anyconstructor.addconstructorproperties = true`, and `@Lazy` is copied onto generated
  constructors.
- Tests use JUnit 5 + Mockito + AssertJ.
- Do not edit anything under `target/`; it is generated (protobuf, Lombok, JPA metamodel).
- PR titles are used verbatim in release notes: past tense, start with `Fixed`/`Added`/`Updated`/etc.,
  no `fix(scope):` prefixes or ticket keys. Areas come from PR labels, not titles.
