# Podman Pod Migration Guide - fx-pod

This guide covers exporting containers from your development environment and importing them to production using Podman's pod architecture.

## Understanding Your Current Setup

Your production environment uses a **Podman Pod** named `fx-pod`. A pod is a group of containers that share:
- Network namespace (all containers can communicate via localhost)
- Can share storage volumes
- Started/stopped as a single unit

---

## Table of Contents

1. [Pre-Migration Checklist](#pre-migration-checklist)
2. [Export Strategy](#export-strategy)
3. [Export Process (Development Environment)](#export-process-development-environment)
4. [Transfer to Production](#transfer-to-production)
5. [Import Process (Production Environment)](#import-process-production-environment)
6. [Database Manual Steps](#database-manual-steps)
7. [Pod Creation and Startup](#pod-creation-and-startup)
8. [Verification](#verification)
9. [Troubleshooting](#troubleshooting)

---

## Pre-Migration Checklist

Before starting migration:

- [ ] Verify all containers are working in development
- [ ] Document current pod configuration: `podman pod inspect fx-pod`
- [ ] Note all volume mounts and configurations
- [ ] Backup production database (if existing)
- [ ] Ensure sufficient disk space on production server (at least 10GB free)
- [ ] Have access to both development and production servers

---

## Export Strategy

We'll use **two approaches** to ensure a smooth migration:

### Approach 1: Export Container Images (Recommended)
Export the built images and recreate pod on production

**Pros:**
- Clean and reliable
- Smaller transfer size
- Easy to version control

**Cons:**
- Need to recreate pod manually
- Database data not included

### Approach 2: Export Entire Pod with Containers
Export running containers with their state

**Pros:**
- Captures exact state

**Cons:**
- Larger file sizes
- May include temporary data
- Database data still needs separate handling

**We'll use Approach 1 with manual database migration.**

---

## Export Process (Development Environment)

### Step 1: Document Current Pod Configuration

```bash
# On development machine
cd /path/to/fx-transaction-processor

# Get pod details
podman pod inspect fx-pod > fx-pod-config.json

# List containers in pod
podman ps -a --pod --filter pod=fx-pod

# Save port mappings
podman pod inspect fx-pod | grep -A 20 "portmappings" > fx-pod-ports.txt
```

### Step 2: Export Container Images

```bash
# Create export directory
mkdir -p ~/fx-export
cd ~/fx-export

# Export backend image
podman save -o fx-backend.tar localhost/fx-transaction-processor-backend:latest

# Export frontend image
podman save -o fx-frontend.tar localhost/fx-transaction-processor-frontend:latest

# Export test-data-emitter image (if built locally)
podman save -o fx-test-emitter.tar localhost/fx-transaction-processor-test-data-emitter:latest

# Export Oracle XE (if using custom image)
# If using official Oracle image, skip this - will pull on production
# podman save -o oracle-xe.tar container-registry.oracle.com/database/express:21.3.0-xe

# Export Kafka and Zookeeper (if using custom images)
# Usually these are public images, so skip and pull on production
```

**File sizes (approximate):**
- fx-backend.tar: ~300-500 MB
- fx-frontend.tar: ~50-100 MB
- fx-test-emitter.tar: ~200-300 MB

### Step 3: Export Database Data

```bash
# Export Oracle database schema and data
podman exec fx-oracle-xe bash -c "expdp system/OraclePassword123@XEPDB1 \
  schemas=SYSTEM \
  directory=DATA_PUMP_DIR \
  dumpfile=fx_export.dmp \
  logfile=fx_export.log"

# Copy export file from container
podman cp fx-oracle-xe:/opt/oracle/admin/XE/dpdump/fx_export.dmp ./fx-database-export.dmp
podman cp fx-oracle-xe:/opt/oracle/admin/XE/dpdump/fx_export.log ./fx-database-export.log
```

**Alternative: SQL Export (smaller, more portable)**
```bash
# Export just the application tables
podman exec fx-oracle-xe bash -c "sqlplus -s system/OraclePassword123@XEPDB1" <<EOF > fx-schema.sql
SET PAGESIZE 0
SET LONG 90000
SET FEEDBACK OFF
SET ECHO OFF

-- Export topic_configurations
SELECT 'DELETE FROM topic_configurations;' FROM dual;

SELECT 'INSERT INTO topic_configurations (id, topic_name, description, bootstrap_servers, group_id, ssl_enabled, security_protocol, auto_offset_reset, max_poll_records, session_timeout_ms, status, created_at, updated_at) VALUES (' ||
       id || ', ''' || topic_name || ''', ''' || description || ''', ''' ||
       bootstrap_servers || ''', ''' || group_id || ''', ' || ssl_enabled || ', ''' ||
       security_protocol || ''', ''' || auto_offset_reset || ''', ' || max_poll_records || ', ' ||
       session_timeout_ms || ', ''' || status || ''', SYSTIMESTAMP, SYSTIMESTAMP);'
FROM topic_configurations;

-- Add more tables as needed
-- orchestration_rules, fx_transactions, etc.

SELECT 'COMMIT;' FROM dual;
EXIT;
EOF
```

### Step 4: Create Transfer Package

```bash
# Compress everything
tar -czf fx-export-$(date +%Y%m%d).tar.gz \
  fx-backend.tar \
  fx-frontend.tar \
  fx-test-emitter.tar \
  fx-database-export.dmp \
  fx-pod-config.json \
  fx-pod-ports.txt \
  fx-schema.sql

# Verify package
ls -lh fx-export-*.tar.gz
```

### Step 5: Copy Project Files

```bash
# Create a clean copy of project (without node_modules, target, etc.)
cd /path/to/fx-transaction-processor
tar -czf ~/fx-export/fx-project-files.tar.gz \
  --exclude='backend/target' \
  --exclude='frontend/node_modules' \
  --exclude='frontend/dist' \
  --exclude='.git' \
  .
```

---

## Transfer to Production

### Option 1: SCP Transfer

```bash
# From development machine
scp ~/fx-export/fx-export-*.tar.gz user@production-server:/tmp/
scp ~/fx-export/fx-project-files.tar.gz user@production-server:/tmp/
```

### Option 2: Physical Media (for large files)

```bash
# Copy to USB drive
cp ~/fx-export/*.tar.gz /media/usb/

# On production server, copy from USB
cp /media/usb/*.tar.gz /tmp/
```

### Option 3: Network Share

```bash
# Mount network share and copy
mount //fileserver/share /mnt/share
cp ~/fx-export/*.tar.gz /mnt/share/
```

---

## Import Process (Production Environment)

### Step 1: Prepare Production Environment

```bash
# SSH to production server
ssh user@production-server

# Create working directory
mkdir -p ~/fx-production
cd ~/fx-production

# Extract transfer package
tar -xzf /tmp/fx-export-*.tar.gz

# Extract project files
tar -xzf /tmp/fx-project-files.tar.gz -C ~/fx-production/project
```

### Step 2: Import Container Images

```bash
# Import backend image
podman load -i fx-backend.tar

# Import frontend image
podman load -i fx-frontend.tar

# Import test-emitter image
podman load -i fx-test-emitter.tar

# Verify images loaded
podman images | grep fx-transaction

# Expected output:
# localhost/fx-transaction-processor-backend         latest  ...
# localhost/fx-transaction-processor-frontend        latest  ...
# localhost/fx-transaction-processor-test-data-emitter latest ...
```

### Step 3: Pull Required Base Images

```bash
# Pull Oracle XE (if not exported)
podman pull container-registry.oracle.com/database/express:21.3.0-xe

# Pull Kafka
podman pull confluentinc/cp-kafka:7.5.0

# Pull Zookeeper
podman pull confluentinc/cp-zookeeper:7.5.0
```

### Step 4: Create Volumes

```bash
# Create persistent volumes for data
podman volume create fx-oracle-data
podman volume create fx-kafka-data
podman volume create fx-zookeeper-data
```

### Step 5: Stop Existing Pod (if any)

```bash
# Stop and remove existing pod (if it exists)
podman pod stop fx-pod
podman pod rm fx-pod
```

---

## Database Manual Steps

**CRITICAL: These steps must be done after starting Oracle container but before starting backend**

### Step 1: Start Oracle Container Standalone First

```bash
# Start Oracle in the pod first, before other services
podman run -d \
  --name fx-oracle-xe \
  --pod fx-pod \
  -e ORACLE_PWD=OraclePassword123 \
  -e ORACLE_CHARACTERSET=AL32UTF8 \
  -v fx-oracle-data:/opt/oracle/oradata \
  --health-cmd='sqlplus -s system/$ORACLE_PWD@XEPDB1 <<< "SELECT 1 FROM DUAL;" || exit 1' \
  --health-interval=30s \
  --health-timeout=10s \
  --health-retries=5 \
  container-registry.oracle.com/database/express:21.3.0-xe

# Wait for Oracle to be healthy (this may take 2-3 minutes on first start)
echo "Waiting for Oracle to be healthy..."
until [ "$(podman inspect --format='{{.State.Health.Status}}' fx-oracle-xe)" == "healthy" ]; do
  echo -n "."
  sleep 10
done
echo ""
echo "Oracle is healthy!"
```

### Step 2: Import Database Schema and Data

**Option A: Using Data Pump Export**

```bash
# Copy dump file to container
podman cp fx-database-export.dmp fx-oracle-xe:/opt/oracle/admin/XE/dpdump/

# Import data
podman exec fx-oracle-xe bash -c "impdp system/OraclePassword123@XEPDB1 \
  directory=DATA_PUMP_DIR \
  dumpfile=fx_export.dmp \
  logfile=fx_import.log \
  remap_schema=SYSTEM:SYSTEM"
```

**Option B: Using SQL Script (Recommended)**

```bash
# Copy SQL script to container
podman cp fx-schema.sql fx-oracle-xe:/tmp/

# Execute SQL script
podman exec -i fx-oracle-xe sqlplus system/OraclePassword123@XEPDB1 < fx-schema.sql
```

### Step 3: Apply Critical Fixes to Database

**This is the MOST IMPORTANT step - fixes Kafka connection issues**

```bash
podman exec -i fx-oracle-xe sqlplus -s system/OraclePassword123@XEPDB1 <<'EOF'
-- Fix 1: Update bootstrap_servers from localhost:9092 to kafka:29092
UPDATE topic_configurations
SET bootstrap_servers = 'kafka:29092'
WHERE bootstrap_servers LIKE '%localhost%' OR bootstrap_servers LIKE '%127.0.0.1%';

-- Fix 2: Ensure security protocol is PLAINTEXT (not SSL)
UPDATE topic_configurations
SET security_protocol = 'PLAINTEXT'
WHERE security_protocol = 'SSL' OR security_protocol IS NULL;

-- Fix 3: Add test-fx-data topic if it doesn't exist
MERGE INTO topic_configurations tc
USING (SELECT 'test-fx-data' AS topic_name FROM dual) src
ON (tc.topic_name = src.topic_name)
WHEN NOT MATCHED THEN
  INSERT (topic_name, description, bootstrap_servers, group_id,
          ssl_enabled, security_protocol, auto_offset_reset,
          max_poll_records, session_timeout_ms, status,
          created_at, updated_at)
  VALUES ('test-fx-data', 'Test FX data from emitter',
          'kafka:29092', 'fx-processor-group',
          0, 'PLAINTEXT', 'earliest',
          500, 30000, 'ACTIVE',
          SYSTIMESTAMP, SYSTIMESTAMP);

-- Fix 4: Ensure fx-eur-usd is ACTIVE (main topic for test emitter)
UPDATE topic_configurations
SET status = 'ACTIVE'
WHERE topic_name = 'fx-eur-usd';

COMMIT;

-- Verify changes
SET PAGESIZE 50
SET LINESIZE 120
COLUMN topic_name FORMAT A20
COLUMN bootstrap_servers FORMAT A20
COLUMN security_protocol FORMAT A15
COLUMN status FORMAT A10

SELECT id, topic_name, bootstrap_servers, security_protocol, status
FROM topic_configurations
ORDER BY id;

EXIT;
EOF
```

**Expected output after verification:**
```
ID  TOPIC_NAME           BOOTSTRAP_SERVERS    SECURITY_PROTOCOL  STATUS
--- -------------------- -------------------- ------------------ ----------
1   fx-eur-usd           kafka:29092          PLAINTEXT          ACTIVE
2   fx-gbp-usd           kafka:29092          PLAINTEXT          INACTIVE
3   fx-usd-jpy           kafka:29092          PLAINTEXT          INACTIVE
...
13  test-fx-data         kafka:29092          PLAINTEXT          ACTIVE
```

**All bootstrap_servers MUST show `kafka:29092` (NOT localhost:9092)**

### Step 4: Verify Database Changes Were Applied

```bash
# Quick verification
podman exec -i fx-oracle-xe sqlplus -s system/OraclePassword123@XEPDB1 <<'EOF'
SELECT COUNT(*) AS "Correct Configs"
FROM topic_configurations
WHERE bootstrap_servers = 'kafka:29092'
  AND security_protocol = 'PLAINTEXT';
EXIT;
EOF

# Should show at least 11 (10 original + test-fx-data)
```

---

## Pod Creation and Startup

### Step 1: Create Pod with Port Mappings

```bash
# Create pod with all required port mappings
podman pod create \
  --name fx-pod \
  --publish 80:80 \
  --publish 8080:8080 \
  --publish 1521:1521 \
  --publish 9092:9092 \
  --publish 2181:2181 \
  --network bridge

# Verify pod created
podman pod ls
```

### Step 2: Start Zookeeper

```bash
podman run -d \
  --name fx-zookeeper \
  --pod fx-pod \
  -e ZOOKEEPER_CLIENT_PORT=2181 \
  -e ZOOKEEPER_TICK_TIME=2000 \
  -e ZOOKEEPER_SYNC_LIMIT=2 \
  -v fx-zookeeper-data:/var/lib/zookeeper/data \
  --health-cmd='echo ruok | nc localhost 2181 | grep imok' \
  --health-interval=30s \
  --health-timeout=10s \
  --health-retries=3 \
  confluentinc/cp-zookeeper:7.5.0

# Wait for Zookeeper to be healthy
echo "Waiting for Zookeeper..."
until [ "$(podman inspect --format='{{.State.Health.Status}}' fx-zookeeper)" == "healthy" ]; do
  echo -n "."
  sleep 5
done
echo "Zookeeper is healthy!"
```

### Step 3: Start Kafka

```bash
podman run -d \
  --name fx-kafka \
  --pod fx-pod \
  -e KAFKA_BROKER_ID=1 \
  -e KAFKA_ZOOKEEPER_CONNECT=localhost:2181 \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:29092,PLAINTEXT_HOST://localhost:9092 \
  -e KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT \
  -e KAFKA_INTER_BROKER_LISTENER_NAME=PLAINTEXT \
  -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
  -e KAFKA_TRANSACTION_STATE_LOG_MIN_ISR=1 \
  -e KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR=1 \
  -e KAFKA_AUTO_CREATE_TOPICS_ENABLE=true \
  -v fx-kafka-data:/var/lib/kafka/data \
  --health-cmd='kafka-broker-api-versions --bootstrap-server localhost:9092 || exit 1' \
  --health-interval=30s \
  --health-timeout=10s \
  --health-retries=5 \
  confluentinc/cp-kafka:7.5.0

# Wait for Kafka to be healthy
echo "Waiting for Kafka..."
until [ "$(podman inspect --format='{{.State.Health.Status}}' fx-kafka)" == "healthy" ]; do
  echo -n "."
  sleep 5
done
echo "Kafka is healthy!"
```

### Step 4: Start Oracle (if not already started in Database Manual Steps)

If you didn't start Oracle in the Database Manual Steps section, start it now:

```bash
podman run -d \
  --name fx-oracle-xe \
  --pod fx-pod \
  -e ORACLE_PWD=OraclePassword123 \
  -e ORACLE_CHARACTERSET=AL32UTF8 \
  -v fx-oracle-data:/opt/oracle/oradata \
  --health-cmd='sqlplus -s system/$ORACLE_PWD@XEPDB1 <<< "SELECT 1 FROM DUAL;" || exit 1' \
  --health-interval=30s \
  --health-timeout=10s \
  --health-retries=5 \
  container-registry.oracle.com/database/express:21.3.0-xe

# Wait for Oracle
echo "Waiting for Oracle..."
until [ "$(podman inspect --format='{{.State.Health.Status}}' fx-oracle-xe)" == "healthy" ]; do
  echo -n "."
  sleep 10
done
echo "Oracle is healthy!"
```

### Step 5: Start Backend

```bash
podman run -d \
  --name fx-backend \
  --pod fx-pod \
  -e SPRING_DATASOURCE_URL=jdbc:oracle:thin:@localhost:1521/XEPDB1 \
  -e SPRING_DATASOURCE_USERNAME=system \
  -e SPRING_DATASOURCE_PASSWORD=OraclePassword123 \
  -e SPRING_KAFKA_BOOTSTRAP_SERVERS=localhost:29092 \
  -e SPRING_KAFKA_PROPERTIES_SECURITY_PROTOCOL=PLAINTEXT \
  --health-cmd='wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1' \
  --health-interval=30s \
  --health-timeout=3s \
  --health-start-period=60s \
  --health-retries=3 \
  localhost/fx-transaction-processor-backend:latest

# Wait for backend
echo "Waiting for Backend..."
until [ "$(podman inspect --format='{{.State.Health.Status}}' fx-backend)" == "healthy" ]; do
  echo -n "."
  sleep 5
done
echo "Backend is healthy!"
```

### Step 6: Start Frontend

```bash
podman run -d \
  --name fx-frontend \
  --pod fx-pod \
  -e API_URL=/api \
  -e WS_URL=/ws \
  --health-cmd='wget --no-verbose --tries=1 --spider http://localhost:80/ || exit 1' \
  --health-interval=30s \
  --health-timeout=3s \
  --health-retries=3 \
  localhost/fx-transaction-processor-frontend:latest

# Wait for frontend
echo "Waiting for Frontend..."
until [ "$(podman inspect --format='{{.State.Health.Status}}' fx-frontend)" == "healthy" ]; do
  echo -n "."
  sleep 5
done
echo "Frontend is healthy!"
```

### Step 7: Start Test Data Emitter

```bash
podman run -d \
  --name fx-test-emitter \
  --pod fx-pod \
  -e KAFKA_BOOTSTRAP_SERVERS=localhost:29092 \
  -e EMIT_INTERVAL_MS=2000 \
  localhost/fx-transaction-processor-test-data-emitter:latest

echo "Test emitter started!"
```

---

## Verification

### Step 1: Check All Containers in Pod

```bash
# List all containers in pod
podman ps --pod --filter pod=fx-pod

# Should show all 6 containers running:
# - fx-zookeeper
# - fx-kafka
# - fx-oracle-xe
# - fx-backend
# - fx-frontend
# - fx-test-emitter
```

### Step 2: Verify Health Status

```bash
# Check health of all containers
for container in fx-zookeeper fx-kafka fx-oracle-xe fx-backend fx-frontend; do
  status=$(podman inspect --format='{{.State.Health.Status}}' $container 2>/dev/null || echo "no-health-check")
  echo "$container: $status"
done
```

### Step 3: Verify Backend Kafka Connection

```bash
# Check backend logs for Kafka connection
podman logs fx-backend --tail 50 | grep -E "kafka|bootstrap.servers|Subscribed to topic"

# Should see:
# ✅ bootstrap.servers = [localhost:29092]
# ✅ Subscribed to topic(s): fx-eur-usd
# ✅ partitions assigned: [fx-eur-usd-0]
```

### Step 4: Verify Test Emitter is Sending Data

```bash
# Check test emitter logs
podman logs fx-test-emitter --tail 20

# Should see:
# Sent EUR/USD to topic fx-eur-usd (bid=1.10423, ask=1.10462)
# Sent USD/JPY to topic fx-usd-jpy (bid=149.39960, ask=149.39998)
```

### Step 5: Verify Backend is Consuming Messages

```bash
# Look for messages being processed
podman logs fx-backend --tail 100 | grep -i "orchestration\|received message"

# Should see:
# WARN - No enabled orchestration rules found for topic: fx-eur-usd
# (This means messages ARE being received!)
```

### Step 6: Test Frontend Access

```bash
# Check if frontend is accessible
curl -I http://localhost:80

# Should return: HTTP/1.1 200 OK

# Or access from browser:
# http://PRODUCTION_SERVER_IP:80
```

### Step 7: Test Backend Health Endpoint

```bash
# Check backend health
curl http://localhost:8080/actuator/health

# Should return: {"status":"UP"}
```

---

## Troubleshooting

### Issue 1: Oracle Not Starting

```bash
# Check Oracle logs
podman logs fx-oracle-xe

# Common issues:
# - Insufficient memory (needs at least 2GB)
# - Volume permission issues

# Fix volume permissions:
podman unshare chown -R 54321:54321 $(podman volume inspect fx-oracle-data --format '{{.Mountpoint}}')
```

### Issue 2: Backend Can't Connect to Oracle

```bash
# Test connection from backend
podman exec fx-backend ping -c 2 localhost

# In a pod, all containers share localhost network
# Check Oracle port is listening:
podman exec fx-oracle-xe netstat -tlnp | grep 1521
```

### Issue 3: Backend Can't Connect to Kafka

```bash
# Check TopicConfiguration in database
podman exec -i fx-oracle-xe sqlplus -s system/OraclePassword123@XEPDB1 <<'EOF'
SELECT topic_name, bootstrap_servers, security_protocol
FROM topic_configurations
WHERE status = 'ACTIVE';
EXIT;
EOF

# If showing localhost:9092, re-run the database fixes from "Database Manual Steps"
```

### Issue 4: Snappy Compression Error

```bash
# Check if backend image is using Debian (not Alpine)
podman inspect localhost/fx-transaction-processor-backend:latest | grep -i "alpine\|jammy"

# Should show "jammy" (Debian)
# If showing "alpine", you need to rebuild the backend image
```

### Issue 5: No Messages Being Consumed

```bash
# Check Kafka topics exist
podman exec fx-kafka kafka-topics --list --bootstrap-server localhost:9092

# Check messages in topic
podman exec fx-kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic fx-eur-usd \
  --from-beginning \
  --max-messages 5
```

### Issue 6: Port Already in Use

```bash
# Find what's using the port
sudo ss -tlnp | grep ':80\|:8080\|:1521'

# Stop conflicting service or change pod port mapping
podman pod rm fx-pod
podman pod create --name fx-pod --publish 8080:80 --publish 8081:8080 ...
```

---

## Post-Migration Checklist

After successful migration:

- [ ] All 6 containers running in fx-pod
- [ ] All health checks passing
- [ ] Backend connects to Oracle (no TNS errors)
- [ ] Backend connects to Kafka (kafka:29092 or localhost:29092)
- [ ] TopicConfiguration has correct bootstrap_servers
- [ ] No Snappy compression errors
- [ ] Test emitter sending data
- [ ] Backend consuming messages
- [ ] Frontend accessible via browser
- [ ] WebSocket connection working
- [ ] Can create orchestration rules
- [ ] Firewall configured for external access (if needed)

---

## Pod Management Commands

### Start/Stop Pod

```bash
# Stop entire pod (all containers)
podman pod stop fx-pod

# Start entire pod
podman pod start fx-pod

# Restart pod
podman pod restart fx-pod
```

### View Pod Logs

```bash
# Logs from all containers in pod
podman pod logs fx-pod

# Logs from specific container
podman logs fx-backend -f
```

### Remove Pod (CAUTION)

```bash
# Stop and remove pod (containers remain)
podman pod rm fx-pod

# Remove pod and all containers
podman pod rm -f fx-pod

# Remove volumes (DESTROYS DATA)
podman volume rm fx-oracle-data fx-kafka-data fx-zookeeper-data
```

### Export Pod for Backup

```bash
# Generate pod YAML for recreation
podman generate kube fx-pod > fx-pod-backup.yaml

# Later restore with:
podman play kube fx-pod-backup.yaml
```

---

## Automated Migration Script

Save this as `migrate-to-pod.sh`:

```bash
#!/bin/bash
# See next artifact for full script
```

---

## Summary

This guide covered:
1. ✅ Exporting containers and data from development
2. ✅ Transferring to production server
3. ✅ Importing images and data
4. ✅ **Critical database fixes** (bootstrap_servers, security_protocol)
5. ✅ Creating pod with proper configuration
6. ✅ Starting all containers in correct order
7. ✅ Comprehensive verification steps
8. ✅ Troubleshooting common issues

**Key Differences from docker-compose:**
- All containers in one pod share localhost network
- Use `localhost:29092` for Kafka (instead of `kafka:29092`)
- Or use `localhost:1521` for Oracle (instead of `oracle-xe:1521`)
- Pod started/stopped as a single unit

**Critical Manual Steps:**
- Database bootstrap_servers MUST be `kafka:29092` (within container network)
- Security protocol MUST be `PLAINTEXT`
- These cannot be auto-migrated and MUST be fixed manually

Your production environment is now running with Podman pods! 🚀
