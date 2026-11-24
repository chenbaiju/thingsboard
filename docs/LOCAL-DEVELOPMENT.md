# ThingsBoard 本地开发环境与启动指南

本文针对直接从源码启动 ThingsBoard（`4.4.1-SNAPSHOT`）的本地开发场景，记录运行所需的数据库、中间件、启动类、配置项和常见错误。

---

## 0. 先说明：为什么启动 ThingsboardEdqsApplication 会报 ZooKeeper 错误

`org.thingsboard.server.edqs.ThingsboardEdqsApplication` **不是主应用**，而是 EDQS（Entity Data Query Service）微服务，属于 `msa` 微服务部署的一部分。

`edqs/src/main/resources/edqs.yml` 的默认值是：

- `zk.enabled: true` → 需要 ZooKeeper（默认 `localhost:2181`）
- `queue.type: kafka` → 需要 Kafka（默认 `localhost:9092`）
- 还需要一个正在运行的 ThingsBoard core（tb-core）向 Kafka 生产 `edqs.events`

所以单独启动 EDQS 必然失败，日志里的 `Connection refused ... 2181` 只是缺少 ZooKeeper 的第一步。日常开发请启动主应用：

```
org.thingsboard.server.ThingsboardServerApplication
```

EDQS 的完整启动方式见 [第 6 节](#6-单独启动-edqs微服务场景)。

---

## 1. 环境要求

| 项目 | 要求 |
|---|---|
| JDK | **25**（项目 `maven.compiler.release=25`） |
| Maven | 3.9.x |
| Docker | Docker Desktop（用于 PostgreSQL 等依赖） |
| Node / Yarn | 仅开发 UI 需要；Maven 构建会自动下载 |
| IDE | IntelliJ IDEA，Project SDK 选 JDK 25 |

> 注意：本机 shell 默认 `java` 可能是 21。IDEA 中请确认 Project SDK 为 `homebrew-25`；命令行运行 Maven/Java 时设置
> `export JAVA_HOME=/opt/homebrew/opt/openjdk@25/libexec/openjdk.jdk/Contents/Home`。

---

## 2. 单机（monolith）模式需要哪些依赖

默认配置（`application/src/main/resources/thingsboard.yml`）下：

| 组件 | 是否必需 | 默认值 | 说明 |
|---|---|---|---|
| PostgreSQL | **必需** | `jdbc:postgresql://localhost:5432/thingsboard`，用户/密码 `postgres/postgres` | 实体数据 + 时序数据（`DATABASE_TS_TYPE=sql`） |
| Redis / Valkey | 可选 | `CACHE_TYPE=caffeine` | 需要分布式缓存时才改 `redis` |
| Kafka | 可选 | `TB_QUEUE_TYPE=in-memory` | 只有微服务/集群部署才需要 |
| ZooKeeper | 可选 | `ZOOKEEPER_ENABLED=false` | 只有微服务/集群部署才需要 |
| Cassandra | 可选 | `DATABASE_TS_TYPE=sql` | 只有把时序库切到 Cassandra 才需要 |
| Node / Yarn | 可选 | — | 只有本地开发 UI 时需要 |

**最小可运行组合：JDK 25 + PostgreSQL + 主应用。**

### 2.1 本机端口说明

目前 `thingslink-*` 容器已停止，ThingsBoard 默认端口（5432、1883、8080 等）都空闲，可以直接使用默认配置。

如果以后需要让 thingslink 和 ThingsBoard 同时运行，注意这些冲突：

| 端口 | thingslink 容器 | 冲突处理 |
|---|---|---|
| 5432 | `thingslink-postgres` | 停掉它，或把 ThingsBoard 数据库映射到 5433 并设置 `SPRING_DATASOURCE_URL` |
| 1883 | `thingslink-emqx` | 设置 `MQTT_BIND_PORT=1884`，或 `MQTT_ENABLED=false` |
| 6379 | `thingslink-redis` | ThingsBoard 默认用 Caffeine，不需要 Redis |
| 19092 | `thingslink-redpanda` | Kafka 兼容接口，调试 EDQS 时可复用：`TB_KAFKA_SERVERS=localhost:19092` |
| 9000/9001 | `thingslink-minio` | 与 ThingsBoard 无冲突 |
| 8080/8081 | — | 主应用 8080、EDQS 8081 |
| 2181 | — | ZooKeeper，仅 EDQS/微服务调试时需要 |
| 5683-5688 / 7070 | — | CoAP / LwM2M / Edge RPC |

> 本文按默认端口（PostgreSQL 5432、MQTT 1883）说明；`tb-postgres` 已在本机按 3.1 启动。

---

## 3. 从零启动主应用（推荐流程）

### 3.1 准备数据库（Docker）

本机 Docker Hub 拉取 `postgres:16` 会超时，因此直接使用本机已有的 TimescaleDB/PostgreSQL 17 镜像
（`timescale/timescaledb-ha:pg17.4-ts2.18.2`，本身就是一个完整的 PostgreSQL，支持 `DATABASE_TS_TYPE=sql`）：

```bash
docker run -d --name tb-postgres \
  -e POSTGRES_DB=thingsboard \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -p 5432:5432 \
  -v tb-postgres-data:/home/postgres/pgdata \
  timescale/timescaledb-ha:pg17.4-ts2.18.2

# 等待就绪并确认库存在
docker exec tb-postgres pg_isready -U postgres
docker exec tb-postgres psql -U postgres -d thingsboard -Atc "select current_database();"
```

如果以后 Docker Hub 可用，也可以换成标准镜像（注意数据目录不同）：

```bash
docker run -d --name tb-postgres \
  -e POSTGRES_DB=thingsboard -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres \
  -p 5432:5432 -v tb-postgres-data:/var/lib/postgresql/data postgres:16
```

> 两种镜像的 `PGDATA` 路径不同，切换镜像前要先删除容器和卷：
> `docker rm -f tb-postgres && docker volume rm tb-postgres-data`。

端口、库名、用户、密码都与 `thingsboard.yml` 默认值一致，**不需要额外环境变量**。

等价的 `docker-compose.yml` 片段：

```yaml
services:
  postgres:
    image: timescale/timescaledb-ha:pg17.4-ts2.18.2
    container_name: tb-postgres
    environment:
      POSTGRES_DB: thingsboard
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - tb-postgres-data:/home/postgres/pgdata

volumes:
  tb-postgres-data:
```

### 3.2 编译项目（首次或改动依赖后）

```bash
# 后端 + 依赖，跳过测试、UI 和打包（最快）
mvn clean install -DskipTests -Dskip.yarn=true
```

> 如果 Maven 下载依赖超时：Maven **不读取** `HTTP_PROXY`/`HTTPS_PROXY` 环境变量，需要在
> `~/.m2/settings.xml` 的 `<proxies>` 中配置代理（本机 IDEA 的代理是 `127.0.0.1:10900`）。
> 之前失败过的下载会被缓存，重试时加 `-U`。本机已创建 `~/.m2/settings.xml` 并配置该代理。
>
> 本机 `mvn` 不在 PATH。可以直接用 IDEA 的 Maven 工具窗口执行，或使用 IDEA 自带 Maven：
> `"$HOME/Applications/IntelliJ IDEA.app/Contents/plugins/maven-plugin/lib/maven3/bin/mvn"`。
> 也可以执行 `brew install maven`。

如果报 `package org.thingsboard.server.common.msg.gen does not exist`、`MsgProtos`、`org.thingsboard.server.gen.agent.v1`
等错误，说明 protobuf 源码还没生成：

```bash
./build_proto.sh
```

生成目录：`common/*/target/generated-sources/protobuf/{java,grpc-java}`。IDEA 中重新加载 Maven 项目后，
必要时把这些目录标记为“生成的源根目录”。

### 3.3 首次安装数据库（只需执行一次）

ThingsBoard 不会在启动主服务时自动建表，首次需要运行安装程序。

IDEA 中新建一个 **Application** 运行配置：

| 配置项 | 值 |
|---|---|
| 主类 | `org.thingsboard.server.ThingsboardInstallApplication` |
| 使用模块的类路径 | `application` |
| JDK | 25 |
| 工作目录 | `$PROJECT_DIR$`（仓库根目录；设为 `application` 目录也可以） |
| 环境变量 | 无需（默认连 `localhost:5432/thingsboard`，用户/密码 `postgres/postgres`） |
| 程序参数 | 无 |
| VM options | `-Dinstall.upgrade=false -Dinstall.data_dir=$HOME/.thingsboard-dev-data`（默认就是 false） |

> **重要**：源码目录里的 `application/src/main/data` 不包含 `sql/`，SQL 脚本在 `dao/src/main/resources/sql`，
> 只有走打包流程时才会被复制到 `application/target/data`。直接从 IDEA 运行时，安装程序会报
> `NoSuchFileException: .../application/src/main/data/sql/schema-entities.sql`。先准备一个合并目录：
>
> ```bash
> DATA_DIR="$HOME/.thingsboard-dev-data"
> mkdir -p "$DATA_DIR"
> for f in "$PWD"/application/src/main/data/*; do ln -sfn "$f" "$DATA_DIR/$(basename "$f")"; done
> ln -sfn "$PWD"/dao/src/main/resources/sql "$DATA_DIR/sql"
> ```
>
> 然后在安装和主服务的 VM options 里都加上 `-Dinstall.data_dir=$HOME/.thingsboard-dev-data`。

运行后看到 `Installation finished successfully!` 即完成。它会：

- 创建全部表、视图、函数、索引
- 写入默认租户配置、系统组件、系统 Widget、通知模板、自定义菜单、LwM2M 资源等

> 安装程序根据工作目录查找 `application/src/main/data`。如果报
> `Not valid working directory ...`，请把工作目录设为仓库根目录，或增加
> `-Dinstall.data_dir=$HOME/IdeaProjects/thingsboard/application`。
>
> 重复在已有数据库上执行 install 可能失败；要重来请删除 Docker 卷后重建数据库：
> `docker rm -f tb-postgres && docker volume rm tb-postgres-data`。

### 3.4 启动主服务

再新建一个 **Application** 运行配置：

| 配置项 | 值 |
|---|---|
| 主类 | `org.thingsboard.server.ThingsboardServerApplication` |
| 使用模块的类路径 | `application` |
| JDK | 25 |
| 工作目录 | `$PROJECT_DIR$` |
| 环境变量 | `NON_PRODUCTION_USE=true`（本地开发无 license，必须设置） |
| VM options | `-Dinstall.data_dir=$HOME/.thingsboard-dev-data` |

> 如果没有设置 `NON_PRODUCTION_USE=true`，同时也没有 `TB_LICENSE_SECRET`，服务会进入 `LICENSE_REQUIRED`：
> `/api/system/setup/state` 返回 423，页面停留在许可证激活。设置后首次打开是“非生产用途确认 +
> 创建 SYS_ADMIN 账号”的初始化流程。

默认端口：

| 服务 | 端口 |
|---|---|
| Web / REST API / UI | 8080 |
| MQTT | 1883（TLS 8883） |
| CoAP | 5683（DTLS 5684） |
| LwM2M | 5685/5686/5687/5688 |
| Edge RPC | 7070 |

启动后访问 <http://localhost:8080>，首次会进入**系统初始化页面**，创建 SYS_ADMIN 账号。初始化时可以选择
**加载演示数据**，勾选后会额外创建：

- `tenant@thingsboard.org` / `tenant`
- `customer@thingsboard.org` / `customer`

（演示 Tenant 的账号不是自动创建的，只有在初始化页面勾选“load demo”或在测试代码中才会生成。）

### 3.5 本地开发 UI（可选）

后端启动在 8080 后，前端用 Angular dev server 启动，API 请求由 `ui-ngx/proxy.conf.js` 代理到 8080：

```bash
cd ui-ngx
yarn install
yarn start          # http://localhost:4200
```

如果系统没有 yarn，可以让 Maven 先下载 Node/Yarn，然后使用 `ui-ngx/target` 下的版本；或者直接跳过 UI，
先用 REST API 调后端。

---

## 4. 常用环境变量与配置开关

配置都在 `application/src/main/resources/thingsboard.yml`，通过环境变量覆盖：

| 环境变量 | 默认值 | 作用 |
|---|---|---|
| `HTTP_BIND_PORT` | `8080` | Web/API 端口 |
| `SPRING_DATASOURCE_URL` | `jdbc:postgresql://localhost:5432/thingsboard` | 主库 JDBC URL |
| `SPRING_DATASOURCE_USERNAME` | `postgres` | 数据库用户 |
| `SPRING_DATASOURCE_PASSWORD` | `postgres` | 数据库密码 |
| `DATABASE_TS_TYPE` | `sql` | 时序库：`sql` / `timescale` / `cassandra` |
| `CACHE_TYPE` | `caffeine` | `caffeine` 或 `redis` |
| `REDIS_HOST` / `REDIS_PORT` | `localhost` / `6379` | Redis 连接 |
| `TB_QUEUE_TYPE` | `in-memory` | `in-memory` 或 `kafka` |
| `TB_KAFKA_SERVERS` | `localhost:9092` | Kafka 地址 |
| `ZOOKEEPER_ENABLED` | `false` | 是否启用 ZK 发现 |
| `ZOOKEEPER_URL` | `localhost:2181` | ZK 地址 |
| `MQTT_ENABLED` / `MQTT_BIND_PORT` | `true` / `1883` | MQTT Transport |
| `COAP_ENABLED` | `true` | CoAP Transport |
| `LWM2M_ENABLED` | `true` | LwM2M Transport |
| `TB_SERVICE_TYPE` | `monolith` | `monolith` / `tb-core` / `tb-rule-engine` |
| `TB_EDQS_MODE` | `local` | `local`（主应用进程内）或 `remote`（独立 EDQS） |
| `TB_EDQS_SYNC_ENABLED` | `false` | core 是否把数据同步到 EDQS Kafka 主题 |
| `TB_EDQS_API_SUPPORTED` | `false` | 是否把实体查询转发给 EDQS |
| `NON_PRODUCTION_USE` | `false` | 本地无 license 开发必须设为 `true` |
| `TB_LICENSE_SECRET` | 空 | 有商业 license 时填写；与 `NON_PRODUCTION_USE` 二选一 |

开发时想减少启动负担，可以关闭不用的协议：

```bash
MQTT_ENABLED=false COAP_ENABLED=false LWM2M_ENABLED=false SNMP_ENABLED=false
```

---

## 5. 数据与端口占用排查

```bash
# 查看容器
docker ps

# 查看 PostgreSQL 日志
docker logs -f tb-postgres

# 检查端口占用（macOS）
lsof -nP -iTCP:5432 -sTCP:LISTEN
lsof -nP -iTCP:8080 -sTCP:LISTEN
```

重置本地开发数据库：

```bash
docker rm -f tb-postgres
docker volume rm tb-postgres-data
# 然后重新执行 3.1 + 3.3
```

---

## 6. 单独启动 EDQS（微服务场景）

只有需要调试 EDQS 微服务时才这样做。EDQS 不连 PostgreSQL，它通过 Kafka 从 core 接收实体变更事件，
因此需要完整的消息基础设施。

### 6.1 准备 ZooKeeper 和 Kafka

本机已有 `thingslink-redpanda` 镜像（Kafka 兼容，端口 19092），需要时可以重新启动它，无需再拉 Kafka：

```bash
docker start thingslink-redpanda     # 监听 localhost:19092
```

ZooKeeper 本机没有现成镜像，需要时再启动（2181 空闲，但 Docker Hub 拉取可能超时）：

```bash
docker run -d --name tb-zookeeper -p 2181:2181 zookeeper:3.9
```

用 Redpanda 时，下面所有 `TB_KAFKA_SERVERS` 都写 `localhost:19092`。如果启动独立的 Kafka
（9092 空闲），则写 `localhost:9092`。

### 6.2 启动 core（主应用）并切到 Kafka 模式

主应用的环境变量：

```bash
TB_QUEUE_TYPE=kafka
TB_KAFKA_SERVERS=localhost:19092   # 复用 Redpanda；用新建 Kafka 则为 localhost:9092
ZOOKEEPER_ENABLED=true
ZOOKEEPER_URL=localhost:2181
TB_EDQS_MODE=remote
TB_EDQS_SYNC_ENABLED=true
TB_EDQS_API_SUPPORTED=true        # 或等同步完成后再开启
```

### 6.3 启动 EDQS

EDQS 运行配置（`org.thingsboard.server.edqs.ThingsboardEdqsApplication`，模块 `edqs`）：

```bash
HTTP_BIND_PORT=8081                # 默认也是 8080，会与主应用冲突
TB_QUEUE_TYPE=kafka
TB_KAFKA_SERVERS=localhost:19092   # 与 core 保持一致
ZOOKEEPER_ENABLED=true
ZOOKEEPER_URL=localhost:2181
```

注意：

- EDQS 默认端口 8080，若与 core 同时运行必须改端口。
- core 与 EDQS 必须使用同一套 Kafka topic（默认 `edqs.events` / `edqs.state` / `edqs.requests` / `edqs.responses`）。
- EDQS 数据来自 Kafka 事件同步；只启动 EDQS 而没有 core，仍然无法提供查询服务。

---

## 7. 常见错误对照表

| 现象 | 原因 | 处理 |
|---|---|---|
| `Connection refused ... localhost:2181` | ZooKeeper 未启动；或误启动了 EDQS 微服务 | 日常开发改启动 `ThingsboardServerApplication` |
| `Connection refused ... localhost:9092` | Kafka 未启动 | 启动 Kafka，或把 `TB_QUEUE_TYPE` 改回 `in-memory` |
| `Connection refused ... localhost:5432` / `FATAL: database "thingsboard" does not exist` | PostgreSQL 未启动或库名不对 | 按 3.1 启动，确认 `POSTGRES_DB=thingsboard` |
| `relation "..." does not exist` / 启动后大量 SQL 错误 | 没执行首次安装 | 执行 3.3 的 `ThingsboardInstallApplication` |
| `Not valid working directory` | 安装程序找不到 `application/src/main/data` | 工作目录设为仓库根目录，或设置 `-Dinstall.data_dir` |
| `NoSuchFileException: .../application/src/main/data/sql/schema-entities.sql` | 源码目录没有 `sql/`，打包时才从 dao 复制 | 按 3.3 准备合并的 `install.data_dir` |
| `/api/system/setup/state` 返回 423、页面停在许可证激活 | 未设置无 license 模式 | 设置 `NON_PRODUCTION_USE=true` 后重启 |
| `package org.thingsboard.server.common.msg.gen does not exist` / 找不到 `MsgProtos` | protobuf 源码未生成 | 运行 `./build_proto.sh`，再刷新 IDEA Maven 项目 |
| `Port 8080 was already in use` | 端口被占用，或 core/EDQS 同时用 8080 | 改 `HTTP_BIND_PORT`，或停掉占用进程 |
| `Port 1883 was already in use` | 重新启动了 `thingslink-emqx` | 设置 `MQTT_BIND_PORT=1884`，或 `MQTT_ENABLED=false` |
| `database "thingsboard" does not exist` / 连到了 thingslink 的库 | 5432 上是 `thingslink-postgres` | 停掉它并启动 `tb-postgres`，或把 `tb-postgres` 映射到 5433 后设置 `SPRING_DATASOURCE_URL` |
| Testcontainers: `unsupported Docker API version` | Docker API 版本过新 | Docker daemon 配置 `"min-api-version": "1.32"` 后重启 Docker |
| 访问 8080 没有页面 / 404 | 构建时用了 `-Dskip.yarn=true`，UI 未打进 classpath | 用 `ui-ngx` 的 `yarn start`（4200），或构建 UI |

---

## 8. 参考

- `CONTRIBUTING.md`：构建、测试、许可证头、PR 规范
- `CLAUDE.md`：项目结构与常用命令速查
- `application/src/main/resources/thingsboard.yml`：全部配置项及默认值
- `edqs/src/main/resources/edqs.yml`：EDQS 默认配置
- 官方文档：<https://thingsboard.io/docs/pe/>
