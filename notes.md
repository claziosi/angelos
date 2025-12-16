# Option 2: NodePort Service - Detailed Explanation

## How NodePort Works

### Architecture

```
EVZ02 Cluster                Network              EVZ01 Cluster
┌──────────────┐                                 ┌──────────────────┐
│ PostgreSQL   │                                 │  Node 1          │
│ Standby Pod  │        Internet/VPN             │  IP: 10.0.1.10   │
│              │───────────────────────────────> │  :30432 ──┐      │
│ Connects to: │                                 │           │      │
│ 10.0.1.10    │                                 │  Node 2   │      │
│ Port: 30432  │                                 │  IP: 10.0.1.11   │
└──────────────┘                                 │  :30432 ──┤      │
                                                 │           │      │
                                                 │  Node 3   │      │
                                                 │  IP: 10.0.1.12   │
                                                 │  :30432 ──┤      │
                                                 │           ▼      │
                                                 │    ┌──────────┐  │
                                                 │    │ Service  │  │
                                                 │    │ ClusterIP│  │
                                                 │    └────┬─────┘  │
                                                 │         │        │
                                                 │    ┌────▼─────┐  │
                                                 │    │PostgreSQL│  │
                                                 │    │Primary   │  │
                                                 │    │Pod       │  │
                                                 │    └──────────┘  │
                                                 └──────────────────┘
```

### How It Works

1. **NodePort allocation**: Kubernetes/OpenShift allocates a port (30000-32767) on **ALL nodes** in the cluster
2. **Traffic routing**: Any traffic hitting `<any-node-ip>:<nodeport>` is forwarded to the Service
3. **Service forwarding**: The Service routes traffic to the appropriate Pod (PostgreSQL primary)
4. **Cross-cluster access**: EVZ02 can connect to any node in EVZ01 on the NodePort

**Key point**: You can connect to **ANY** node IP in EVZ01, and the traffic will be routed to the correct Pod.

## Complete NodePort Configuration

### Step 1: Create NodePort Service on EVZ01

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gravitee-db-nodeport
  namespace: gravitee
  labels:
    app: gravitee-db-external
spec:
  type: NodePort
  sessionAffinity: ClientIP  # Keep connections sticky
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 hours
  ports:
    - name: postgres
      port: 5432
      targetPort: postgres
      nodePort: 30432  # Optional: specify port, otherwise auto-assigned
      protocol: TCP
  selector:
    postgres-operator.crunchydata.com/cluster: gravitee-db
    postgres-operator.crunchydata.com/role: master
```

### Step 2: Verify Service Creation

```bash
# Get the NodePort service
oc get svc gravitee-db-nodeport -n gravitee

# Output:
# NAME                    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
# gravitee-db-nodeport    NodePort   172.30.123.45   <none>        5432:30432/TCP   5s

# Get all node IPs
oc get nodes -o wide

# Output:
# NAME                 STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP
# evz01-master-1       Ready    master   30d   v1.27.0   10.0.1.10     203.0.113.10
# evz01-worker-1       Ready    worker   30d   v1.27.0   10.0.1.20     203.0.113.20
# evz01-worker-2       Ready    worker   30d   v1.27.0   10.0.1.21     203.0.113.21
```

### Step 3: Test Connection

```bash
# From EVZ02 or any external location
# Test with internal IP (if VPN/private network exists)
psql -h 10.0.1.20 -p 30432 -U postgres -d gravitee

# Or with external IP (if nodes have public IPs)
psql -h 203.0.113.20 -p 30432 -U postgres -d gravitee

# Test with telnet
telnet 10.0.1.20 30432
```

## Security Considerations

### ⚠️ NodePort is NOT encrypted by default

**Problem**: NodePort exposes raw TCP traffic without encryption. PostgreSQL traffic is **unencrypted** unless you configure SSL/TLS.

### Security Risks

1. **Unencrypted traffic**: Passwords and data travel in cleartext
2. **Exposed on all nodes**: The port is open on every node
3. **No authentication layer**: Direct access to PostgreSQL
4. **Network visibility**: Anyone with network access can attempt connections

## How to Secure NodePort with SSL/TLS

### Option A: PostgreSQL Native SSL (Recommended)

#### Step 1: Generate SSL Certificates

```bash
# On your local machine or a secure location
# Create CA (Certificate Authority)
openssl genrsa -out ca-key.pem 4096
openssl req -new -x509 -days 3650 -key ca-key.pem -out ca-cert.pem \
  -subj "/CN=PostgreSQL-CA"

# Create server certificate
openssl genrsa -out server-key.pem 4096
openssl req -new -key server-key.pem -out server-req.pem \
  -subj "/CN=gravitee-db.gravitee.svc.cluster.local"

# Sign server certificate with CA
openssl x509 -req -days 3650 -in server-req.pem \
  -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial \
  -out server-cert.pem

