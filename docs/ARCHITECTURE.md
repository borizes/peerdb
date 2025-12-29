# PeerDB Services Architecture

This document provides a visual representation of the PeerDB services architecture based on the `docker-compose.yml` configuration.

## Architecture Overview

The PeerDB stack consists of multiple service groups working together:

- **Core Infrastructure**: Database and storage services
- **Workflow Engine**: Temporal-based orchestration
- **Flow Services**: Data flow processing workers
- **PeerDB Services**: Main application server and UI
- **Message Queue**: Redpanda for event streaming
- **Object Storage**: MinIO for S3-compatible storage

## Services Architecture Diagram

```mermaid
graph TB
    subgraph "Core Infrastructure"
        Catalog[("Catalog<br/>PostgreSQL:16<br/>Port: 9901")]
        MinIO[("MinIO<br/>S3 Storage<br/>Ports: 9001, 9002")]
    end

    subgraph "Temporal Workflow Engine"
        Temporal[Temporal Server<br/>Port: 7233]
        TemporalAdmin[Temporal Admin Tools]
        TemporalUI[Temporal UI<br/>Port: 8085]
    end

    subgraph "Flow Services"
        FlowAPI[Flow API<br/>Ports: 8112, 8113]
        FlowWorker[Flow Worker]
        FlowSnapshotWorker[Flow Snapshot Worker]
    end

    subgraph "PeerDB Services"
        PeerDB[PeerDB Server<br/>Port: 9900]
        PeerDBUI[PeerDB UI<br/>Port: 3000]
    end

    subgraph "Analytics & Messaging"
        ClickHouse[("ClickHouse<br/>Ports: 9000, 8123, 9009")]
        Redpanda[Redpanda<br/>Kafka API<br/>Ports: 18081, 18082, 19092, 19644]
        RedpandaConsole[Redpanda Console<br/>Port: 8080]
    end

    %% Core Infrastructure Dependencies
    Catalog -->|Database| Temporal
    Catalog -->|Database| FlowAPI
    Catalog -->|Database| FlowWorker
    Catalog -->|Database| FlowSnapshotWorker
    Catalog -->|Database| PeerDB
    Catalog -->|Database| PeerDBUI

    %% Temporal Dependencies
    Temporal -->|Depends on| Catalog
    TemporalAdmin -->|Depends on| Temporal
    TemporalUI -->|Depends on| Temporal

    %% Flow Services Dependencies
    FlowAPI -->|Depends on| TemporalAdmin
    FlowAPI -->|Uses| Temporal
    FlowAPI -->|Uses| Catalog
    FlowAPI -->|Uses| MinIO
    FlowWorker -->|Depends on| TemporalAdmin
    FlowWorker -->|Uses| Temporal
    FlowWorker -->|Uses| Catalog
    FlowWorker -->|Uses| MinIO
    FlowSnapshotWorker -->|Depends on| TemporalAdmin
    FlowSnapshotWorker -->|Uses| Temporal
    FlowSnapshotWorker -->|Uses| Catalog
    FlowSnapshotWorker -->|Uses| MinIO

    %% PeerDB Services Dependencies
    PeerDB -->|Depends on| Catalog
    PeerDB -->|Uses| FlowAPI
    PeerDBUI -->|Depends on| FlowAPI
    PeerDBUI -->|Uses| Catalog

    %% Storage Dependencies
    ClickHouse -->|Depends on| MinIO
    ClickHouse -->|Uses S3| MinIO

    %% Messaging Dependencies
    RedpandaConsole -->|Depends on| Redpanda

    %% Styling
    classDef database fill:#e1f5ff,stroke:#01579b,stroke-width:2px
    classDef storage fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef service fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef ui fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px
    classDef worker fill:#fce4ec,stroke:#880e4f,stroke-width:2px

    class Catalog,ClickHouse database
    class MinIO storage
    class Temporal,FlowAPI,PeerDB,Redpanda service
    class TemporalUI,PeerDBUI,RedpandaConsole ui
    class FlowWorker,FlowSnapshotWorker,TemporalAdmin worker
```

## Service Details

### Core Infrastructure Services

#### Catalog (PostgreSQL)
- **Container**: `catalog`
- **Image**: `postgres:16-alpine`
- **Port**: `9901:5432`
- **Purpose**: Primary database for catalog metadata and Temporal persistence
- **Health Check**: PostgreSQL readiness check

#### MinIO
- **Container**: `minio`
- **Image**: `minio/minio:RELEASE.2024-07-16T23-46-41Z`
- **Ports**: `9001:9000` (API), `9002:36987` (Console)
- **Purpose**: S3-compatible object storage for data staging and ClickHouse integration
- **Bucket**: `peerdbbucket`

### Temporal Workflow Engine

#### Temporal Server
- **Container**: `temporal`
- **Image**: `temporalio/auto-setup:1.25`
- **Port**: `7233:7233`
- **Purpose**: Workflow orchestration engine
- **Database**: Uses Catalog (PostgreSQL) for persistence

#### Temporal Admin Tools
- **Container**: `temporal-admin-tools`
- **Image**: `temporalio/admin-tools:1.25`
- **Purpose**: Administrative tools for Temporal workflows
- **Health Check**: Temporal workflow list command

#### Temporal UI
- **Container**: `temporal-ui`
- **Image**: `temporalio/ui:2.29.1`
- **Port**: `8085:8080`
- **Purpose**: Web UI for monitoring Temporal workflows

### Flow Services

