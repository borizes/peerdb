# PeerDB Stack Resource Requirements & Dimensions

This document provides comprehensive resource planning and dimensioning guidelines for the PeerDB stack, including CPU, memory, storage, and network requirements for different deployment scenarios.

## Executive Summary

The PeerDB stack consists of 13 services organized into functional groups. Resource requirements vary significantly between development and production environments. This document provides detailed specifications for capacity planning and infrastructure provisioning.

**Total Stack Resource Estimates:**
- **Development**: ~8-12 CPU cores, ~16-24 GB RAM, ~100-500 GB storage
- **Production (Small)**: ~16-24 CPU cores, ~32-48 GB RAM, ~500 GB - 2 TB storage
- **Production (Medium)**: ~32-48 CPU cores, ~64-96 GB RAM, ~2-10 TB storage
- **Production (Large)**: ~64+ CPU cores, ~128+ GB RAM, ~10+ TB storage

## Resource Allocation by Service

### Core Infrastructure Services

#### Catalog (PostgreSQL)

**Purpose**: Primary metadata database and Temporal persistence layer

| Environment | CPU | Memory | Storage | Notes |
|------------|-----|--------|---------|-------|
| Development | 0.5-1 | 512 MB - 1 GB | 10-50 GB | Minimal workload |
| Production (Small) | 2-4 | 4-8 GB | 100-500 GB | < 1000 mirrors |
| Production (Medium) | 4-8 | 8-16 GB | 500 GB - 2 TB | 1000-10000 mirrors |
| Production (Large) | 8-16 | 16-32 GB | 2-10 TB | > 10000 mirrors |

**Storage Considerations:**
- PostgreSQL data growth depends on:
  - Number of mirrors (CDC flows)
  - Workflow history retention
  - Catalog metadata volume
- Recommended: SSD storage with IOPS > 3000
- Backup storage: 2-3x primary storage for retention

**Configuration Recommendations:**
```yaml
# Production PostgreSQL tuning
shared_buffers: 25% of RAM
effective_cache_size: 50-75% of RAM
maintenance_work_mem: 1-2 GB
work_mem: 16-64 MB per connection
max_connections: 100-500 (adjust based on load)
```

#### MinIO (S3 Storage)

**Purpose**: Object storage for data staging and ClickHouse integration

| Environment | CPU | Memory | Storage | Notes |
|------------|-----|--------|---------|-------|
| Development | 0.5-1 | 512 MB - 1 GB | 50-200 GB | Local development |
| Production (Small) | 2-4 | 4-8 GB | 500 GB - 2 TB | Single node |
| Production (Medium) | 4-8 | 8-16 GB | 2-10 TB | Distributed (4+ nodes) |
| Production (Large) | 8-16 | 16-32 GB | 10+ TB | Distributed (8+ nodes) |

**Storage Considerations:**
- Storage scales with:
  - Snapshot data volume
  - CDC staging data
  - ClickHouse S3 integration data
- For production: Use distributed MinIO with erasure coding
- Network bandwidth critical for data transfer
- Recommended: High IOPS storage (NVMe SSD preferred)

**Scaling Strategy:**
- Horizontal scaling: Add more MinIO nodes
- Erasure coding: 4+2 or 8+2 for redundancy
- Consider cloud S3 (AWS S3, GCS) for production

### Temporal Workflow Engine

#### Temporal Server

**Purpose**: Workflow orchestration and state management

| Environment | CPU | Memory | Storage | Notes |
|------------|-----|--------|---------|-------|
| Development | 1-2 | 2-4 GB | N/A (uses Catalog) | Single instance |
| Production (Small) | 2-4 | 4-8 GB | N/A (uses Catalog) | Single instance |
| Production (Medium) | 4-8 | 8-16 GB | N/A (uses Catalog) | High availability |
| Production (Large) | 8-16 | 16-32 GB | N/A (uses Catalog) | Multi-region |

**Resource Notes:**
- Temporal uses PostgreSQL for persistence (Catalog service)
- Memory usage scales with:
  - Number of active workflows
  - Workflow history size
  - Concurrent activity executions
- CPU usage scales with workflow execution rate

**Configuration Recommendations:**
- For production: Deploy Temporal in high-availability mode
- Consider Temporal Cloud for managed service
- Monitor workflow queue depth and execution latency

#### Temporal Admin Tools

**Purpose**: Administrative CLI tools for Temporal

| Environment | CPU | Memory | Storage | Notes |
|------------|-----|--------|---------|-------|
| All | 0.25-0.5 | 256-512 MB | Minimal | Lightweight utility |

