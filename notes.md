# Option 2: Cross-cluster PostgreSQL Replication with Crunchy DB

Here's how to set up a PostgreSQL replication architecture between your two OpenShift clusters:

## Global Architecture

```
EVZ01 (Primary Cluster)                    EVZ02 (Secondary Cluster)
┌─────────────────────────┐                ┌─────────────────────────┐
│ Gravitee (active)       │                │ Gravitee (standby)      │
│         ↓               │                │         ↓               │
│ Crunchy PostgreSQL      │   Replication  │ Crunchy PostgreSQL      │
│ (PRIMARY)               │ ═══════════════>│ (STANDBY/REPLICA)       │
└─────────────────────────┘                └─────────────────────────┘
         ↑                                           ↑
         └────────── [VIP/DNS Endpoint] ────────────┘
```

## Step 1: Primary Cluster Configuration (EVZ01)

Create your Crunchy PostgresCluster with replication enabled:

```yaml
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: gravitee-db
  namespace: gravitee
spec:
  postgresVersion: 15
  instances:
    - name: instance1
      replicas: 2  # Local HA on EVZ01
      dataVolumeClaimSpec:
        accessModes:
          - "ReadWriteOnce"
        resources:
          requests:
            storage: 50Gi
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 1
              podAffinityTerm:
                topologyKey: kubernetes.io/hostname
                labelSelector:
                  matchLabels:
                    postgres-operator.crunchydata.com/cluster: gravitee-db
  
  # Replication configuration
  standby:
    enabled: false  # This cluster is the primary
  
  backups:
    pgbackrest:
      repos:
        - name: repo1
          volume:
            volumeClaimSpec:
              accessModes:
                - "ReadWriteOnce"
              resources:
                requests:
                  storage: 100Gi
  
  # Expose service for cross-cluster replication
  proxy:
    pgBouncer:
      replicas: 2
      config:
        global:
          pool_mode: transaction
```

## Step 2: Expose Primary for Secondary Cluster

Create a route/ingress or LoadBalancer to expose PostgreSQL to EVZ02 cluster:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gravitee-db-replication
  namespace: gravitee
spec:
  type: LoadBalancer  # or NodePort depending on your infrastructure
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
      protocol: TCP
  selector:
    postgres-operator.crunchydata.com/cluster: gravitee-db
    postgres-operator.crunchydata.com/role: master
```

## Step 3: Secondary Cluster Configuration (EVZ02)

On EVZ02, create a PostgresCluster in standby mode:

```yaml
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: gravitee-db-replica
  namespace: gravitee
spec:
  postgresVersion: 15
  instances:
    - name: instance1
      replicas: 1  # Read-only replica
      dataVolumeClaimSpec:
        accessModes:
          - "ReadWriteOnce"
        resources:
          requests:
            storage: 50Gi
  
  # STANDBY mode configuration
  standby:
    enabled: true
    repoName: repo1
    host: "gravitee-db-replication.evz01.example.com"  # EVZ01 cluster endpoint
    port: 5432
  
  backups:
    pgbackrest:
      repos:
        - name: repo1
          volume:
            volumeClaimSpec:
              accessModes:
                - "ReadWriteOnce"
              resources:
                requests:
                  storage: 100Gi
```

## Step 4: Cross-cluster Network Configuration

### A. Configure Network Connectivity

You need to establish connectivity between the two clusters:

- **Site-to-site VPN** between datacenters
- **Submariner** for OpenShift multi-cluster connectivity
- **Network policies** to allow PostgreSQL traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-postgres-replication
  namespace: gravitee
spec:
  podSelector:
    matchLabels:
      postgres-operator.crunchydata.com/cluster: gravitee-db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector: {}
        - podSelector: {}
      ports:
        - protocol: TCP
          port: 5432
```

### B. Configure Secrets for Authentication

Copy replication secrets from EVZ01 to EVZ02:

```bash
# On EVZ01
oc get secret gravitee-db-pguser-postgres -n gravitee -o yaml > postgres-secret.yaml

# Modify namespace if needed, then on EVZ02
oc apply -f postgres-secret.yaml -n gravitee
```

## Step 5: Gravitee Configuration

### On EVZ01 (active)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gravitee-config
  namespace: gravitee