#### Flow API
- **Container**: `flow_api`
- **Image**: `ghcr.io/peerdb-io/flow-api:latest-dev`
- **Ports**: `8112:8112` (gRPC), `8113:8113` (HTTP)
- **Purpose**: API service for data flow operations
- **Dependencies**: Catalog, Temporal, MinIO

#### Flow Worker
- **Container**: `flow-worker`
- **Image**: `ghcr.io/peerdb-io/flow-worker:latest-dev`
- **Purpose**: Background worker for processing data flows
- **Dependencies**: Catalog, Temporal, MinIO

#### Flow Snapshot Worker
- **Container**: `flow-snapshot-worker`
- **Image**: `ghcr.io/peerdb-io/flow-snapshot-worker:latest-dev`
- **Purpose**: Specialized worker for snapshot operations
- **Dependencies**: Catalog, Temporal, MinIO

### PeerDB Services

#### PeerDB Server
- **Container**: `peerdb-server`
- **Image**: `ghcr.io/peerdb-io/peerdb-server:latest-dev`
- **Port**: `9900:9900`
- **Purpose**: Main PeerDB application server
- **Dependencies**: Catalog, Flow API

#### PeerDB UI
- **Container**: `peerdb-ui`
- **Image**: `ghcr.io/peerdb-io/peerdb-ui:latest-dev`
- **Port**: `3000:3000`
- **Purpose**: Web-based user interface for PeerDB
- **Dependencies**: Catalog, Flow API

### Analytics & Messaging

#### ClickHouse
- **Container**: `clickhouse`
- **Image**: `clickhouse/clickhouse-server:latest`
- **Ports**: `9000:9000` (Native), `8123:8123` (HTTP), `9009:9009` (Inter-server)
- **Purpose**: Columnar database for analytics
- **Storage**: Uses MinIO for S3 integration
- **Health Check**: ClickHouse client query

#### Redpanda
- **Container**: `redpanda`
- **Image**: `ghcr.io/peerdb-io/redpanda:latest-dev`
- **Ports**: 
  - `18081:18081` (Schema Registry)
  - `18082:18082` (Pandaproxy)
  - `19092:19092` (Kafka API)
  - `19644:9644` (Admin API)
- **Purpose**: Kafka-compatible message broker
- **Health Check**: Cluster health check

#### Redpanda Console
- **Container**: `redpanda-console`
- **Image**: `docker.redpanda.com/redpandadata/console:latest`
- **Port**: `8080:8080`
- **Purpose**: Web UI for Redpanda/Kafka management
- **Dependencies**: Redpanda

## Service Dependencies

### Dependency Chain

1. **Foundation Layer**
   - `catalog` (PostgreSQL) - No dependencies
   - `minio` - No dependencies

2. **Workflow Layer**
   - `temporal` → depends on `catalog`
   - `temporal-admin-tools` → depends on `temporal`
   - `temporal-ui` → depends on `temporal`

3. **Flow Layer**
   - `flow-api` → depends on `temporal-admin-tools` (health check)
   - `flow-worker` → depends on `temporal-admin-tools` (health check)
   - `flow-snapshot-worker` → depends on `temporal-admin-tools` (health check)

4. **Application Layer**
   - `peerdb` → depends on `catalog` (health check)
   - `peerdb-ui` → depends on `flow-api`

5. **Storage & Messaging Layer**
   - `clickhouse` → depends on `minio`
   - `redpanda` - No dependencies
   - `redpanda-console` → depends on `redpanda`

## Network Configuration

- **Network Name**: `peerdb_network`
- **Network Type**: Default bridge network
- **Extra Hosts**: `host.docker.internal:host-gateway` (for accessing host services)

## Volumes

- `pgdata` - PostgreSQL data persistence
- `minio-data` - MinIO object storage
- `clickhouse-data` - ClickHouse data directory
- `clickhouse-logs` - ClickHouse log files
- `redpanda-data` - Redpanda data persistence

## Port Mapping Summary

| Service | Internal Port | External Port | Protocol |
|---------|--------------|---------------|----------|
| Catalog | 5432 | 9901 | PostgreSQL |
| Temporal | 7233 | 7233 | gRPC |
| Temporal UI | 8080 | 8085 | HTTP |
| Flow API | 8112, 8113 | 8112, 8113 | gRPC, HTTP |
| PeerDB Server | 9900 | 9900 | gRPC/HTTP |
| PeerDB UI | 3000 | 3000 | HTTP |
| MinIO API | 9000 | 9001 | S3 API |
| MinIO Console | 36987 | 9002 | HTTP |
| ClickHouse Native | 9000 | 9000 | Native |
| ClickHouse HTTP | 8123 | 8123 | HTTP |
| ClickHouse Inter-server | 9009 | 9009 | Native |
| Redpanda Schema Registry | 8081 | 18081 | HTTP |
| Redpanda Pandaproxy | 8082 | 18082 | HTTP |
| Redpanda Kafka API | 9092 | 19092 | Kafka |
| Redpanda Admin API | 9644 | 19644 | HTTP |
| Redpanda Console | 8080 | 8080 | HTTP |

## Health Checks

All services implement health checks to ensure proper startup ordering:

- **Catalog**: PostgreSQL readiness (`pg_isready`)
- **Temporal Admin Tools**: Temporal workflow list command
- **ClickHouse**: ClickHouse client query (`SELECT 1`)
- **Redpanda**: Cluster health check (`rpk cluster health`)
- **Redpanda Console**: HTTP health endpoint (`/api/health`)

## Environment Configuration

The architecture uses YAML anchors for shared configuration:

- `*catalog-config`: Database connection settings
- `*minio-config`: S3/MinIO credentials and endpoint
- `*flow-worker-env`: Temporal and AWS configuration for flow workers

