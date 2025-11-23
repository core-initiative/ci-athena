## PostgreSQL HA Deployment

For a fully manual YAML setup that provides basic replication, you would need:

1.  **Secret** (for credentials).
2.  **ConfigMap** (for PostgreSQL configuration and custom replication settings).
3.  **Services** (Primary and Read Replica).
4.  **StatefulSet** (to manage the primary and replica pods).

**Note:** This YAML **DOES NOT** include an automatic failover solution (like Patroni). If the primary pod dies, the replicas will not automatically promote one another.

### 1\. 🔑 PostgreSQL Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secrets
type: Opaque
data:
  # Base64 encoded: e.g., 'kong'
  POSTGRES_USER: a29uZw==
  # Base64 encoded: e.g., 'supersecretpassword'
  POSTGRES_PASSWORD: c3VwZXJzZWNyZXRwYXNzd29yZA==
  # Base64 encoded: e.g., 'replication_user'
  REPLICATION_USER: cmVwbGljYXRpb25fdXNlcg==
  # Base64 encoded: e.g., 'repl_password'
  REPLICATION_PASSWORD: cmVwbF9wYXNzd29yZA==
```

### 2\. 🗺️ ConfigMap (Primary Configuration)

This ConfigMap contains the custom `postgresql.conf` settings needed for streaming replication, including `wal_level`, `max_wal_senders`, and the **`hot_standby`** parameter.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
data:
  postgresql.conf: |
    # Core Settings
    listen_addresses = '*'
    port = 5432
    max_connections = 100
    # Replication Settings
    wal_level = replica
    max_wal_senders = 10
    max_replication_slots = 10
    hot_standby = on
    synchronous_commit = off
    synchronous_standby_names = ''

  init-primary.sh: |
    #!/bin/bash
    # Script to initialize the primary database and create replication user
    psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" <<-EOSQL
        CREATE USER $REPLICATION_USER WITH REPLICATION ENCRYPTED PASSWORD '$REPLICATION_PASSWORD';
    EOSQL
```

### 3\. 🌐 Services (Primary and Replica)

We use two services: one for the primary (write) and one for replicas (read).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-primary
  labels:
    app: postgres
    role: primary
spec:
  ports:
  - port: 5432
    name: postgres
  selector:
    app: postgres
    role: primary # Targets only the primary pod
  type: ClusterIP

---
apiVersion: v1
kind: Service
metadata:
  name: postgres-read-replica
  labels:
    app: postgres
    role: replica
spec:
  ports:
  - port: 5432
    name: postgres
  selector:
    app: postgres
    role: replica # Targets all replica pods
  type: ClusterIP
```

### 4\. 🗄️ StatefulSet (Primary and Replicas)

This StatefulSet uses a simple entrypoint to determine if the pod is the **primary (index 0)** or a **replica (index 1 and higher)**.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-ha
spec:
  serviceName: "postgres-headless"
  replicas: 3 # 1 Primary, 2 Replicas
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
          name: postgres
        envFrom:
        - secretRef:
            name: postgres-secrets
        env:
        - name: POSTGRES_DB
          value: kong # DB Kong will connect to
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: PRIMARY_HOST
          value: "postgres-ha-0.postgres-headless" # Stable network ID of the primary
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        # Mount the postgresql.conf to the standard config directory
        - name: postgres-config-mount 
          mountPath: /etc/postgresql/
          readOnly: true
        # Mount the init script to run on primary startup
        - name: postgres-config-init 
          mountPath: /docker-entrypoint-initdb.d/
          readOnly: true
        
        # *** The logic that determines Primary vs. Replica ***
        command: ["/bin/sh", "-c"]
        args:
        - |
          # Define config paths
          CONFIG_DIR="/etc/postgresql"
          CONFIG_FILE="$CONFIG_DIR/postgresql.conf"
          HBA_FILE="$CONFIG_DIR/pg_hba.conf"
          
          # Link mounted pg_hba.conf and postgresql.conf 
          ln -sf $CONFIG_DIR/pg_hba.conf /var/lib/postgresql/data/pg_hba.conf
          ln -sf $CONFIG_FILE /var/lib/postgresql/data/postgresql.conf
          
          if [ "$POD_NAME" = "postgres-ha-0" ]; then
            # --- PRIMARY (POD 0) ---
            echo "Starting as Primary"
            # Label this pod as the primary for the Service selector
            kubectl label pod $POD_NAME role=primary --overwrite -n $(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
            
            # This runs the init script to create the replication user
            /usr/local/bin/docker-entrypoint.sh
            
          else
            # --- REPLICA (POD 1, 2, ...) ---
            echo "Starting as Replica, performing base backup..."
            
            # Label this pod as a replica for the Service selector
            kubectl label pod $POD_NAME role=replica --overwrite -n $(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)

            # Ensure data directory is empty before backup
            rm -rf /var/lib/postgresql/data/*
            
            # Perform base backup from the primary
            PGPASSWORD="$REPLICATION_PASSWORD" pg_basebackup \
              -h "$PRIMARY_HOST" \
              -U "$REPLICATION_USER" \
              -D /var/lib/postgresql/data/ \
              -P \
              -R \
              -S "$POD_NAME"
            
            # Ensure the replication configuration is linked/copied
            # The -R option in pg_basebackup creates a standby.signal file

          fi
          
          # Start PostgreSQL using the mounted config file
          # The startup command is the same for both primary and replica now
          # Note: The init script for the primary is handled by the entrypoint logic above
          exec /usr/local/bin/docker-entrypoint.sh postgres -c config_file=$CONFIG_FILE
          
      volumes:
      - name: postgres-config-mount
        configMap:
          name: postgres-config
          items:
          - key: postgresql.conf
            path: postgresql.conf 
          - key: pg_hba.conf
            path: pg_hba.conf 
      - name: postgres-config-init
        configMap:
          name: postgres-config
          items:
          - key: init-primary.sh
            path: init-primary.sh
  
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi # CHANGE THIS: Ensure your StorageClass is available
```