data:
  gravitee.yml: |
    management:
      type: jdbc
      jdbc:
        url: jdbc:postgresql://gravitee-db-primary:5432/gravitee
        username: gravitee
        password: ${GRAVITEE_DB_PASSWORD}
```

### On EVZ02 (standby)

Gravitee in read-only mode or disabled until failover:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gravitee-config
  namespace: gravitee
data:
  gravitee.yml: |
    management:
      type: jdbc
      jdbc:
        url: jdbc:postgresql://gravitee-db-replica-primary:5432/gravitee
        username: gravitee
        password: ${GRAVITEE_DB_PASSWORD}
        # Read-only mode
        readOnly: true
```

## Step 6: Virtual Endpoint with Failover

Create a DNS endpoint or external Load Balancer pointing to the active cluster:

```yaml
# External Load Balancer configuration (HAProxy/F5 example)
frontend gravitee_frontend
    bind *:443
    mode tcp
    default_backend gravitee_backend

backend gravitee_backend
    mode tcp
    balance roundrobin
    option tcp-check
    server evz01 evz01-gravitee.example.com:443 check
    server evz02 evz02-gravitee.example.com:443 check backup
```

## Step 7: Failover Procedure

### Planned Failover (maintenance)

```bash
# 1. On EVZ02: Promote replica to primary
oc patch postgrescluster gravitee-db-replica -n gravitee \
  --type merge -p '{"spec":{"standby":{"enabled":false}}}'

# 2. Wait for promotion
oc wait --for=condition=Ready postgrescluster/gravitee-db-replica -n gravitee

# 3. Set EVZ01 as standby
oc patch postgrescluster gravitee-db -n gravitee \
  --type merge -p '{"spec":{"standby":{"enabled":true,"host":"gravitee-db-replica-primary.evz02.example.com"}}}'

# 4. Switch DNS/LB endpoint to EVZ02
# 5. Restart Gravitee pods on EVZ02
oc rollout restart deployment/gravitee-apim -n gravitee
```

### Automatic Failover

For automatic failover, you would need an additional tool like **Patroni** or an external orchestrator, as Crunchy DB doesn't automatically handle cross-cluster failover.

## Important Considerations

**Network Latency**: Synchronous replication between datacenters can cause performance issues. Favor asynchronous replication:

```yaml
spec:
  patroni:
    dynamicConfiguration:
      synchronous_mode: false
      synchronous_mode_strict: false
```

**Monitoring**: Monitor replication lag:

```bash
oc exec -it gravitee-db-instance1-xxxx -n gravitee -- \
  psql -c "SELECT pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn(), \
  pg_last_wal_receive_lsn() - pg_last_wal_replay_lsn() AS replication_lag;"
```

**Split-brain**: Ensure you have a fencing mechanism to prevent both clusters from becoming primary simultaneously.

## Alternative Approaches

### Using pgBackRest for Standby

Instead of streaming replication, you can use pgBackRest-based standby:

```yaml
# On EVZ02
spec:
  standby:
    enabled: true
    repoName: repo1
    # Point to shared backup repository (S3, GCS, etc.)
```

### Using PostgreSQL Logical Replication

For more flexibility (different PostgreSQL versions, selective replication):

```sql
-- On EVZ01 (primary)
CREATE PUBLICATION gravitee_pub FOR ALL TABLES;

-- On EVZ02 (subscriber)
CREATE SUBSCRIPTION gravitee_sub 
  CONNECTION 'host=evz01-db.example.com port=5432 dbname=gravitee user=replicator' 
  PUBLICATION gravitee_pub;
```

## Health Checks and Monitoring

Create monitoring resources:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-exporter-queries
  namespace: gravitee
data:
  queries.yaml: |
    pg_replication:
      query: "SELECT CASE WHEN pg_is_in_recovery() THEN 0 ELSE 1 END as is_primary, 
              EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))::INT as lag_seconds"
      metrics:
        - is_primary:
            usage: "GAUGE"
            description: "Whether this instance is primary (1) or replica (0)"
        - lag_seconds:
            usage: "GAUGE"
            description: "Replication lag in seconds"
```

This solution works but remains complex. For true multi-datacenter HA, option 1 (external managed DB) or option 3 (MongoDB) would be more robust.
