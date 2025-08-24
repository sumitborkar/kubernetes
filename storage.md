# Storage (Week 4)

**Complete guide to Kubernetes storage, persistent volumes, and data management**

---

## Table of Contents

1. [Storage Fundamentals](#storage-fundamentals)
2. [Persistent Volumes (PVs)](#persistent-volumes-pvs)
3. [Persistent Volume Claims (PVCs)](#persistent-volume-claims-pvcs)
4. [Storage Classes](#storage-classes)
5. [Dynamic Provisioning](#dynamic-provisioning)
6. [Volume Types and CSI Drivers](#volume-types-and-csi-drivers)
7. [Storage Best Practices](#storage-best-practices)
8. [Backup and Disaster Recovery](#backup-and-disaster-recovery)
9. [Practice Labs](#practice-labs)
10. [Hands-on Projects](#hands-on-projects)
11. [Troubleshooting Guide](#troubleshooting-guide)

---

## Storage Fundamentals

### Kubernetes Storage Architecture

Kubernetes storage operates on a layered architecture that separates storage provisioning from consumption:

```
┌─────────────────────────────────────────────────┐
│                Applications                     │
│            (Pods using PVCs)                    │
└─────────────────┬───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│           Persistent Volume Claims              │
│          (Storage Requests/Abstractions)        │
└─────────────────┬───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│            Persistent Volumes                   │
│         (Storage Resources in Cluster)          │
└─────────────────┬───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│              Storage Classes                    │
│          (Storage Templates/Policies)           │
└─────────────────┬───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│          Container Storage Interface            │
│               (CSI Drivers)                     │
└─────────────────┬───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│            Physical Storage                     │
│     (AWS EBS, GCP Disks, NFS, iSCSI, etc.)     │
└─────────────────────────────────────────────────┘
```

### Core Storage Concepts

#### 1. Volume Types
- **Ephemeral Volumes**: Tied to pod lifecycle, temporary storage
- **Persistent Volumes**: Independent of pod lifecycle, durable storage
- **Static Volumes**: Pre-created by administrators
- **Dynamic Volumes**: Created on-demand through storage classes

#### 2. Storage Lifecycle
1. **Provisioning**: Storage resources are created
2. **Binding**: PVCs are matched to PVs
3. **Using**: Pods mount and use the storage
4. **Releasing**: PVC is deleted, volume becomes available
5. **Reclaiming**: Storage is cleaned up based on reclaim policy

#### 3. Access Modes
| Access Mode | Description | Use Case |
|-------------|-------------|----------|
| **ReadWriteOnce (RWO)** | Mount as read-write by single node | Database, file systems |
| **ReadOnlyMany (ROX)** | Mount as read-only by multiple nodes | Shared configuration |
| **ReadWriteMany (RWX)** | Mount as read-write by multiple nodes | Shared storage, NFS |
| **ReadWriteOncePod (RWOP)** | Mount as read-write by single pod | Pod-exclusive storage |

---

## Persistent Volumes (PVs)

### What is a Persistent Volume?

A **Persistent Volume (PV)** is a piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned using Storage Classes. PVs have a lifecycle independent of any individual pod.

### PV Characteristics
- **Cluster Resource**: PVs are cluster-wide resources
- **Independent Lifecycle**: Outlive individual pods
- **Storage Abstraction**: Abstract underlying storage details
- **Reclaim Policies**: Define what happens when PVC is released

### PV Specification

#### Complete PV Example
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
  labels:
    storage-tier: fast
    environment: production
  annotations:
    volume.beta.kubernetes.io/storage-class: "fast-ssd"
spec:
  # Storage capacity
  capacity:
    storage: 10Gi
  
  # Volume mode (Filesystem or Block)
  volumeMode: Filesystem
  
  # Access modes
  accessModes:
  - ReadWriteOnce
  - ReadOnlyMany
  
  # Reclaim policy
  persistentVolumeReclaimPolicy: Retain
  
  # Storage class
  storageClassName: fast-ssd
  
  # Node affinity (for local volumes)
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker-node-1
  
  # Volume source (choose one)
  # AWS EBS
  awsElasticBlockStore:
    volumeID: vol-1234567890abcdef0
    fsType: ext4
  
  # OR NFS
  # nfs:
  #   path: /path/to/nfs/share
  #   server: nfs-server.example.com
  
  # OR Local storage
  # local:
  #   path: /mnt/disks/vol1
  
  # OR CSI driver
  # csi:
  #   driver: ebs.csi.aws.com
  #   volumeHandle: vol-1234567890abcdef0
  #   fsType: ext4

status:
  # System-populated status
  phase: Available
```

### PV States
- **Available**: Free resource, not yet bound to a claim
- **Bound**: Volume is bound to a claim
- **Released**: Claim has been deleted, but resource not yet reclaimed
- **Failed**: Volume has failed its automatic reclamation

### Reclaim Policies

#### 1. Retain Policy
```yaml
# Retain policy - manual cleanup required
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-retain
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain  # Manual cleanup
  storageClassName: manual
  hostPath:
    path: /tmp/data
```

**Characteristics:**
- **Manual Cleanup**: Admin must manually clean up storage
- **Data Preservation**: Data remains on storage system
- **Use Case**: Critical data that needs manual verification

#### 2. Delete Policy
```yaml
# Delete policy - automatic cleanup
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-delete
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete  # Automatic cleanup
  storageClassName: fast
  awsElasticBlockStore:
    volumeID: vol-abcdef1234567890
    fsType: ext4
```

**Characteristics:**
- **Automatic Cleanup**: Storage automatically deleted
- **No Data Recovery**: Data permanently lost
- **Use Case**: Temporary or replaceable data

#### 3. Recycle Policy (Deprecated)
- **Legacy**: No longer supported in recent Kubernetes versions
- **Replaced By**: Dynamic provisioning with Delete policy

### Volume Modes

#### Filesystem Mode (Default)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-filesystem
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem  # Default mode
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: standard
  awsElasticBlockStore:
    volumeID: vol-filesystem123
    fsType: ext4
```

#### Block Mode (Raw Block Storage)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-block
spec:
  capacity:
    storage: 100Gi
  volumeMode: Block      # Raw block device
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: block-storage
  awsElasticBlockStore:
    volumeID: vol-block456
```

### Local Persistent Volumes
```yaml
# Local PV with node affinity
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 50Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker-node-1
```

**Local PV Considerations:**
- **Node Binding**: Pods using local PVs are scheduled to specific nodes
- **High Performance**: Direct access to local storage
- **No Replication**: Data lost if node fails
- **Use Cases**: Databases, caching layers requiring high IOPS

---

## Persistent Volume Claims (PVCs)

### What is a Persistent Volume Claim?

A **Persistent Volume Claim (PVC)** is a request for storage by a user. It's similar to how pods consume node resources, but PVCs consume PV resources.

### PVC Workflow
1. **Request**: User creates PVC specifying storage requirements
2. **Binding**: Kubernetes finds matching PV and binds them
3. **Mounting**: Pod references PVC and mounts the storage
4. **Usage**: Application reads/writes data to persistent storage
5. **Cleanup**: PVC deletion triggers reclaim policy

### PVC Specification

#### Complete PVC Example
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-app-storage
  namespace: default
  labels:
    app: web-app
    tier: frontend
  annotations:
    volume.beta.kubernetes.io/storage-provisioner: kubernetes.io/aws-ebs
spec:
  # Access modes
  accessModes:
  - ReadWriteOnce
  
  # Volume mode
  volumeMode: Filesystem
  
  # Resource requirements
  resources:
    requests:
      storage: 20Gi
    # limits:           # Optional storage limits
    #   storage: 50Gi
  
  # Storage class
  storageClassName: fast-ssd
  
  # Selector (for static binding)
  selector:
    matchLabels:
      storage-tier: fast
    matchExpressions:
    - key: environment
      operator: In
      values: [production, staging]
  
  # Data source (for cloning/restoring)
  dataSource:
    kind: PersistentVolumeClaim
    name: source-pvc
  
  # Data source ref (for cross-namespace cloning)
  # dataSourceRef:
  #   kind: VolumeSnapshot
  #   name: snapshot-123
  #   apiGroup: snapshot.storage.k8s.io

status:
  # System-populated status
  phase: Bound
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 20Gi
```

### PVC States
- **Pending**: PVC created, waiting for matching PV
- **Bound**: PVC successfully bound to PV
- **Lost**: Bound PV no longer exists

### Using PVCs in Pods

#### Volume Mount Example
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app-pod
spec:
  containers:
  - name: web-app
    image: nginx:1.21
    ports:
    - containerPort: 80
    volumeMounts:
    - name: web-data
      mountPath: /usr/share/nginx/html
    - name: logs
      mountPath: /var/log/nginx
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
  
  volumes:
  - name: web-data
    persistentVolumeClaim:
      claimName: web-app-storage
  - name: logs
    persistentVolumeClaim:
      claimName: web-app-logs
```

#### StatefulSet with PVC Templates
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database-cluster
spec:
  serviceName: database
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:13
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: database-data
          mountPath: /var/lib/postgresql/data
        - name: database-config
          mountPath: /etc/postgresql
        resources:
          requests:
            memory: "512Mi"
            cpu: "200m"
          limits:
            memory: "1Gi"
            cpu: "500m"
  
  # VolumeClaimTemplates create PVCs for each pod
  volumeClaimTemplates:
  - metadata:
      name: database-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
  - metadata:
      name: database-config
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 1Gi
```

### PVC Expansion
```yaml
# PVC with expansion enabled
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: expandable-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi    # Initial size
  storageClassName: expandable-storage
```

```bash
# Expand PVC storage
kubectl patch pvc expandable-pvc -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# Check expansion status
kubectl describe pvc expandable-pvc
kubectl get events --field-selector involvedObject.name=expandable-pvc
```

---

## Storage Classes

### What is a Storage Class?

A **Storage Class** provides a way for administrators to describe different "classes" of storage they offer. Different classes might map to quality-of-service levels, backup policies, or arbitrary policies determined by cluster administrators.

### Storage Class Benefits
- **Dynamic Provisioning**: Automatically create PVs on demand
- **Storage Abstraction**: Hide complex storage configuration
- **Policy Management**: Consistent storage policies across cluster
- **Multi-tenancy**: Different storage tiers for different teams

### Storage Class Specification

#### Complete Storage Class Example
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
  labels:
    storage-tier: premium
    performance: high
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3                    # EBS volume type
  iops: "3000"                 # Provisioned IOPS
  throughput: "125"            # Throughput in MiB/s
  encrypted: "true"            # Enable encryption
  kmsKeyId: "arn:aws:kms:us-west-2:123456789012:key/12345678-1234-1234-1234-123456789012"
reclaimPolicy: Delete          # Delete PV when PVC is deleted
allowVolumeExpansion: true     # Allow PVC expansion
volumeBindingMode: WaitForFirstConsumer  # Delay binding until pod scheduling
mountOptions:                  # Mount options for all PVs
- debug
- discard
```

### Key Storage Class Fields

#### 1. Provisioner
Specifies which volume plugin to use for provisioning PVs:
```yaml
# AWS EBS CSI Driver
provisioner: ebs.csi.aws.com

# GCP Persistent Disk CSI Driver  
provisioner: pd.csi.storage.gke.io

# Azure Disk CSI Driver
provisioner: disk.csi.azure.com

# NFS CSI Driver
provisioner: nfs.csi.k8s.io

# Local Path Provisioner
provisioner: rancher.io/local-path
```

#### 2. Parameters
Provider-specific parameters for volume creation:
```yaml
# AWS EBS Parameters
parameters:
  type: gp3              # Volume type (gp2, gp3, io1, io2)
  iops: "3000"          # Provisioned IOPS
  throughput: "125"     # Throughput (gp3 only)
  encrypted: "true"     # Enable encryption
  fsType: ext4          # Filesystem type
  
# GCP Persistent Disk Parameters
parameters:
  type: pd-ssd          # Disk type (pd-standard, pd-ssd, pd-extreme)
  zones: us-central1-a,us-central1-b
  replication-type: regional-pd
```

#### 3. Volume Binding Mode
Controls when volume binding occurs:

**Immediate Binding:**
```yaml
volumeBindingMode: Immediate
# Volume binding happens immediately when PVC is created
# May cause scheduling constraints
```

**WaitForFirstConsumer:**
```yaml
volumeBindingMode: WaitForFirstConsumer
# Volume binding delayed until pod using PVC is created
# Ensures volume is created in same zone as pod
```

### Cloud Provider Storage Classes

#### AWS EBS Storage Classes
```yaml
# General Purpose SSD (gp3)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
# Provisioned IOPS SSD (io2)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-io2
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iops: "1000"
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
# Cold HDD (sc1)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-cold-hdd
provisioner: ebs.csi.aws.com
parameters:
  type: sc1
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

#### GCP Persistent Disk Storage Classes
```yaml
# Standard Persistent Disk
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcp-standard
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-standard
  replication-type: none
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
# SSD Persistent Disk
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcp-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: none
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
# Regional Persistent Disk
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcp-regional-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

#### Azure Disk Storage Classes
```yaml
# Premium SSD
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-premium
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  cachingmode: ReadOnly
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
# Standard HDD
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-standard
provisioner: disk.csi.azure.com
parameters:
  skuName: Standard_LRS
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### Default Storage Class
```yaml
# Set default storage class
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: default-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
reclaimPolicy: Delete
allowVolumeExpansion: true
```

```bash
# Manage default storage class
kubectl get storageclass
kubectl patch storageclass <old-default> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
kubectl patch storageclass <new-default> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

---

## Dynamic Provisioning

### How Dynamic Provisioning Works

Dynamic provisioning automatically creates storage resources when users create PVCs, eliminating the need for administrators to pre-provision storage.

#### Dynamic Provisioning Flow
```
1. User creates PVC with storageClassName
                ↓
2. Kubernetes finds matching StorageClass
                ↓
3. StorageClass provisioner creates PV
                ↓
4. PVC binds to newly created PV
                ↓
5. Pod mounts the volume and uses storage
```

### Dynamic Provisioning Example

#### Storage Class for Dynamic Provisioning
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: dynamic-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

#### PVC Using Dynamic Provisioning
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: dynamic-ssd  # References storage class
```

#### Pod Using Dynamically Provisioned Storage
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-dynamic-storage
spec:
  containers:
  - name: app
    image: nginx:1.21
    volumeMounts:
    - name: data-volume
      mountPath: /data
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: dynamic-pvc
```

### Multi-Storage Class Strategy
```yaml
# Performance tiers
---
# Tier 1: High Performance
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: tier1-high-performance
  labels:
    tier: "1"
    performance: high
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iops: "5000"
  throughput: "250"
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true

---
# Tier 2: Balanced
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: tier2-balanced
  labels:
    tier: "2"
    performance: balanced
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true

---
# Tier 3: Cost-Effective
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: tier3-cost-effective
  labels:
    tier: "3"
    performance: standard
provisioner: ebs.csi.aws.com
parameters:
  type: gp2
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true

---
# Tier 4: Archive
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: tier4-archive
  labels:
    tier: "4"
    performance: archive
provisioner: ebs.csi.aws.com
parameters:
  type: sc1  # Cold HDD
  encrypted: "true"
reclaimPolicy: Retain
allowVolumeExpansion: false
```

### Topology-Aware Provisioning
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: regional-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
  zones: us-central1-a,us-central1-b,us-central1-c
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: topology.gke.io/zone
    values:
    - us-central1-a
    - us-central1-b
    - us-central1-c
```

---

## Volume Types and CSI Drivers

### Container Storage Interface (CSI)

CSI is a standard for exposing arbitrary block and file storage systems to containerized workloads on Container Orchestration Systems like Kubernetes.

#### CSI Architecture
```
┌─────────────────────────────────────────┐
│            Kubernetes                   │
│      (Master & Worker Nodes)           │
└────────────────┬────────────────────────┘
                 │ CSI API Calls
┌────────────────▼────────────────────────┐
│           CSI Driver                    │
│   ┌─────────────────┬─────────────────┐ │
│   │ Controller      │   Node          │ │
│   │ Plugin          │   Plugin        │ │
│   └─────────────────┴─────────────────┘ │
└────────────────┬────────────────────────┘
                 │ Storage API Calls
┌────────────────▼────────────────────────┐
│         Storage System                  │
│   (AWS EBS, GCP PD, Azure Disk,        │
│    NFS, Ceph, etc.)                    │
└─────────────────────────────────────────┘
```

### Popular CSI Drivers

#### 1. AWS EBS CSI Driver
```yaml
# Install AWS EBS CSI Driver
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: csi-driver
    app.kubernetes.io/name: aws-ebs-csi-driver
  name: ebs-csi-controller-sa
  namespace: kube-system

---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

#### 2. NFS CSI Driver
```yaml
# NFS Storage Class
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.example.com
  share: /srv/nfs
  # subdir: ${pvc.metadata.namespace}/${pvc.metadata.name}
mountOptions:
  - nfsvers=4.1
  - proto=tcp
  - hard
  - intr
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

#### 3. Local Path Provisioner
```yaml
# Local storage class
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete

---
# ConfigMap for local path provisioner
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-path-config
  namespace: local-path-storage
data:
  config.json: |-
    {
      "nodePathMap":[
        {
          "node":"DEFAULT_PATH_FOR_NON_LISTED_NODES",
          "paths":["/opt/local-path-provisioner"]
        }
      ]
    }
  helperPod.yaml: |-
    apiVersion: v1
    kind: Pod
    metadata:
      name: helper-pod
    spec:
      containers:
      - name: helper-pod
        image: busybox
        imagePullPolicy: IfNotPresent
```

### Volume Snapshots

#### VolumeSnapshotClass
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Delete
parameters:
  tagSpecification_1: "Name=*"
  tagSpecification_2: "Environment=production"
```

#### Creating Volume Snapshots
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: database-snapshot
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: database-pvc
```

#### Restoring from Snapshot
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-database-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: aws-ebs-gp3
  resources:
    requests:
      storage: 100Gi
  dataSource:
    name: database-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

### Volume Cloning
```yaml
# Clone existing PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cloned-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: aws-ebs-gp3
  resources:
    requests:
      storage: 50Gi
  dataSource:
    kind: PersistentVolumeClaim
    name: source-pvc
```

---

## Storage Best Practices

### Performance Optimization

#### 1. Choose Appropriate Storage Types
```yaml
# High IOPS for databases
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: database-storage
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iops: "10000"
  throughput: "1000"
reclaimPolicy: Retain
allowVolumeExpansion: true

# Throughput optimized for analytics
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: analytics-storage
provisioner: ebs.csi.aws.com
parameters:
  type: st1  # Throughput Optimized HDD
reclaimPolicy: Delete
allowVolumeExpansion: true
```

#### 2. Use Appropriate Access Modes
```yaml
# Single writer workload
spec:
  accessModes:
  - ReadWriteOnce    # Best performance for single pod

# Shared read-only configuration
spec:
  accessModes:
  - ReadOnlyMany     # Multiple pods reading shared data

# Shared writeable storage (NFS required)
spec:
  accessModes:
  - ReadWriteMany    # Multiple pods writing (limited storage types)
```

#### 3. Resource Requests and Limits
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: storage-optimized-app
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:
        memory: "1Gi"
        cpu: "500m"
        ephemeral-storage: "2Gi"    # Temp storage request
      limits:
        memory: "2Gi"
        cpu: "1"
        ephemeral-storage: "4Gi"    # Temp storage limit
    volumeMounts:
    - name: persistent-data
      mountPath: /data
      # Optional: mount options for performance
  volumes:
  - name: persistent-data
    persistentVolumeClaim:
      claimName: high-performance-pvc
```

### Security Best Practices

#### 1. Encryption at Rest
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: encrypted-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:us-west-2:123456789012:key/12345678-1234-1234-1234-123456789012"
reclaimPolicy: Delete
allowVolumeExpansion: true
```

#### 2. Pod Security Context
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-storage-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000              # Set filesystem group ownership
    fsGroupChangePolicy: Always # Change ownership of mounted volumes
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:1.21
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: [ALL]
    volumeMounts:
    - name: data
      mountPath: /data
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: secure-pvc
  - name: tmp
    emptyDir: {}
```

#### 3. Network Policies for Storage Access
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: storage-access-policy
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
  egress:
  - to: []  # Allow DNS
    ports:
    - protocol: UDP
      port: 53
```

### Capacity Management

#### 1. Resource Quotas
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: development
spec:
  hard:
    requests.storage: "100Gi"          # Total storage requests
    persistentvolumeclaims: "10"       # Number of PVCs
    count/storageclass.storage.k8s.io/fast-ssd: "5"  # PVCs using specific storage class
```

#### 2. Limit Ranges
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: storage-limit-range
  namespace: development
spec:
  limits:
  - type: PersistentVolumeClaim
    max:
      storage: "50Gi"                  # Maximum PVC size
    min:
      storage: "1Gi"                   # Minimum PVC size
    default:
      storage: "10Gi"                  # Default PVC size
    defaultRequest:
      storage: "5Gi"                   # Default PVC request
```

#### 3. Monitoring Storage Usage
```bash
# Monitor storage usage
kubectl top nodes --show-capacity
kubectl get pv -o custom-columns="NAME:.metadata.name,CAPACITY:.spec.capacity.storage,STATUS:.status.phase,CLAIM:.spec.claimRef.name"
kubectl get pvc --all-namespaces -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,STATUS:.status.phase,CAPACITY:.status.capacity.storage"

# Check storage class usage
kubectl get pvc --all-namespaces -o json | jq '.items[] | {name: .metadata.name, namespace: .metadata.namespace, storageClass: .spec.storageClassName, capacity: .status.capacity.storage}'
```

---

## Backup and Disaster Recovery

### Backup Strategies

#### 1. Application-Consistent Backups
For databases and stateful applications, ensure backups are application-consistent:

```yaml
# Pre-backup job to flush database
apiVersion: batch/v1
kind: Job
metadata:
  name: db-pre-backup
spec:
  template:
    spec:
      containers:
      - name: backup-prep
        image: postgres:13
        command:
        - /bin/bash
        - -c
        - |
          # Connect to database and flush
          export PGPASSWORD=$DB_PASSWORD
          psql -h $DB_HOST -U $DB_USER -d $DB_NAME -c "CHECKPOINT;"
          psql -h $DB_HOST -U $DB_USER -d $DB_NAME -c "SELECT pg_start_backup('backup', true);"
        env:
        - name: DB_HOST
          value: postgres-service
        - name: DB_USER
          value: postgres
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: DB_NAME
          value: myapp
      restartPolicy: OnFailure
```

#### 2. Velero Backup Configuration
```yaml
# Velero backup schedule
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  template:
    metadata:
      labels:
        backup-type: scheduled
    spec:
      # Include specific namespaces
      includedNamespaces:
      - production
      - staging
      
      # Exclude system namespaces
      excludedNamespaces:
      - kube-system
      - kube-public
      - velero
      
      # Include specific resources
      includedResources:
      - persistentvolumes
      - persistentvolumeclaims
      - secrets
      - configmaps
      
      # Storage location
      storageLocation: aws-backup-location
      
      # Snapshot volumes
      snapshotVolumes: true
      
      # TTL for backup retention
      ttl: 720h  # 30 days
      
      # Hooks for application-consistent backups
      hooks:
        resources:
        - name: database-backup-hook
          includedNamespaces:
          - production
          labelSelector:
            matchLabels:
              app: postgres
          pre:
          - exec:
              command:
              - /bin/bash
              - -c
              - "pg_start_backup('velero-backup')"
              container: postgres
          post:
          - exec:
              command:
              - /bin/bash
              - -c
              - "pg_stop_backup()"
              container: postgres
```

### Disaster Recovery Planning

#### 1. Recovery Time Objective (RTO) Strategy
```yaml
# Fast recovery with regional replication
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: disaster-recovery-storage
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd  # Regional replication for DR
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

#### 2. Cross-Region Backup Strategy
```bash
# Backup to multiple regions
velero backup create full-cluster-backup \
  --storage-location primary-region \
  --volume-snapshot-locations primary-region,secondary-region \
  --include-cluster-resources \
  --wait

# Restore in different region
velero restore create restore-$(date +%Y%m%d-%H%M%S) \
  --from-backup full-cluster-backup \
  --restore-volumes \
  --wait
```

#### 3. Multi-Cloud Backup Strategy
```yaml
# Backup to multiple cloud providers
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: aws-backup-location
spec:
  provider: aws
  objectStorage:
    bucket: kubernetes-backup-aws
    region: us-west-2
  config:
    s3ForcePathStyle: "false"

---
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: gcp-backup-location
spec:
  provider: gcp
  objectStorage:
    bucket: kubernetes-backup-gcp
    region: us-central1
  config:
    serviceAccount: velero@project-id.iam.gserviceaccount.com
```

### Backup Validation and Testing

#### 1. Backup Verification Job
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-verification
spec:
  schedule: "0 6 * * 1"  # Weekly on Monday at 6 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup-verifier
            image: velero/velero:latest
            command:
            - /bin/bash
            - -c
            - |
              # List recent backups
              velero backup get --output json > /tmp/backups.json
              
              # Check backup status
              if jq -e '.items[] | select(.status.phase != "Completed")' /tmp/backups.json; then
                echo "Found failed backups!"
                exit 1
              fi
              
              # Verify backup content
              LATEST_BACKUP=$(jq -r '.items | sort_by(.metadata.creationTimestamp) | .[-1].metadata.name' /tmp/backups.json)
              velero backup describe $LATEST_BACKUP --details
              
              echo "All backups verified successfully"
          restartPolicy: OnFailure
```

#### 2. Disaster Recovery Testing
```bash
#!/bin/bash
# DR testing script

# 1. Create test backup
velero backup create dr-test-$(date +%Y%m%d) \
  --include-namespaces test-app \
  --wait

# 2. Delete test namespace
kubectl delete namespace test-app

# 3. Restore from backup
velero restore create dr-test-restore-$(date +%Y%m%d) \
  --from-backup dr-test-$(date +%Y%m%d) \
  --wait

# 4. Verify restoration
kubectl get all -n test-app
kubectl wait --for=condition=Ready pod -l app=test-app -n test-app --timeout=300s

# 5. Run application tests
kubectl exec -n test-app deployment/test-app -- /app/health-check.sh

echo "DR test completed successfully"
```

---

## Practice Labs

### Lab 1: Static Storage Provisioning

#### Objective
Create and manage persistent volumes and claims manually.

#### Setup Static PVs
```yaml
# Static PV using hostPath (for testing)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: static-pv-1
  labels:
    type: local
    tier: standard
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /tmp/data/pv-1

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: static-pv-2
  labels:
    type: local
    tier: premium
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  - ReadOnlyMany
  persistentVolumeReclaimPolicy: Delete
  storageClassName: manual
  hostPath:
    path: /tmp/data/pv-2
```

#### Create PVCs
```yaml
# PVC requesting standard storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: standard-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
  storageClassName: manual
  selector:
    matchLabels:
      tier: standard

---
# PVC requesting premium storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: premium-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
  storageClassName: manual
  selector:
    matchLabels:
      tier: premium
```

#### Testing Commands
```bash
# Apply static PVs and PVCs
kubectl apply -f static-storage.yaml

# Check PV status
kubectl get pv
kubectl describe pv static-pv-1

# Check PVC binding
kubectl get pvc
kubectl describe pvc standard-pvc

# Create test pods
kubectl run pv-test-1 --image=busybox --rm -it --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"pv-test","image":"busybox","volumeMounts":[{"name":"storage","mountPath":"/data"}],"command":["sleep","3600"]}],"volumes":[{"name":"storage","persistentVolumeClaim":{"claimName":"standard-pvc"}}]}}'

# Test storage inside pod
kubectl exec pv-test-1 -- df -h /data
kubectl exec pv-test-1 -- echo "test data" > /data/test.txt
kubectl exec pv-test-1 -- cat /data/test.txt

# Verify PV status
kubectl get pv -o custom-columns="NAME:.metadata.name,STATUS:.status.phase,CLAIM:.spec.claimRef.name,SIZE:.spec.capacity.storage"
```

### Lab 2: Dynamic Storage Provisioning

#### Objective
Configure and test dynamic storage provisioning with storage classes.

#### Create Storage Classes
```yaml
# Fast SSD storage class
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: rancher.io/local-path  # Use local-path for testing
parameters:
  pathPattern: "/opt/local-path-provisioner/fast-ssd"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
# Standard storage class
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
parameters:
  pathPattern: "/opt/local-path-provisioner/standard"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
# Archive storage class
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: archive
provisioner: rancher.io/local-path
parameters:
  pathPattern: "/opt/local-path-provisioner/archive"
reclaimPolicy: Retain
allowVolumeExpansion: false
volumeBindingMode: Immediate
```

#### Create Dynamic PVCs
```yaml
# Fast storage PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fast-storage-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast-ssd

---
# Default storage PVC (uses default storage class)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: default-storage-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  # No storageClassName specified = uses default

---
# Archive storage PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: archive-storage-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: archive
```

#### Test Application
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: storage-test-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: storage-test
  template:
    metadata:
      labels:
        app: storage-test
    spec:
      containers:
      - name: app
        image: nginx:1.21
        ports:
        - containerPort: 80
        volumeMounts:
        - name: fast-storage
          mountPath: /fast-data
        - name: default-storage
          mountPath: /data
        - name: archive-storage
          mountPath: /archive
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
      volumes:
      - name: fast-storage
        persistentVolumeClaim:
          claimName: fast-storage-pvc
      - name: default-storage
        persistentVolumeClaim:
          claimName: default-storage-pvc
      - name: archive-storage
        persistentVolumeClaim:
          claimName: archive-storage-pvc
```

#### Testing Commands
```bash
# Install local-path provisioner if not available
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.24/deploy/local-path-storage.yaml

# Apply storage classes and PVCs
kubectl apply -f storage-classes.yaml
kubectl apply -f dynamic-pvcs.yaml

# Check storage classes
kubectl get storageclass
kubectl describe storageclass standard

# Wait for PVCs to be provisioned
kubectl get pvc -w

# Deploy test application
kubectl apply -f storage-test-app.yaml

# Wait for pods to be ready
kubectl wait --for=condition=Ready pod -l app=storage-test --timeout=120s

# Test storage access
kubectl exec deployment/storage-test-app -- df -h
kubectl exec deployment/storage-test-app -- ls -la /fast-data /data /archive

# Test volume expansion
kubectl patch pvc fast-storage-pvc -p '{"spec":{"resources":{"requests":{"storage":"30Gi"}}}}'
kubectl describe pvc fast-storage-pvc

# Monitor expansion
kubectl get events --field-selector involvedObject.name=fast-storage-pvc
```

### Lab 3: StatefulSet with Storage

#### Objective
Deploy a StatefulSet with persistent storage using volumeClaimTemplates.

#### StatefulSet with Persistent Storage
```yaml
# Storage class for StatefulSet
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: stateful-storage
provisioner: rancher.io/local-path
parameters:
  pathPattern: "/opt/local-path-provisioner/stateful"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
# PostgreSQL StatefulSet
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
data:
  password: cG9zdGdyZXM=  # postgres

---
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432

---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless
  replicas: 3
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
        image: postgres:13
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: POSTGRES_REPLICATION_MODE
          value: master
        - name: POSTGRES_REPLICATION_USER
          value: replicator
        - name: POSTGRES_REPLICATION_PASSWORD
          value: replicatorpassword
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        - name: postgres-config
          mountPath: /etc/postgresql
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "300m"
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 15
          periodSeconds: 10
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 30
          periodSeconds: 10
  
  # Volume claim templates
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: stateful-storage
      resources:
        requests:
          storage: 20Gi
  - metadata:
      name: postgres-config
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: stateful-storage
      resources:
        requests:
          storage: 1Gi
```

#### Testing Commands
```bash
# Apply StatefulSet
kubectl apply -f postgres-statefulset.yaml

# Monitor StatefulSet deployment
kubectl get statefulsets -w
kubectl get pods -l app=postgres -w

# Check PVCs created for each pod
kubectl get pvc
kubectl get pvc -l app=postgres

# Verify each pod has unique storage
kubectl exec postgres-0 -- df -h /var/lib/postgresql/data
kubectl exec postgres-1 -- df -h /var/lib/postgresql/data
kubectl exec postgres-2 -- df -h /var/lib/postgresql/data

# Test database in each instance
kubectl exec postgres-0 -- psql -U postgres -c "CREATE TABLE test_data(id SERIAL, data TEXT);"
kubectl exec postgres-0 -- psql -U postgres -c "INSERT INTO test_data(data) VALUES ('data from postgres-0');"

kubectl exec postgres-1 -- psql -U postgres -c "CREATE TABLE test_data(id SERIAL, data TEXT);"
kubectl exec postgres-1 -- psql -U postgres -c "INSERT INTO test_data(data) VALUES ('data from postgres-1');"

# Verify data persistence by deleting and recreating pods
kubectl delete pod postgres-0
kubectl wait --for=condition=Ready pod postgres-0 --timeout=120s
kubectl exec postgres-0 -- psql -U postgres -c "SELECT * FROM test_data;"

# Scale StatefulSet
kubectl scale statefulset postgres --replicas=5
kubectl get pods -l app=postgres
kubectl get pvc

# Scale down
kubectl scale statefulset postgres --replicas=2
kubectl get pods -l app=postgres
kubectl get pvc  # PVCs remain for potential scale-up
```

---

## Hands-on Projects

### Project 1: Multi-Tier Application with Comprehensive Storage Strategy

#### Objective
Deploy a complete e-commerce application with different storage tiers and backup strategies.

#### Architecture
```
┌─────────────────────────────────────────────────┐
│                Load Balancer                    │
└─────────────────┬───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│              Frontend (3 replicas)              │
│         • Static assets (ReadOnlyMany)          │
│         • Logs (emptyDir)                       │
└─────────────────┬───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│            Backend API (2 replicas)             │
│         • Config files (ConfigMap)              │
│         • Upload storage (ReadWriteMany)        │
│         • Cache (emptyDir)                      │
└─────────────────┬───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│         Database Cluster (3 replicas)           │
│         • Primary data (high IOPS)              │
│         • WAL logs (fast SSD)                   │
│         • Backups (archive storage)             │
└─────────────────────────────────────────────────┘
```

#### Implementation

**Step 1: Storage Classes Definition**
```yaml
# High-performance storage for database primary data
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: high-performance-ssd
  labels:
    performance-tier: "1"
provisioner: rancher.io/local-path
parameters:
  pathPattern: "/opt/local-path-provisioner/high-perf"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
# Fast storage for WAL logs and caches
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  labels:
    performance-tier: "2"
provisioner: rancher.io/local-path
parameters:
  pathPattern: "/opt/local-path-provisioner/fast"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
# Standard storage for application data
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-storage
  labels:
    performance-tier: "3"
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
parameters:
  pathPattern: "/opt/local-path-provisioner/standard"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
# Archive storage for backups and logs
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: archive-storage
  labels:
    performance-tier: "4"
provisioner: rancher.io/local-path
parameters:
  pathPattern: "/opt/local-path-provisioner/archive"
reclaimPolicy: Retain
allowVolumeExpansion: false
volumeBindingMode: Immediate

---
# Shared storage for multi-pod access
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: shared-storage
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.default.svc.cluster.local
  share: /shared
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
```

**Step 2: Database Layer with Multiple Storage Types**
```yaml
# Database secrets
apiVersion: v1
kind: Secret
metadata:
  name: database-secret
  namespace: ecommerce
type: Opaque
data:
  postgres-password: c2VjdXJlcGFzc3dvcmQ=
  replication-password: cmVwbGljYXRvcg==

---
# Database configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: database-config
  namespace: ecommerce
data:
  postgresql.conf: |
    # PostgreSQL configuration for high performance
    shared_buffers = 256MB
    effective_cache_size = 1GB
    maintenance_work_mem = 64MB
    checkpoint_completion_target = 0.9
    wal_buffers = 16MB
    default_statistics_target = 100
    random_page_cost = 1.1
    effective_io_concurrency = 200
    
    # WAL configuration
    wal_level = replica
    archive_mode = on
    archive_command = 'cp %p /archive/%f'
    max_wal_senders = 3
    wal_keep_segments = 32
  
  pg_hba.conf: |
    # Database access configuration
    local   all             all                                     trust
    host    all             all             127.0.0.1/32            md5
    host    all             all             ::1/128                 md5
    host    replication     all             0.0.0.0/0               md5

---
# Database StatefulSet with multiple storage types
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-cluster
  namespace: ecommerce
spec:
  serviceName: postgres-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
        tier: database
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: postgres-password
        - name: POSTGRES_REPLICATION_USER
          value: replicator
        - name: POSTGRES_REPLICATION_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: replication-password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        # Primary data on high-performance storage
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        # WAL logs on fast SSD
        - name: postgres-wal
          mountPath: /var/lib/postgresql/wal
        # Configuration files
        - name: postgres-config
          mountPath: /etc/postgresql
        # Archive storage for backups
        - name: postgres-archive
          mountPath: /archive
        resources:
          requests:
            memory: "512Mi"
            cpu: "200m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 15
          periodSeconds: 10
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 30
          periodSeconds: 10
      
      # Configuration volume
      volumes:
      - name: postgres-config
        configMap:
          name: database-config
  
  volumeClaimTemplates:
  # High-performance storage for main data
  - metadata:
      name: postgres-data
      annotations:
        volume.beta.kubernetes.io/storage-class: high-performance-ssd
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: high-performance-ssd
      resources:
        requests:
          storage: 100Gi
  
  # Fast SSD for WAL logs
  - metadata:
      name: postgres-wal
      annotations:
        volume.beta.kubernetes.io/storage-class: fast-ssd
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 20Gi
  
  # Archive storage for backups
  - metadata:
      name: postgres-archive
      annotations:
        volume.beta.kubernetes.io/storage-class: archive-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: archive-storage
      resources:
        requests:
          storage: 200Gi

---
# Database services
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: ecommerce
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432

---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: ecommerce
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

**Step 3: Backend with Shared Storage**
```yaml
# Backend application with shared upload storage
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: ecommerce
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend-api
  template:
    metadata:
      labels:
        app: backend-api
        tier: backend
    spec:
      containers:
      - name: api
        image: node:18-alpine
        command: ['sh', '-c']
        args:
        - |
          cat > server.js << 'EOF'
          const express = require('express');
          const multer = require('multer');
          const path = require('path');
          const fs = require('fs');
          
          const app = express();
          const port = 3000;
          
          // Configure multer for file uploads
          const storage = multer.diskStorage({
            destination: '/shared/uploads/',
            filename: (req, file, cb) => {
              cb(null, Date.now() + '-' + file.originalname);
            }
          });
          const upload = multer({ storage: storage });
          
          app.use(express.json());
          app.use('/uploads', express.static('/shared/uploads'));
          
          app.get('/health', (req, res) => {
            res.json({ status: 'healthy', pod: process.env.HOSTNAME });
          });
          
          app.post('/upload', upload.single('file'), (req, res) => {
            res.json({
              message: 'File uploaded successfully',
              filename: req.file.filename,
              pod: process.env.HOSTNAME
            });
          });
          
          app.get('/files', (req, res) => {
            fs.readdir('/shared/uploads', (err, files) => {
              if (err) {
                return res.status(500).json({ error: err.message });
              }
              res.json({ files, pod: process.env.HOSTNAME });
            });
          });
          
          app.listen(port, () => {
            console.log(`Backend API running on port ${port}`);
          });
          EOF
          
          npm init -y
          npm install express multer
          mkdir -p /shared/uploads
          node server.js
        ports:
        - containerPort: 3000
        volumeMounts:
        # Shared storage for file uploads
        - name: shared-uploads
          mountPath: /shared/uploads
        # Local cache storage
        - name: cache-storage
          mountPath: /cache
        # Configuration
        - name: app-config
          mountPath: /etc/config
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "300m"
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
      
      volumes:
      # Shared storage for uploads (accessible by all backend pods)
      - name: shared-uploads
        persistentVolumeClaim:
          claimName: shared-uploads-pvc
      # Local cache (per-pod)
      - name: cache-storage
        emptyDir:
          sizeLimit: 1Gi
      # Configuration
      - name: app-config
        configMap:
          name: backend-config

---
# Shared storage PVC for uploads
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-uploads-pvc
  namespace: ecommerce
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  storageClassName: shared-storage

---
# Backend configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: ecommerce
data:
  app.json: |
    {
      "database": {
        "host": "postgres-service",
        "port": 5432,
        "database": "ecommerce"
      },
      "upload": {
        "maxFileSize": "10MB",
        "allowedTypes": ["image/jpeg", "image/png", "image/gif"]
      }
    }

---
# Backend service
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: ecommerce
spec:
  selector:
    app: backend-api
  ports:
  - port: 3000
    targetPort: 3000
```

**Step 4: Frontend with Static Assets**
```yaml
# Static assets PVC (ReadOnlyMany for shared access)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: static-assets-pvc
  namespace: ecommerce
spec:
  accessModes:
  - ReadOnlyMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard-storage

---
# Job to populate static assets
apiVersion: batch/v1
kind: Job
metadata:
  name: populate-static-assets
  namespace: ecommerce
spec:
  template:
    spec:
      containers:
      - name: asset-loader
        image: busybox
        command: ['sh', '-c']
        args:
        - |
          mkdir -p /assets/css /assets/js /assets/images
          
          # Create sample static assets
          cat > /assets/index.html << 'EOF'
          <!DOCTYPE html>
          <html>
          <head>
            <title>E-Commerce App</title>
            <link rel="stylesheet" href="/css/style.css">
          </head>
          <body>
            <h1>Welcome to E-Commerce</h1>
            <div id="app"></div>
            <script src="/js/app.js"></script>
          </body>
          </html>
          EOF
          
          cat > /assets/css/style.css << 'EOF'
          body { font-family: Arial, sans-serif; margin: 0; padding: 20px; }
          h1 { color: #333; }
          EOF
          
          cat > /assets/js/app.js << 'EOF'
          console.log('E-Commerce app loaded');
          EOF
          
          echo "Static assets populated successfully"
        volumeMounts:
        - name: assets
          mountPath: /assets
      restartPolicy: OnFailure
      volumes:
      - name: assets
        persistentVolumeClaim:
          claimName: static-assets-pvc

---
# Frontend deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: ecommerce
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: frontend
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: frontend
              topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        volumeMounts:
        # Static assets (shared read-only)
        - name: static-assets
          mountPath: /usr/share/nginx/html
          readOnly: true
        # Nginx configuration
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
        # Logs (per-pod)
        - name: nginx-logs
          mountPath: /var/log/nginx
        resources:
          requests:
            memory: "128Mi"
            cpu: "50m"
          limits:
            memory: "256Mi"
            cpu: "100m"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
      
      volumes:
      # Shared static assets
      - name: static-assets
        persistentVolumeClaim:
          claimName: static-assets-pvc
      # Nginx configuration
      - name: nginx-config
        configMap:
          name: nginx-config
      # Local log storage
      - name: nginx-logs
        emptyDir:
          sizeLimit: 500Mi
```

**Step 5: Backup and Monitoring Setup**
```yaml
# Backup CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-backup
  namespace: ecommerce
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:15
            command:
            - /bin/bash
            - -c
            - |
              # Database backup
              export PGPASSWORD=$POSTGRES_PASSWORD
              BACKUP_FILE="/archive/backup-$(date +%Y%m%d-%H%M%S).sql"
              
              echo "Starting database backup..."
              pg_dump -h postgres-service -U postgres -d ecommerce > $BACKUP_FILE
              
              if [ -s $BACKUP_FILE ]; then
                echo "Backup completed successfully: $BACKUP_FILE"
                # Compress backup
                gzip $BACKUP_FILE
                
                # Clean old backups (keep last 30 days)
                find /archive -name "backup-*.sql.gz" -mtime +30 -delete
                
                echo "Backup retention cleanup completed"
              else
                echo "Backup failed - file is empty"
                exit 1
              fi
            env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: database-secret
                  key: postgres-password
            volumeMounts:
            - name: backup-storage
              mountPath: /archive
            resources:
              requests:
                memory: "256Mi"
                cpu: "100m"
              limits:
                memory: "512Mi"
                cpu: "200m"
          restartPolicy: OnFailure
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-storage-pvc

---
# Backup storage PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: backup-storage-pvc
  namespace: ecommerce
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Gi
  storageClassName: archive-storage

---
# Storage monitoring job
apiVersion: batch/v1
kind: CronJob
metadata:
  name: storage-monitoring
  namespace: ecommerce
spec:
  schedule: "*/15 * * * *"  # Every 15 minutes
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: monitor
            image: busybox
            command:
            - /bin/sh
            - -c
            - |
              echo "=== Storage Usage Report $(date) ==="
              echo "Database data:"
              du -sh /var/lib/postgresql/data/* 2>/dev/null || echo "No database data"
              echo "Upload storage:"
              du -sh /shared/uploads/* 2>/dev/null || echo "No uploads"
              echo "Archive storage:"
              du -sh /archive/* 2>/dev/null || echo "No archives"
              echo "Static assets:"
              du -sh /assets/* 2>/dev/null || echo "No static assets"
              echo "=== End Report ==="
            volumeMounts:
            - name: database-data
              mountPath: /var/lib/postgresql/data
              readOnly: true
            - name: shared-uploads
              mountPath: /shared/uploads
              readOnly: true
            - name: archive-storage
              mountPath: /archive
              readOnly: true
            - name: static-assets
              mountPath: /assets
              readOnly: true
          restartPolicy: OnFailure
          volumes:
          - name: database-data
            persistentVolumeClaim:
              claimName: postgres-data-postgres-cluster-0
              readOnly: true
          - name: shared-uploads
            persistentVolumeClaim:
              claimName: shared-uploads-pvc
              readOnly: true
          - name: archive-storage
            persistentVolumeClaim:
              claimName: backup-storage-pvc
              readOnly: true
          - name: static-assets
            persistentVolumeClaim:
              claimName: static-assets-pvc
              readOnly: true
```

#### Deployment and Testing
```bash
# 1. Create namespace
kubectl create namespace ecommerce

# 2. Deploy storage classes
kubectl apply -f storage-classes.yaml

# 3. Deploy database layer
kubectl apply -f database-layer.yaml

# 4. Wait for database to be ready
kubectl wait --for=condition=Ready pod -l app=postgres -n ecommerce --timeout=300s

# 5. Populate static assets
kubectl apply -f populate-assets-job.yaml
kubectl wait --for=condition=Complete job/populate-static-assets -n ecommerce --timeout=120s

# 6. Deploy backend
kubectl apply -f backend-layer.yaml

# 7. Deploy frontend
kubectl apply -f frontend-layer.yaml

# 8. Deploy backup and monitoring
kubectl apply -f backup-monitoring.yaml

# 9. Wait for all deployments
kubectl wait --for=condition=Available deployment -l tier=backend -n ecommerce --timeout=300s
kubectl wait --for=condition=Available deployment -l tier=frontend -n ecommerce --timeout=300s

# 10. Test the application
kubectl port-forward -n ecommerce svc/backend-service 3000:3000 &
kubectl port-forward -n ecommerce svc/frontend-service 8080:80 &

# Test file upload
curl -X POST -F "file=@test.txt" http://localhost:3000/upload

# Test file listing
curl http://localhost:3000/files

# Test frontend
curl http://localhost:8080

# 11. Check storage usage
kubectl get pvc -n ecommerce
kubectl describe pvc -n ecommerce

# 12. Check backup execution
kubectl get cronjobs -n ecommerce
kubectl get jobs -n ecommerce

# 13. Monitor storage across all tiers
kubectl logs -n ecommerce -l job-name=storage-monitoring
```

---

## Troubleshooting Guide

### PV/PVC Issues

#### 1. PVC Stuck in Pending State

**Symptoms:**
```bash
kubectl get pvc
# NAME        STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# my-pvc      Pending                                      fast-ssd       5m
```

**Diagnosis:**
```bash
# Check PVC events
kubectl describe pvc my-pvc
kubectl get events --field-selector involvedObject.name=my-pvc

# Common issues to check:
# 1. No matching PV available
kubectl get pv

# 2. Storage class doesn't exist or is misconfigured
kubectl get storageclass
kubectl describe storageclass fast-ssd

# 3. Insufficient permissions for dynamic provisioning
kubectl get pods -n kube-system | grep provisioner
kubectl logs -n kube-system -l app=csi-controller

# 4. Node constraints not met
kubectl describe nodes
```

**Solutions:**
```bash
# Create matching static PV (if needed)
kubectl apply -f matching-pv.yaml

# Check storage class configuration
kubectl describe storageclass fast-ssd

# Verify provisioner permissions
kubectl get clusterrole system:csi-external-provisioner
kubectl get clusterrolebinding | grep provisioner
```

#### 2. Volume Mount Failures

**Symptoms:**
```bash
kubectl get pods
# NAME      READY   STATUS                  RESTARTS   AGE
# my-pod    0/1     ContainerCreating       0          2m
```

**Diagnosis:**
```bash
# Check pod events
kubectl describe pod my-pod

# Common mount errors:
# - Volume not formatted
# - Permission issues  
# - Node doesn't have required drivers

# Check node where pod is scheduled
kubectl get pod my-pod -o wide
kubectl describe node <node-name>

# Check CSI driver pods
kubectl get pods -n kube-system | grep csi
kubectl logs -n kube-system <csi-node-pod>
```

### Storage Performance Issues

#### 1. Slow I/O Performance

**Diagnosis:**
```bash
# Test I/O performance inside pod
kubectl exec -it <pod-name> -- bash

# Inside pod:
# Test sequential write performance
dd if=/dev/zero of=/data/testfile bs=1M count=1000 conv=fsync

# Test random I/O (install fio)
fio --name=random-rw --ioengine=libaio --iodepth=4 --rw=randrw --bs=4k --size=1G --filename=/data/test

# Check filesystem type and mount options
mount | grep /data
```

**Solutions:**
```yaml
# Use appropriate storage class for workload
# High IOPS for databases
storageClassName: high-iops-ssd

# Add performance mount options
mountOptions:
- discard
- noatime
```

#### 2. Storage Capacity Issues

**Monitoring:**
```bash
# Check PVC usage
kubectl get pvc --all-namespaces -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,STATUS:.status.phase,CAPACITY:.status.capacity.storage,USED:.status.usage.used"

# Check node storage capacity
kubectl top nodes --show-capacity

# Monitor storage usage over time
kubectl exec -it <pod-name> -- df -h
```

### Dynamic Provisioning Issues

#### 1. Provisioner Not Working

**Diagnosis:**
```bash
# Check provisioner pods
kubectl get pods -n kube-system | grep provisioner
kubectl get pods -n kube-system | grep csi

# Check provisioner logs
kubectl logs -n kube-system -l app=csi-external-provisioner
kubectl logs -n kube-system -l app=csi-provisioner

# Check storage class configuration
kubectl describe storageclass <storage-class-name>
```

#### 2. Cloud Provider Issues

**AWS EBS Issues:**
```bash
# Check AWS CSI driver
kubectl get pods -n kube-system | grep ebs-csi

# Check IAM permissions
kubectl describe pod -n kube-system <ebs-csi-controller-pod>

# Check AWS credentials
kubectl get secret -n kube-system aws-secret
```

**GCP PD Issues:**
```bash
# Check GCP CSI driver
kubectl get pods -n kube-system | grep pd-csi

# Check service account
kubectl describe serviceaccount -n kube-system <gcp-compute-persistent-disk-csi-driver>
```

### StatefulSet Storage Issues

#### 1. Volume Claim Template Problems

**Diagnosis:**
```bash
# Check StatefulSet status
kubectl get statefulsets
kubectl describe statefulset <statefulset-name>

# Check PVCs created by StatefulSet
kubectl get pvc -l app=<statefulset-app>

# Check pod startup order
kubectl get pods -l app=<statefulset-app> --sort-by=.metadata.name
```

#### 2. Volume Expansion Issues

**Diagnosis:**
```bash
# Check if storage class supports expansion
kubectl describe storageclass <storage-class-name>

# Check PVC expansion status
kubectl describe pvc <pvc-name>

# Check filesystem expansion
kubectl exec -it <pod-name> -- df -h
kubectl exec -it <pod-name> -- lsblk
```

**Manual Expansion:**
```bash
# Expand PVC
kubectl patch pvc <pvc-name> -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'

# If filesystem doesn't expand automatically
kubectl exec -it <pod-name> -- resize2fs /dev/<device>
# or for XFS
kubectl exec -it <pod-name> -- xfs_growfs /mount/point
```

---

This comprehensive guide covers all aspects of Kubernetes storage management. Master these concepts to effectively manage persistent storage, implement proper backup strategies, and troubleshoot storage issues in your Kubernetes clusters.