#### Temporal UI

**Purpose**: Web interface for workflow monitoring

| Environment | CPU | Memory | Storage | Notes |
|------------|-----|--------|---------|-------|
| Development | 0.25-0.5 | 256-512 MB | Minimal | Single user |
| Production | 0.5-1 | 512 MB - 1 GB | Minimal | Multiple concurrent users |

### Flow Services

#### Flow API

**Purpose**: gRPC and HTTP API for data flow operations

| Environment | CPU | Memory | Storage | Notes |
|------------|-----|--------|---------|-------|
| Development | 1-2 | 1-2 GB | Minimal | Low traffic |
| Production (Small) | 2-4 | 2-4 GB | Minimal | < 100 concurrent requests |
| Production (Medium) | 4-8 | 4-8 GB | Minimal | 100-1000 concurrent requests |
| Production (Large) | 8-16 | 8-16 GB | Minimal | > 1000 concurrent requests |

**Scaling Strategy:**
- Horizontal scaling: Multiple Flow API instances behind load balancer
- Stateless service - easy to scale horizontally
- Consider auto-scaling based on request rate

#### Flow Worker

**Purpose**: Background worker for processing CDC flows

| Environment | CPU | Memory | Storage | Notes |
|------------|-----|--------|---------|-------|
| Development | 1-2 | 2-4 GB | Minimal | Single worker |
| Production (Small) | 2-4 | 4-8 GB | Minimal | 2-4 workers |
| Production (Medium) | 4-8 | 8-16 GB | Minimal | 4-8 workers |
| Production (Large) | 8-16 | 16-32 GB | Minimal | 8-16+ workers |

**Resource Notes:**
- CPU-intensive: Data transformation and processing
- Memory scales with:
  - Batch size
  - Concurrent flow processing
  - Data buffering requirements
- Horizontal scaling: Add more worker instances

**Scaling Strategy:**
- Scale workers based on:
  - Number of active mirrors
  - Data throughput per mirror
  - Target latency requirements
- Consider worker pools for different mirror types

#### Flow Snapshot Worker

**Purpose**: Specialized worker for snapshot operations

| Environment | CPU | Memory | Storage | Notes |
|------------|-----|--------|---------|-------|
| Development | 1-2 | 2-4 GB | Minimal | Single worker |
| Production (Small) | 2-4 | 4-8 GB | Minimal | 1-2 workers |
| Production (Medium) | 4-8 | 8-16 GB | Minimal | 2-4 workers |
| Production (Large) | 8-16 | 16-32 GB | Minimal | 4-8 workers |

**Resource Notes:**
- Snapshot operations are memory-intensive
- CPU usage spikes during initial snapshot creation
- Consider dedicated workers for large snapshots

### PeerDB Services

#### PeerDB Server

**Purpose**: Main application server (Rust-based)

| Environment | CPU | Memory | Storage | Notes |
|------------|-----|--------|---------|-------|
| Development | 1-2 | 1-2 GB | Minimal | Single instance |
| Production (Small) | 2-4 | 2-4 GB | Minimal | Single instance |
| Production (Medium) | 4-8 | 4-8 GB | Minimal | High availability (2+ instances) |
| Production (Large) | 8-16 | 8-16 GB | Minimal | High availability (3+ instances) |

**Resource Notes:**
- Rust service - efficient memory usage
- CPU scales with:
  - Query complexity
  - Concurrent connections
  - Mirror management operations
- Stateless service - can scale horizontally

**Scaling Strategy:**
- Horizontal scaling behind load balancer
- Consider connection pooling
- Monitor query performance and optimize

#### PeerDB UI

**Purpose**: Next.js web application

| Environment | CPU | Memory | Storage | Notes |
|------------|-----|--------|---------|-------|
| Development | 0.5-1 | 512 MB - 1 GB | Minimal | Single instance |
| Production (Small) | 1-2 | 1-2 GB | Minimal | Single instance |
| Production (Medium) | 2-4 | 2-4 GB | Minimal | 2+ instances (load balanced) |
| Production (Large) | 4-8 | 4-8 GB | Minimal | 3+ instances (load balanced) |

**Resource Notes:**
- Next.js application - server-side rendering
- Memory scales with concurrent users
- Consider CDN for static assets in production

### Analytics & Messaging

#### ClickHouse

**Purpose**: Columnar database for analytics and data warehousing

