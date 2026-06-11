# SigNoz Docker 部署包

自包含的 SigNoz Docker Compose 部署包，支持在同一台服务器上部署多个独立实例。

## 目录结构

```
signoz-docker/
├── docker-compose.yaml                     # Docker Compose 主文件（参数化）
├── otel-collector-config.yaml              # OpenTelemetry Collector 配置
├── .env.instance-a                         # 实例 A 示例配置
├── .env.instance-b                         # 实例 B 示例配置
├── common/                                 # 共享配置文件
│   ├── clickhouse/
│   │   ├── config.xml                      # ClickHouse 主配置
│   │   ├── users.xml                       # ClickHouse 用户配置
│   │   ├── custom-function.xml             # ClickHouse 自定义函数
│   │   ├── cluster.xml                     # ClickHouse 集群配置
│   │   └── user_scripts/                   # ClickHouse UDF 脚本目录
│   └── signoz/
│       └── otel-collector-opamp-config.yaml
├── data/                                   # 运行时自动创建
│   ├── clickhouse/                         # ClickHouse 时序数据
│   ├── signoz/                             # SigNoz 元数据（SQLite）
│   └── zookeeper/                          # ZooKeeper 状态数据
└── README.md
```

## 前置条件

- Docker Engine >= 20.10
- Docker Compose V2 (`docker compose` 命令)
- 服务器可访问 GitHub（`init-clickhouse` 启动时需从 GitHub Releases 下载二进制文件）
- 建议最低 4GB 内存，8GB+ 为佳

## 服务组成

| 服务 | 镜像 | 说明 |
|------|------|------|
| init-clickhouse | `clickhouse/clickhouse-server:${CLICKHOUSE_TAG}` | 初始化脚本（一次性） |
| zookeeper-1 | `signoz/zookeeper:${ZOOKEEPER_TAG}` | ClickHouse 协调服务 |
| clickhouse | `clickhouse/clickhouse-server:${CLICKHOUSE_TAG}` | 时序数据库 |
| signoz | `signoz/signoz:${VERSION}` | SigNoz 主服务（Web UI + API） |
| otel-collector | `signoz/signoz-otel-collector:${OTELCOL_TAG}` | OpenTelemetry 数据收集器 |
| signoz-telemetrystore-migrator | `signoz/signoz-otel-collector:${OTELCOL_TAG}` | 数据库迁移工具（一次性） |

所有镜像版本均通过 `.env` 配置，未设置时使用 compose 文件中的默认值。

## 暴露端口

| 端口 | 协议 | 用途 |
|------|------|------|
| 8080 | HTTP | SigNoz Web UI |
| 4317 | gRPC | OTLP gRPC Receiver（接收 Traces/Metrics/Logs） |
| 4318 | HTTP | OTLP HTTP Receiver（接收 Traces/Metrics/Logs） |

## 快速开始

```bash
# 1. 将整个目录复制到服务器
scp -r signoz-docker/ user@server:/opt/signoz-docker/

# 2. 为每个实例创建独立目录，配置 .env
cp -r /opt/signoz-docker /opt/instance-a
cp /opt/instance-a/.env.instance-a /opt/instance-a/.env

# 3. 启动
cd /opt/instance-a && docker compose up -d

# 4. 访问 Web UI
# http://<server-ip>:8080
```

## 环境变量说明

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `COMPOSE_PROJECT_NAME` | Docker Compose 项目名 | `signoz` |
| `INSTANCE` | 实例标识，用于容器名/卷名/网络名前缀 | - |
| `SIGNOZ_PORT` | SigNoz Web UI 端口 | `8080` |
| `OTLP_GRPC_PORT` | OTLP gRPC 接收端口 | `4317` |
| `OTLP_HTTP_PORT` | OTLP HTTP 接收端口 | `4318` |
| `JWT_SECRET` | JWT 签名密钥（生产环境务必修改） | `secret` |
| `VERSION` | SigNoz 镜像版本 | `v0.128.0` |
| `OTELCOL_TAG` | OTel Collector 镜像版本 | `v0.144.5` |
| `CLICKHOUSE_TAG` | ClickHouse 镜像版本（建议跟随 SigNoz 官方钉定版本） | `25.5.6` |
| `ZOOKEEPER_TAG` | ZooKeeper 镜像版本（建议跟随 SigNoz 官方钉定版本） | `3.7.1` |

## 多实例部署

每个实例使用独立目录，拥有独立的容器、网络、数据和端口，互不干扰。

```bash
# 实例 A
cp -r /opt/signoz-docker /opt/instance-a
cp /opt/instance-a/.env.instance-a /opt/instance-a/.env
cd /opt/instance-a && docker compose up -d

# 实例 B
cp -r /opt/signoz-docker /opt/instance-b
cp /opt/instance-b/.env.instance-b /opt/instance-b/.env
cd /opt/instance-b && docker compose up -d
```

启动后：

- 实例 A: `http://<server-ip>:8080`
- 实例 B: `http://<server-ip>:8081`

### 自定义新实例

创建 `.env` 文件，确保端口和 `INSTANCE` 名称不与已有实例冲突：

```bash
COMPOSE_PROJECT_NAME=signoz-c
INSTANCE=signoz-c
SIGNOZ_PORT=8082
OTLP_GRPC_PORT=4337
OTLP_HTTP_PORT=4338
JWT_SECRET=your-secret-here
VERSION=v0.128.0
OTELCOL_TAG=v0.144.5
CLICKHOUSE_TAG=25.5.6
ZOOKEEPER_TAG=3.7.1
```

## 常用运维命令

```bash
# 查看服务状态
docker compose ps

# 查看日志
docker compose logs -f signoz
docker compose logs -f otel-collector

# 停止服务
docker compose down

# 停止并清除所有数据（会丢失所有数据！）
docker compose down && rm -rf data/

# 更新镜像版本（修改 .env 中的版本变量后）
docker compose pull && docker compose up -d
```

## 向 SigNoz 发送数据

应用程序通过 OpenTelemetry SDK 将遥测数据发送到 OTel Collector：

```bash
# gRPC 端点
OTEL_EXPORTER_OTLP_ENDPOINT=http://<server-ip>:4317

# HTTP 端点
OTEL_EXPORTER_OTLP_ENDPOINT=http://<server-ip>:4318
```

替换为对应实例的端口即可。

## 数据持久化

数据通过 bind mount 存储在当前目录下的 `data/` 子目录中（首次启动时自动创建）：

| 宿主机路径 | 容器路径 | 用途 |
|------------|----------|------|
| `./data/clickhouse/` | `/var/lib/clickhouse/` | ClickHouse 时序数据 |
| `./data/signoz/` | `/var/lib/signoz/` | SigNoz 元数据（用户、仪表盘、告警等） |
| `./data/zookeeper/` | `/bitnami/zookeeper` | ZooKeeper 状态数据 |

备份只需直接复制 `data/` 目录即可。多实例场景下每个实例目录都有自己独立的 `data/`，互不干扰。

> **注意**: 停止并删除数据请使用 `rm -rf data/`，不再需要 `docker compose down -v`。