# Set correct permissions
chmod 600 server-key.pem
```

#### Step 2: Create Secret with Certificates

```bash
# On EVZ01
oc create secret generic gravitee-db-ssl-certs \
  -n gravitee \
  --from-file=ca.crt=ca-cert.pem \
  --from-file=tls.crt=server-cert.pem \
  --from-file=tls.key=server-key.pem
```

#### Step 3: Configure PostgresCluster with SSL

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
      replicas: 2
      dataVolumeClaimSpec:
        accessModes:
          - "ReadWriteOnce"
        resources:
          requests:
            storage: 50Gi
  
  # Enable custom TLS certificates
  customTLSSecret:
    name: gravitee-db-ssl-certs
  
  # Force SSL connections
  patroni:
    dynamicConfiguration:
      postgresql:
        parameters:
          ssl: "on"
          ssl_cert_file: "/pgconf/tls/tls.crt"
          ssl_key_file: "/pgconf/tls/tls.key"
          ssl_ca_file: "/pgconf/tls/ca.crt"
          ssl_min_protocol_version: "TLSv1.2"
          ssl_ciphers: "HIGH:MEDIUM:+3DES:!aNULL"
          # Require SSL for all connections
          ssl_prefer_server_ciphers: "on"
  
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

#### Step 4: Configure pg_hba.conf to Require SSL

```yaml
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: gravitee-db
  namespace: gravitee
spec:
  # ... previous config ...
  
  patroni:
    dynamicConfiguration:
      postgresql:
        parameters:
          ssl: "on"
        pg_hba:
          - "hostssl all all 0.0.0.0/0 scram-sha-256"  # Require SSL for all
          - "hostssl replication all 0.0.0.0/0 scram-sha-256"  # SSL for replication
```

#### Step 5: Configure EVZ02 Standby with SSL

```bash
# Copy CA certificate to EVZ02
oc create secret generic gravitee-db-ca-cert \
  -n gravitee \
  --from-file=ca.crt=ca-cert.pem
```

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
      replicas: 1
      dataVolumeClaimSpec:
        accessModes:
          - "ReadWriteOnce"
        resources:
          requests:
            storage: 50Gi
  
  standby:
    enabled: true
    repoName: repo1
    host: "10.0.1.20"  # EVZ01 node IP
    port: 30432
  
  # Configure SSL for standby connection
  customTLSSecret:
    name: gravitee-db-ca-cert
  
  patroni:
    dynamicConfiguration:
      postgresql:
        parameters:
          ssl: "on"
```

#### Step 6: Test SSL Connection

```bash
# From EVZ02, test SSL connection
oc run -it --rm ssl-test --image=postgres:15 --restart=Never -- \
  psql "postgresql://postgres@10.0.1.20:30432/gravitee?sslmode=require&sslrootcert=/path/to/ca.crt"

# Check if SSL is active
oc exec -it <postgres-pod> -n gravitee -- \
  psql -c "SELECT * FROM pg_stat_ssl;"
```

### Option B: Stunnel/TLS Proxy (Alternative)

If you can't modify PostgreSQL SSL settings, use a TLS proxy:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-stunnel
  namespace: gravitee
spec:
  replicas: 2
  selector:
    matchLabels:
      app: postgres-stunnel
  template:
    metadata:
      labels:
        app: postgres-stunnel
    spec:
      containers:
      - name: stunnel
        image: dweomer/stunnel
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: stunnel-config
          mountPath: /etc/stunnel
        - name: certs
          mountPath: /etc/stunnel/certs
      volumes:
      - name: stunnel-config
        configMap:
          name: stunnel-config
      - name: certs
        secret:
          secretName: stunnel-certs
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: stunnel-config
  namespace: gravitee
data:
  stunnel.conf: |
    foreground = yes
    [postgres]
    accept = 5432
    connect = gravitee-db-primary:5432
    cert = /etc/stunnel/certs/tls.crt
    key = /etc/stunnel/certs/tls.key
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-stunnel-nodeport
  namespace: gravitee
spec:
  type: NodePort
  ports:
    - port: 5432
      targetPort: 5432
      nodePort: 30432
  selector:
    app: postgres-stunnel
```

## Network Security Best Practices

### 1. Firewall Rules

```bash
# Only allow EVZ02 cluster IPs to access NodePort on EVZ01
# Example iptables rule on EVZ01 nodes:

iptables -A INPUT -p tcp --dport 30432 -s 10.130.0.0/16 -j ACCEPT  # EVZ02 pod network
iptables -A INPUT -p tcp --dport 30432 -j DROP  # Drop all other traffic
```

### 2. OpenShift NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: gravitee-db-external-access
  namespace: gravitee
spec:
  podSelector:
    matchLabels:
      postgres-operator.crunchydata.com/cluster: gravitee-db
  policyTypes:
    - Ingress
  ingress:
    # Allow from specific IP ranges only
    - from:
        - ipBlock:
            cidr: 10.130.0.0/16  # EVZ02 pod network
        - ipBlock:
            cidr: 192.168.50.0/24  # Your admin network
      ports:
        - protocol: TCP
          port: 5432
```