| Environment | CPU | Memory | Storage | Notes |
|------------|-----|--------|---------|-------|
| Development | 2-4 | 4-8 GB | 50-200 GB | Single node |
| Production (Small) | 4-8 | 16-32 GB | 500 GB - 2 TB | Single node |
| Production (Medium) | 8-16 | 32-64 GB | 2-10 TB | Cluster (3+ nodes) |
| Production (Large) | 16-32 | 64-128 GB | 10+ TB | Cluster (6+ nodes) |

**Resource Notes:**
- Memory-intensive: Uses RAM for query processing
- Storage scales with:
  - Data retention period
  - Number of tables/columns
  - Compression ratio (typically 5-10x)
- CPU scales with query complexity and concurrency
- **Ulimit Configuration**: `nofile: 262144` (already configured)

**Storage Considerations:**
- ClickHouse data directory: `/var/lib/clickhouse`
- ClickHouse logs: `/var/log/clickhouse-server`
- S3 integration for cold storage (MinIO)
- Recommended: NVMe SSD for hot data, S3 for cold data

**Configuration Recommendations:**
```xml
<!-- ClickHouse memory settings -->
<max_memory_usage>0</max_memory_usage> <!-- 0 = unlimited, set based on available RAM -->
<max_server_memory_usage_to_ram_ratio>0.9</max_server_memory_usage_to_ram_ratio>
```

#### Redpanda

**Purpose**: Kafka-compatible message broker for event streaming

| Environment | CPU | Memory | Storage | Notes |
|------------|-----|--------|---------|-------|
| Development | 1 | 1 GB | 10-50 GB | Single node (current config) |
| Production (Small) | 2-4 | 4-8 GB | 100-500 GB | Single node |
| Production (Medium) | 4-8 | 8-16 GB | 500 GB - 2 TB | Cluster (3 nodes) |
| Production (Large) | 8-16 | 16-32 GB | 2-10 TB | Cluster (5+ nodes) |

**Current Configuration:**
- CPU: `--smp 1` (1 CPU core)
- Memory: `--memory 1G` (1 GB RAM)
- Mode: `dev-container`

**Production Recommendations:**
- CPU: 2-4 cores minimum, scale based on throughput
- Memory: 4-8 GB minimum, scales with:
  - Topic retention
  - Number of partitions
  - Consumer lag
- Storage: Scales with message retention policy
- Network: High bandwidth for message replication

**Scaling Strategy:**
- Horizontal scaling: Add more Redpanda nodes
- Replication factor: 3 for production
- Partition count: Scale based on consumer parallelism

#### Redpanda Console

**Purpose**: Web UI for Redpanda/Kafka management

| Environment | CPU | Memory | Storage | Notes |
|------------|-----|--------|---------|-------|
| All | 0.25-0.5 | 256-512 MB | Minimal | Lightweight UI |

## Aggregate Resource Requirements

### Development Environment

**Total Resources:**
- **CPU**: 8-12 cores
- **Memory**: 16-24 GB
- **Storage**: 100-500 GB
- **Network**: 1 Gbps

**Service Distribution:**
```
Catalog:           1 core,  1 GB RAM
Temporal:          2 cores, 4 GB RAM
Temporal UI:       0.5 core, 512 MB RAM
Flow API:          1 core,  2 GB RAM
Flow Worker:       1 core,  2 GB RAM
Flow Snapshot:     1 core,  2 GB RAM
PeerDB Server:    1 core,  2 GB RAM
PeerDB UI:         0.5 core, 1 GB RAM
MinIO:             1 core,  1 GB RAM
ClickHouse:        2 cores, 4 GB RAM
Redpanda:          1 core,  1 GB RAM (configured)
Redpanda Console:   0.25 core, 256 MB RAM
```

### Production - Small Scale

**Total Resources:**
- **CPU**: 16-24 cores
- **Memory**: 32-48 GB
- **Storage**: 500 GB - 2 TB
- **Network**: 10 Gbps

**Service Distribution:**
```
Catalog:           4 cores,  8 GB RAM
Temporal:          4 cores,  8 GB RAM
Temporal UI:       1 core,   1 GB RAM
Flow API:          4 cores,  4 GB RAM (2 instances)
Flow Worker:       4 cores,  8 GB RAM (2 instances)
Flow Snapshot:      2 cores,  4 GB RAM (1 instance)
PeerDB Server:     4 cores,  4 GB RAM (2 instances)
PeerDB UI:         2 cores,  2 GB RAM (2 instances)
MinIO:             4 cores,  8 GB RAM
ClickHouse:        8 cores,  32 GB RAM
Redpanda:          4 cores,  8 GB RAM
Redpanda Console:  0.5 core, 512 MB RAM
```

### Production - Medium Scale

