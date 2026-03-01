# ELK Stack Architecture and Data Streams Documentation

## Table of Contents
1. [Overview](#overview)
2. [Data Streams Created](#data-streams-created)
3. [Architecture Overview](#architecture-overview)
4. [How the ELK Stack Works](#how-the-elk-stack-works)
5. [Data Stream Creation Process](#data-stream-creation-process)
6. [Log Processing Pipeline](#log-processing-pipeline)
7. [Security Architecture](#security-architecture)
8. [Configuration Files](#configuration-files)
9. [Querying in Kibana](#querying-in-kibana)
10. [Troubleshooting](#troubleshooting)

---

## Overview

This ELK (Elasticsearch, Logstash, Kibana) stack is configured to collect, process, and visualize logs from a  application. The stack uses **Elasticsearch Data Streams** (introduced in Elasticsearch 7.9+) for time-series log data, which provides better performance, automatic index management, and simplified lifecycle policies compared to traditional indices.

### Key Features
- **Data Streams** for different services in the application
- **Automatic log categorization** into streams
- **Secure authentication** with X-Pack security
- **Docker-based deployment** for easy setup and management

---

## Data Streams Created

### Summary

### Data Stream Structure

Each data stream has the following structure:
- **Type**: `logs` (fixed)
- **Dataset**: Service name (django, celery, celerybeat, flower, nginx, react)
- **Namespace**: `pt` (Parcel Tracking group)

**Example**: `logs-service.component-env` = `logs` (type) + `service.component` (dataset) + `env` (namespace)

---

## Architecture Overview

### Component Diagram

```
┌─────────────┐
│  App        │
│  Services   │
│             │
│  Django     │──┐
│  database   |  | Filebeat
└─────────────┘
      │
      │ (Beats protocol on port 5044)
      ▼
┌─────────────┐
│  Logstash   │
│  Port: 5044 │
│             │
│  - Parse    │
│  - Filter   │
│  - Enrich   │
└─────────────┘
      │
      │ (HTTP/REST API)
      ▼
┌─────────────┐
│Elasticsearch│
│  Port: 9200 │
│             │
│  Data       │
│  Streams    │
└─────────────┘
      │
      │ (HTTP/REST API)
      ▼
┌─────────────┐
│   Kibana    │
│  Port: 5601 │
│             │
│  - Search   │
│  - Visualize│
│  - Dashboard│
└─────────────┘
```

### Network Configuration

- **Docker Network**: `elk-net` (bridge network)
- **Service Discovery**: Services communicate using Docker service names (elasticsearch, kibana, logstash)

---

## How the ELK Stack Works

### End-to-End Flow

1. **Log Generation**
   - application services (Django, database, etc.) generate logs in format: `LEVEL|LOGGER_NAME|TIMESTAMP|MESSAGE`
   - Logs are written to files or stdout/stderr

2. **Log Collection (Filebeat)**
   - Filebeat (running on the application server) collects logs from configured sources
   - Filebeat adds metadata:
     - `service`: "project_name" (identifies the application group)
     - `component`: backend, database (identifies the service)
   - Filebeat sends logs to Logstash using the **Beats protocol** on port 5044

3. **Log Processing (Logstash)**
   - Logstash receives logs on port 5044 (Beats input)
   - **Filter Stage**:
     - Parses log format using Grok: `LEVEL|LOGGER_NAME|TIMESTAMP|MESSAGE`
     - Sets data stream metadata:
       - `[data_stream][type]`: "logs"
       - `[data_stream][dataset]`: "service.component"
       - `[data_stream][namespace]`: development server
     - Normalizes `@timestamp` from parsed log timestamp

4. **Data Storage (Elasticsearch)**
   - Logstash outputs to Elasticsearch using `data_stream => true`
   - Elasticsearch automatically creates data streams based on metadata:
     - Pattern: `{type}-{dataset}-{namespace}`
     - Example: `logs-recipegen.backend-dev`
   - Index template (`logs-service.component*`) defines field mappings and settings
   - Data is stored in backing indices (automatically managed by Elasticsearch)

5. **Visualization (Kibana)**
   - Create data views in Kibana:
     - `logs-service.component*`
     - Individual streams: `logs-recipegen.backend-dev`, `logs-recipegen.database-dev`, etc.
   - Query and visualize logs using Kibana Discover, Dashboards, and Visualizations

---

## Data Stream Creation Process

### Why Data Streams?

Data streams are the recommended way to store time-series data in Elasticsearch because:
- **Automatic index management**: Backing indices are created automatically
- **Simplified lifecycle**: Easier to manage retention and rollover
- **Better performance**: Optimized for append-only workloads
- **No manual index creation**: Elasticsearch handles index creation based on metadata

### How Data Streams Are Created

Data streams are created **automatically** when Logstash writes the first document. The process:

1. **Index Template Setup** (One-time, manual)
   ```bash
   curl -u elasticsearch_username:PASSWORD -X PUT "http://localhost:9200/_index_template/template_name" \
     -H "Content-Type: application/json" \
     -d @elasticsearch-index-template-pt-logs.json
   ```
   - Template matches pattern: `logs-service.component*`
   - Defines field mappings and index settings
   - Marks indices as data streams: `"data_stream": {}`

2. **Security Role Creation** (One-time, manual)
   ```bash
   # Create role with permissions for logs-*-pt pattern
   PUT _security/role/logstash_writer_service_name
   {
     "indices": [{
       "names": ["logs-service.component*"],
       "privileges": ["create_index", "create_doc", "index", "create", "write", "read"]
     }]
   }
   ```

3. **User Creation** (One-time, manual)
   ```bash
   # Create user with the role
   POST _security/user/logstash_writer
   {
     "password": "PASSWORD",
     "roles": ["logstash_writer_service_name"],
     "full_name": "Logstash writer for service_name data streams"
   }
   ```

4. **Automatic Stream Creation** (Automatic, on first write)
   - When Logstash writes a document with:
     - `[data_stream][type]`: "logs"
     - `[data_stream][namespace]`: "dev"
     - `[data_stream][dataset]`: "service.component" (or celery, etc.)
   - Elasticsearch automatically creates the data stream: `logs-service.component-dev`
   - First backing index is created automatically

### Data Stream Naming Convention

```
{type}-{dataset}-{namespace}
```

- **type**: Always `logs` (from Logstash configuration)
- **dataset**: Service name from Filebeat (recipegen) + Component name (backend, database)
- **namespace**: Group identifier (`dev` for development)

**Examples**:
- `logs-recipegen.backend-dev` = logs type, recipegen.backend dataset, dev namespace
---

## Log Processing Pipeline

### Input Stage

```ruby
input {
  beats {
    port => 5044
  }
}
```

- Listens on port 5044 for Beats protocol connections
- Receives logs from Filebeat with metadata (group, service)

#### 1. Data Stream Metadata
```ruby
mutate {
  replace => {
    "[data_stream][type]"      => "logs"
    "[data_stream][namespace]" => "{environment}"
    "[data_stream][dataset]"   => "%{[service].[component]}"
  }
}
```

### Output Stage

```ruby
output {
  if [service] == "recipegen" {
    elasticsearch {
            hosts => ["http://elasticsearch:9200"]
            user => "logstash_writer_recipeGen"
            password => "${LOGSTASH_WRITER_RECIPEGEN_PASSWORD}"
            ecs_compatibility => "v8"

            # rely on [data_stream] event metadata populated in the filter section
            data_stream => true
        }
  }
}
```

**Key Points**:
- `data_stream => true`: Tells Elasticsearch to use data streams instead of indices
- `ecs_compatibility => "v8"`: Uses ECS v8 field naming conventions

---

## Security Architecture

### Users and Roles

| User | Purpose | Permissions | Used By |
|------|---------|-------------|---------|
| `elastic` | Superuser | Full cluster access | Human admin, Kibana login |
| `kibana_system` | Kibana operations | `.kibana*` indices | Kibana service |
| `logstash_system` | Logstash monitoring | Monitoring indices | Logstash monitoring |
| `logstash_writer` | Log ingestion | `logs-*-pt` write/read | Logstash output |

### Security Flow

1. **Elasticsearch Security**
   - X-Pack security enabled: `xpack.security.enabled=true`
   - All requests require authentication
   - Passwords stored in Docker volumes (persist across restarts)

2. **Kibana → Elasticsearch**
   - Uses `kibana_system` user
   - Password from environment: `ELASTICSEARCH_PASSWORD`
   - Limited to `.kibana*` indices

3. **Logstash → Elasticsearch**
   - **Monitoring**: `logstash_system` user (for xpack.monitoring)
   - **Data Ingestion**: `logstash_writer` user (for writing logs)
   - Password from environment: `LOGSTASH_WRITER_PASSWORD`

4. **Human Access**
   - Login to Kibana UI as `elastic` user
   - Full admin access for management

### Role Permissions

**logstash_writer_pt Role**:
```json
{
  "indices": [{
    "names": ["logs-*-pt"],
    "privileges": [
      "create_index",  // Create backing indices
      "create_doc",    // Create documents
      "index",         // Index documents
      "create",        // Create operations
      "write",         // Write operations
      "read",           // Read operations
      "manage" 
    ]
  }]
}
```

**Why Least Privilege?**
- `logstash_writer` can only write to `logs-service.component*` pattern
- Cannot access other indices
- Cannot modify security settings
- Reduces blast radius if compromised

---

## Configuration Files

### 1. docker-compose.yml
- **Purpose**: Orchestrates Elasticsearch, Kibana, and Logstash services
- **Key Features**:
  - Health checks for all services
  - Service dependencies (Kibana/Logstash wait for Elasticsearch)
  - Volume persistence for data and configs
  - Network isolation (`elk-net`)
  - Environment variable injection for passwords

### 2. elasticsearch.yml
- **Purpose**: Elasticsearch node configuration
- **Key Settings**:
  - `cluster.name`: "elk-cluster"
  - `node.name`: "node-1"
  - `discovery.type`: single-node
  - `xpack.security.enabled`: true
  - `network.host`: 0.0.0.0 (allows container access)

### 3. kibana.yml
- **Purpose**: Kibana server configuration
- **Key Settings**:
  - `server.host`: "0.0.0.0" (allows Docker port mapping)
  - `elasticsearch.hosts`: ["http://elasticsearch:9200"]
  - `elasticsearch.username`: "kibana_system"
  - `xpack.reporting.kibanaServer.hostname`: localhost (for exports)

### 4. logstash.conf
- **Purpose**: Log processing pipeline
- **Sections**:
  - Input: Beats on port 5044
  - Filter: Parsing, categorization, enrichment
  - Output: Elasticsearch data streams

### 5. elasticsearch-index-template.json
- **Purpose**: Index template for logs
- **Key Features**:
  - Pattern: `logs-service.component*`
  - Data stream enabled: `"data_stream": {}`
  - Field mappings: @timestamp, message, group, service, stream, logger_name, log_level, log_message, app_module
  - Settings: 1 shard, 0 replicas (suitable for single-node)

### 6. elasticsearch-role-user-logstash-writer.json
- **Purpose**: Security role and user definition
- **Contains**: Role permissions and user creation instructions

---

## Querying in Kibana

### Data Views

### Useful Kibana Features

1. **Discover**: Search and filter logs
2. **Dashboards**: Create visualizations
3. **Visualize**: Build charts and graphs
4. **Dev Tools**: Run Elasticsearch queries directly

---

## Troubleshooting

### Common Issues

1. **No Data in Kibana**
   - Check if Filebeat is sending logs to Logstash
   - Verify Logstash is receiving on port 5044: `docker logs logstash`
   - Check Elasticsearch data streams: `GET _data_stream`
   - Verify `logstash_writer` user has correct permissions

2. **Connection Refused**
   - Ensure Elasticsearch is healthy: `docker compose ps`
   - Check `network.host: 0.0.0.0` in `elasticsearch.yml`
   - Verify services are on the same Docker network

3. **Permission Denied**
   - Verify `logstash_writer` role has `logs-service.component*` permissions
   - Check password in `.env` matches Elasticsearch user
   - Test authentication: `curl -u logstash_writer:PASSWORD http://localhost:9200`

4. **Data Stream Not Created**
   - Ensure index template is created: `GET _index_template/pt-logs`
   - Check Logstash output has `data_stream => true`
   - Verify data stream metadata is set correctly in Logstash filter

### Verification Commands

```bash
# Check Elasticsearch health
curl -u elastic:PASSWORD http://localhost:9200/_cluster/health

# List data streams
curl -u elastic:PASSWORD http://localhost:9200/_data_stream

# Check index template
curl -u elastic:PASSWORD http://localhost:9200/_index_template/pt-logs

# View Logstash logs
docker logs logstash --tail 100

# Check service status
docker compose ps

# Test Logstash connection to Elasticsearch
docker exec logstash curl -u logstash_writer:PASSWORD http://elasticsearch:9200
```

---

## Summary

This ELK stack implementation provides:

✅ **Data Streams** for comprehensive log collection from all services  
✅ **Automatic Categorization** into logical streams 
✅ **Application Module Classification** for detailed app log analysis  
✅ **Secure Authentication** with least-privilege access  
✅ **Docker-Based Deployment** for easy setup and management  
✅ **Production-Ready Configuration** with health checks, persistence, and restart policies  

The stack is designed to scale and can be extended with additional services, streams, or modules as needed.