### 3. Use VPN/Private Network

```
┌─────────────┐                    ┌─────────────┐
│   EVZ02     │                    │   EVZ01     │
│             │                    │             │
│  ┌────────┐ │    Site-to-Site   │  ┌────────┐ │
│  │  VPN   │◄├────── VPN ────────┤►│  VPN   │ │
│  │ Client │ │     (IPSec/       │  │ Server │ │
│  └────────┘ │    WireGuard)     │  └────────┘ │
│      │      │                    │      │      │
│      ▼      │                    │      ▼      │
│ PostgreSQL  │   Encrypted        │ NodePort   │
│  Standby    │   Tunnel           │  :30432    │
└─────────────┘                    └─────────────┘
```

## Performance Considerations

### NodePort Limitations

1. **Extra network hop**: Traffic goes through kube-proxy
2. **SNAT/DNAT**: Source IP may be changed
3. **Session affinity**: Important for PostgreSQL connections

### Optimize with externalTrafficPolicy

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gravitee-db-nodeport
  namespace: gravitee
spec:
  type: NodePort
  externalTrafficPolicy: Local  # Avoid extra hops
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
      nodePort: 30432
      protocol: TCP
  selector:
    postgres-operator.crunchydata.com/cluster: gravitee-db
    postgres-operator.crunchydata.com/role: master
```

**externalTrafficPolicy: Local** benefits:
- Preserves source IP address
- Reduces latency (no extra hop through kube-proxy)
- Direct routing to Pod

**Caveat**: Only routes to nodes where the Pod is running

## Complete Secure Setup Example

```yaml
# On EVZ01 - Complete secure configuration
---
# 1. SSL Certificates Secret
apiVersion: v1
kind: Secret
metadata:
  name: gravitee-db-ssl
  namespace: gravitee
type: kubernetes.io/tls
data:
  ca.crt: <base64-encoded-ca-cert>
  tls.crt: <base64-encoded-server-cert>
  tls.key: <base64-encoded-server-key>

---
# 2. PostgresCluster with SSL
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: gravitee-db
  namespace: gravitee
spec:
  postgresVersion: 15
  instances:
    - name: instance1
      replicas: 2
      dataVolumeClaimSpec:
        accessModes:
          - "ReadWriteOnce"
        resources:
          requests:
            storage: 50Gi
  
  customTLSSecret:
    name: gravitee-db-ssl
  
  patroni:
    dynamicConfiguration:
      postgresql:
        parameters:
          ssl: "on"
          ssl_min_protocol_version: "TLSv1.2"
        pg_hba:
          - "hostssl all all 0.0.0.0/0 scram-sha-256"
          - "hostssl replication all 0.0.0.0/0 scram-sha-256"
  
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

---
# 3. Secure NodePort Service
apiVersion: v1
kind: Service
metadata:
  name: gravitee-db-nodeport
  namespace: gravitee
spec:
  type: NodePort
  externalTrafficPolicy: Local
  sessionAffinity: ClientIP
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
      nodePort: 30432
      protocol: TCP
  selector:
    postgres-operator.crunchydata.com/cluster: gravitee-db
    postgres-operator.crunchydata.com/role: master

---
# 4. NetworkPolicy for access control
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: gravitee-db-nodeport-access
  namespace: gravitee
spec:
  podSelector:
    matchLabels:
      postgres-operator.crunchydata.com/cluster: gravitee-db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - ipBlock:
            cidr: 10.130.0.0/16  # EVZ02 network
      ports:
        - protocol: TCP
          port: 5432
```

## Monitoring and Troubleshooting

```bash
# Check if NodePort is listening on nodes
ss -tlnp | grep 30432

# Test connection from EVZ02
oc run -it --rm debug --image=postgres:15 --restart=Never -- \
  psql "postgresql://postgres@10.0.1.20:30432/gravitee?sslmode=require"

# Check SSL is active
oc exec -it gravitee-db-instance1-xxxx -n gravitee -- \
  psql -c "SELECT ssl, cipher, bits FROM pg_stat_ssl WHERE pid = pg_backend_pid();"

# Monitor connections
oc exec -it gravitee-db-instance1-xxxx -n gravitee -- \
  psql -c "SELECT client_addr, ssl, cipher FROM pg_stat_ssl JOIN pg_stat_activity ON pg_stat_ssl.pid = pg_stat_activity.pid;"
```

## Summary

| Aspect | Without SSL | With SSL |
|--------|------------|----------|
| **Encryption** | ❌ Plaintext | ✅ TLS 1.2+ |
| **Security** | ⚠️ Low | ✅ High |
| **Performance** | Fast | Slightly slower (~5-10%) |
| **Complexity** | Simple | Moderate |
| **Recommended** | ❌ No | ✅ Yes |

**Bottom line**: NodePort without SSL is **NOT secure** for production. Always use SSL/TLS when exposing PostgreSQL across networks.