**Total Resources:**
- **CPU**: 32-48 cores
- **Memory**: 64-96 GB
- **Storage**: 2-10 TB
- **Network**: 25 Gbps

**Service Distribution:**
```
Catalog:           8 cores,  16 GB RAM
Temporal:          8 cores,  16 GB RAM (HA)
Temporal UI:       1 core,   1 GB RAM
Flow API:          8 cores,  8 GB RAM (4 instances)
Flow Worker:       8 cores,  16 GB RAM (4 instances)
Flow Snapshot:      4 cores,  8 GB RAM (2 instances)
PeerDB Server:     8 cores,  8 GB RAM (3 instances)
PeerDB UI:         4 cores,  4 GB RAM (3 instances)
MinIO:             8 cores,  16 GB RAM (distributed)
ClickHouse:        16 cores, 64 GB RAM (cluster)
Redpanda:          8 cores,  16 GB RAM (cluster)
Redpanda Console:  0.5 core, 512 MB RAM
```

### Production - Large Scale

**Total Resources:**
- **CPU**: 64+ cores
- **Memory**: 128+ GB
- **Storage**: 10+ TB
- **Network**: 100 Gbps

**Service Distribution:**
```
Catalog:           16 cores, 32 GB RAM (HA)
Temporal:          16 cores, 32 GB RAM (multi-region)
Temporal UI:       1 core,   1 GB RAM
Flow API:          16 cores, 16 GB RAM (8+ instances)
Flow Worker:       16 cores, 32 GB RAM (8+ instances)
Flow Snapshot:      8 cores,  16 GB RAM (4+ instances)
PeerDB Server:     16 cores, 16 GB RAM (5+ instances)
PeerDB UI:         8 cores,  8 GB RAM (5+ instances)
MinIO:             16 cores, 32 GB RAM (distributed)
ClickHouse:        32 cores, 128 GB RAM (large cluster)
Redpanda:          16 cores, 32 GB RAM (large cluster)
Redpanda Console:  1 core,   1 GB RAM
```

## Storage Requirements

### Storage Growth Factors

1. **Catalog (PostgreSQL)**
   - Base: ~1-5 GB
   - Per mirror: ~10-100 MB metadata
   - Workflow history: ~1-10 GB per 1000 workflows
   - Growth rate: 1-5 GB/month (small), 10-50 GB/month (medium), 50+ GB/month (large)

2. **MinIO (Object Storage)**
   - Snapshot data: Varies by source database size
   - CDC staging: ~10-50% of source database size
   - Retention: Configurable (typically 7-30 days)
   - Growth rate: Highly variable based on data volume

3. **ClickHouse**
   - Data compression: 5-10x typical
   - Retention: Configurable (typically 30-365 days)
   - Growth rate: Depends on query patterns and retention

4. **Redpanda**
   - Message retention: Configurable (typically 7-30 days)
   - Growth rate: Depends on message volume and retention

### Storage Recommendations

| Component | Storage Type | IOPS Requirement | Notes |
|-----------|-------------|------------------|-------|
| Catalog | SSD | > 3000 IOPS | Database workloads |
| MinIO | SSD/NVMe | > 5000 IOPS | Object storage |
| ClickHouse (hot) | NVMe SSD | > 10000 IOPS | Query performance |
| ClickHouse (cold) | S3/Object | N/A | Archive storage |
| Redpanda | SSD | > 5000 IOPS | Message log |

## Network Requirements

### Bandwidth Considerations

1. **Inter-service Communication**
   - gRPC: Low latency, moderate bandwidth
   - HTTP: Standard web traffic
   - Database connections: Moderate bandwidth

2. **Data Transfer**
   - MinIO S3 operations: High bandwidth
   - ClickHouse queries: Moderate to high bandwidth
   - Redpanda replication: High bandwidth (production clusters)

3. **External Connectivity**
   - Source database connections: Varies by source
   - Target database connections: Varies by target
   - User access (UI): Low to moderate bandwidth

### Network Recommendations

| Environment | Bandwidth | Latency | Notes |
|------------|-----------|---------|-------|
| Development | 1 Gbps | < 10ms | Local network |
| Production (Small) | 10 Gbps | < 5ms | Data center |
| Production (Medium) | 25 Gbps | < 2ms | High-speed network |
| Production (Large) | 100 Gbps | < 1ms | Ultra-low latency |

## Resource Monitoring & Optimization

### Key Metrics to Monitor

1. **CPU Utilization**
   - Target: 60-80% average, < 95% peak
   - Alert: > 90% sustained

2. **Memory Utilization**
   - Target: 70-85% average, < 95% peak
   - Alert: > 90% sustained
   - Watch for OOM (Out of Memory) events

3. **Storage Utilization**
   - Target: < 80% capacity
   - Alert: > 85% capacity
   - Plan expansion at 70%

4. **Network Utilization**
   - Target: < 70% of available bandwidth
   - Alert: > 85% sustained

5. **Service-Specific Metrics**
   - **PostgreSQL**: Connection count, query latency, replication lag
   - **Temporal**: Workflow queue depth, execution latency
   - **ClickHouse**: Query latency, memory usage, disk I/O
   - **Redpanda**: Consumer lag, partition count, replication lag

### Optimization Strategies

1. **Right-sizing**
   - Start with development estimates
   - Monitor for 1-2 weeks
   - Adjust based on actual usage patterns

2. **Horizontal Scaling**
   - Scale stateless services (Flow API, PeerDB Server, PeerDB UI)
   - Add worker instances for increased throughput
   - Use load balancers for distribution

3. **Vertical Scaling**
   - Increase resources for stateful services (Catalog, ClickHouse)
   - Upgrade storage for better IOPS
   - Increase memory for memory-intensive services

4. **Storage Optimization**
   - Implement data retention policies
   - Use compression (ClickHouse, PostgreSQL)
   - Archive old data to cold storage (S3)

5. **Network Optimization**
   - Use dedicated networks for inter-service communication
   - Implement connection pooling
   - Consider service mesh for advanced networking

## High Availability & Disaster Recovery

### HA Requirements

1. **Database (Catalog)**
   - Primary-replica setup
   - Automated failover
   - Synchronous replication for critical data

2. **Temporal**
   - Multi-instance deployment
   - Shared database backend
   - Load balanced access

3. **Stateless Services**
   - Multiple instances behind load balancer
   - Health checks and auto-recovery
   - Graceful shutdown handling

4. **Stateful Services (ClickHouse, Redpanda)**
   - Cluster deployment with replication
   - Distributed storage
   - Automated recovery

### Disaster Recovery

1. **Backup Strategy**
   - Catalog: Daily backups with point-in-time recovery
   - MinIO: Object versioning and replication
   - ClickHouse: Regular snapshots and S3 backups
   - Configuration: Version-controlled infrastructure as code

2. **Recovery Time Objectives (RTO)**
   - Critical services: < 15 minutes
   - Non-critical services: < 1 hour
   - Full stack recovery: < 4 hours

3. **Recovery Point Objectives (RPO)**
   - Critical data: < 1 hour
   - Non-critical data: < 24 hours

## Cost Estimation (Cloud Providers)

### AWS Example (Production Medium Scale)

| Service | Instance Type | Monthly Cost (approx) |
|---------|---------------|----------------------|
| Catalog (RDS) | db.r5.2xlarge | $500-800 |
| Temporal | m5.2xlarge (2x) | $600-800 |
| Flow Services | m5.xlarge (6x) | $900-1200 |
| PeerDB Services | m5.xlarge (6x) | $900-1200 |
| MinIO (S3) | S3 Standard | $200-500 (storage) |
| ClickHouse | r5.4xlarge (3x) | $1500-2000 |
| Redpanda | m5.2xlarge (3x) | $900-1200 |
| **Total** | | **$5500-7700/month** |

*Note: Costs vary by region, usage, and reserved instances. Storage costs additional.*

### Cost Optimization

1. **Reserved Instances**: 30-50% savings for predictable workloads
2. **Spot Instances**: 70-90% savings for non-critical workloads
3. **Storage Optimization**: Use appropriate storage tiers
4. **Auto-scaling**: Scale down during low-usage periods
5. **Managed Services**: Consider managed databases (RDS, Temporal Cloud)

## Deployment Recommendations

### Development
- Single-node deployment acceptable
- Resource over-provisioning acceptable for simplicity
- Focus on ease of development and debugging

### Staging
- Mirror production architecture
- Use smaller instance sizes
- Test scaling and failover scenarios

### Production
- High availability mandatory
- Monitoring and alerting required
- Automated scaling policies
- Regular capacity planning reviews
- Disaster recovery testing

## Conclusion

Resource requirements for the PeerDB stack scale significantly based on:
- Number of active mirrors (CDC flows)
- Data volume and throughput
- Query complexity and concurrency
- Availability requirements

Regular monitoring and capacity planning are essential for optimal performance and cost management. Start with development estimates, monitor actual usage, and adjust based on real-world patterns.

---

**Document Version**: 1.0  
**Last Updated**: 2024  
**Maintained By**: PeerDB Infrastructure Team

