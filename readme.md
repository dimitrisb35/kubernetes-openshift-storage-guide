# Kubernetes & OpenShift Storage Guide

## Overview

Kubernetes doesn't store application data itself — it integrates with storage systems via CSI (Container Storage Interface) drivers and abstracts them into **PersistentVolumes (PV)** and **PersistentVolumeClaims (PVC)**. This guide covers the main storage types, their use cases, and how they're implemented in both Kubernetes and OpenShift.

---

## Storage Architecture

### Core Concepts

**CSI (Container Storage Interface)**
- Standard interface for storage systems to integrate with Kubernetes
- Enables vendors to develop storage plugins independently
- Supports dynamic provisioning and volume snapshots

**PersistentVolumes (PV)**
- Cluster-level storage resources provisioned by administrators or dynamically
- Independent lifecycle from pods
- Backed by physical storage systems

**PersistentVolumeClaims (PVC)**
- User requests for storage
- Pods consume PVs through PVCs
- Abstract storage details from application developers

**StorageClasses**
- Define different storage tiers and backends
- Enable dynamic provisioning
- Specify provisioner, parameters, and reclaim policies

---

## Storage Types

### 1. Block Storage

**What it is:**
- Raw block devices similar to virtual disks or LUNs
- Mounted to a pod as a volume
- Pod's filesystem manages it (ext4, xfs, etc.)
- Direct access to storage blocks

**How it works:**
```
Application → Filesystem (ext4/xfs) → Block Device → Physical Storage
```

**Use Cases:**
- Databases (PostgreSQL, MySQL, MongoDB, Oracle)
- Message queues (Kafka, RabbitMQ)
- Any stateful application requiring fast, low-latency access
- Applications needing direct block-level I/O

**Kubernetes Examples:**
- AWS EBS (Elastic Block Store)
- Azure Disk
- GCP Persistent Disk
- Local PersistentVolumes
- iSCSI storage

**OpenShift Examples:**
- Ceph RBD (ocs-storagecluster-ceph-rbd)
- AWS EBS on AWS clusters
- Azure Disk on Azure clusters
- VMware vSphere storage

**Example PVC:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ocs-storagecluster-ceph-rbd
  resources:
    requests:
      storage: 50Gi
```

**Pros:**
- High performance and low latency
- Excellent for database workloads
- Direct block-level access
- Predictable I/O performance

**Cons:**
- Tied to a single pod/node at a time (ReadWriteOnce)
- Cannot be shared across multiple pods
- More complex to manage than file storage

---

### 2. File Storage

**What it is:**
- Shared filesystem accessible by multiple pods simultaneously
- Uses protocols like NFS, CephFS, or GlusterFS
- POSIX-compliant filesystem interface
- Mounted as a directory in pods

**How it works:**
```
Pod A ─┐
Pod B ─┼→ Shared Filesystem → Storage Backend
Pod C ─┘
```

**Use Cases:**
- Shared configuration and secrets
- Legacy applications requiring shared directories
- CI/CD systems storing build artifacts
- Content management systems
- Development environments with shared code

**Kubernetes Examples:**
- NFS (Network File System)
- Azure Files
- AWS EFS (Elastic File System)
- GCP Filestore
- GlusterFS

**OpenShift Examples:**
- CephFS (ocs-storagecluster-cephfs)
- NFS CSI driver
- NetApp Trident

**Example PVC:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-storage-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ocs-storagecluster-cephfs
  resources:
    requests:
      storage: 100Gi
```

**Pros:**
- Shared access across multiple pods (ReadWriteMany)
- Easy to use and understand
- POSIX-compliant for legacy applications
- Simplifies data sharing between services

**Cons:**
- Lower performance compared to block storage
- Network overhead for distributed filesystems
- Not optimal for database workloads
- Potential consistency issues with concurrent writes

---

### 3. Object Storage

**What it is:**
- Data stored as objects with metadata
- Accessed via HTTP/S APIs (not mounted as volumes)
- Uses protocols like S3, Swift, or Azure Blob
- Flat namespace with bucket/container structure

**How it works:**
```
Application → S3/Swift API → Object Storage → Distributed Storage
```

**Use Cases:**
- Backups and archives
- AI/ML datasets and model storage
- Application logs and metrics
- Images, videos, and media files
- Static website content
- Data lakes and analytics

**Kubernetes Examples:**
- AWS S3
- Google Cloud Storage (GCS)
- Azure Blob Storage
- MinIO (self-hosted S3-compatible)
- Ceph Object Gateway (RGW)

**OpenShift Examples:**
- Ceph RGW (S3-compatible via OpenShift Data Foundation)
- AWS S3 on AWS clusters
- Azure Blob Storage on Azure clusters
- NooBaa Multi-Cloud Gateway (part of ODF)

**Example Usage (not a PVC - uses API):**
```python
import boto3

s3 = boto3.client('s3',
    endpoint_url='https://s3.openshift.example.com',
    aws_access_key_id='ACCESS_KEY',
    aws_secret_access_key='SECRET_KEY')

s3.upload_file('backup.tar.gz', 'my-bucket', 'backups/backup.tar.gz')
```

**Pros:**
- Scales massively (petabytes+)
- Excellent for unstructured data
- Built-in redundancy and durability
- Cost-effective for cold storage
- Multi-region replication

**Cons:**
- Higher latency than block storage
- Not POSIX-compliant (cannot mount as filesystem)
- API-based access only
- Eventual consistency in some implementations

---

### 4. Ephemeral Storage

**What it is:**
- Temporary storage that exists only for the pod's lifetime
- Comes from the node's local disk
- Data is lost when the pod is deleted or restarted
- No persistent state

**How it works:**
```
Pod Creation → Storage Allocated from Node → Pod Deletion → Storage Deleted
```

**Use Cases:**
- Caching and temporary data
- Scratch space for processing
- Build artifacts during CI/CD
- Session data in stateless applications
- Temporary file uploads

**Kubernetes Examples:**
- emptyDir volumes
- ConfigMaps (small data)
- Secrets (small sensitive data)
- Ephemeral CSI volumes

**OpenShift Examples:**
- Same as Kubernetes (emptyDir, ephemeral CSI)
- Local node storage

**Example Configuration:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cache-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: cache
      mountPath: /cache
  volumes:
  - name: cache
    emptyDir:
      sizeLimit: 1Gi
```

**Pros:**
- Simple to configure
- Fast access (local to node)
- No external dependencies
- No cleanup required

**Cons:**
- Data loss on pod restart
- Not suitable for stateful applications
- Limited by node's local storage
- Cannot be shared across nodes

---

## Access Modes Comparison

| Access Mode | Abbreviation | Description | Storage Types |
|-------------|--------------|-------------|---------------|
| ReadWriteOnce | RWO | Single node mount read-write | Block, File |
| ReadOnlyMany | ROX | Multiple nodes mount read-only | File |
| ReadWriteMany | RWX | Multiple nodes mount read-write | File |
| ReadWriteOncePod | RWOP | Single pod mount read-write | Block (K8s 1.22+) |

---

## Storage Selection Guide

### By Use Case

| Use Case | Recommended Storage | Why |
|----------|-------------------|-----|
| **Database (PostgreSQL, MySQL)** | Block (RBD, EBS, Azure Disk) | Low latency, high IOPS, consistent performance |
| **Shared Configuration** | File (CephFS, NFS, EFS) | Multiple pods need access, easy to manage |
| **Message Queue (Kafka, RabbitMQ)** | Block (RBD, EBS) | High throughput, low latency required |
| **CI/CD Artifacts** | File (CephFS, NFS) or Object (S3) | Shared access or long-term storage |
| **Backup/Archive** | Object (S3, RGW, Blob) | Cost-effective, massive scale, durability |
| **AI/ML Datasets** | Object (S3, GCS) | Large files, API access, cost-effective |
| **Container Images** | Object (S3, Registry) | Immutable, distributed access |
| **Application Logs** | Object (S3, Blob) | Long-term retention, analytics |
| **Session Cache** | Ephemeral (emptyDir) | Temporary, performance-critical |
| **Media Files (Images, Videos)** | Object (S3, Blob) | Large files, CDN integration |
| **Shared Code Repository** | File (NFS, CephFS) | Developer access, version control |

### By Performance Requirements

| Requirement | Storage Type | Examples |
|-------------|-------------|----------|
| **Ultra-low latency (<1ms)** | Block (Local SSD) | Local PV, NVMe |
| **Low latency (<10ms)** | Block (Network) | Ceph RBD, EBS, Azure Disk |
| **Moderate latency (<50ms)** | File (Distributed) | CephFS, NFS, EFS |
| **High latency OK (>100ms)** | Object (S3-compatible) | S3, RGW, Blob Storage |

---

## Kubernetes vs OpenShift Storage Comparison

### Native Kubernetes Storage Options

| Storage Type | Technology | Access Mode | Use Case |
|--------------|-----------|-------------|----------|
| **Block** | AWS EBS | RWO | Databases on AWS |
| **Block** | Azure Disk | RWO | Databases on Azure |
| **Block** | GCP Persistent Disk | RWO | Databases on GCP |
| **Block** | Local PersistentVolume | RWO | High-performance local storage |
| **File** | AWS EFS | RWX | Shared storage on AWS |
| **File** | Azure Files | RWX | Shared storage on Azure |
| **File** | NFS | RWX | Legacy shared storage |
| **Object** | AWS S3 | API | Backups, archives, media |
| **Object** | GCS | API | ML datasets, backups |
| **Ephemeral** | emptyDir | - | Caching, temp data |

### OpenShift Data Foundation (ODF) Storage

| Storage Type | ODF Component | Protocol | Access Mode | Use Case |
|--------------|--------------|----------|-------------|----------|
| **Block** | Ceph RBD | iSCSI/RBD | RWO | Databases, VMs |
| **File** | CephFS | CephFS | RWX | Shared workloads |
| **Object** | Ceph RGW | S3/Swift | API | Backups, archives, AI/ML |
| **Object** | NooBaa MCG | S3 | API | Multi-cloud object storage |

### StorageClass Examples

**Kubernetes (AWS EBS):**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
volumeBindingMode: WaitForFirstConsumer
```

**OpenShift Data Foundation (Ceph RBD):**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ocs-storagecluster-ceph-rbd
provisioner: openshift-storage.rbd.csi.ceph.com
parameters:
  clusterID: openshift-storage
  pool: ocs-storagecluster-cephblockpool
  imageFeatures: layering
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

---

## Red Hat OpenShift Data Foundation (ODF)

### Overview

ODF is the integrated storage layer for OpenShift, built on Ceph technology. It provides a unified storage platform supporting block, file, and object storage.

**Key Features:**
- Multi-cloud and hybrid cloud support
- Built-in data replication and high availability
- Snapshot and clone capabilities
- Encryption at rest and in transit
- Multi-tenancy with namespace isolation

### ODF Architecture

```
┌─────────────────────────────────────────────┐
│        OpenShift Data Foundation            │
├─────────────────────────────────────────────┤
│  Block (RBD)  │  File (CephFS)  │  Object   │
│               │                 │  (RGW/MCG)│
├─────────────────────────────────────────────┤
│              Ceph Storage Cluster           │
├─────────────────────────────────────────────┤
│         OSD Nodes (Storage Nodes)           │
└─────────────────────────────────────────────┘
```

### Disaster Recovery with ODF

**Regional DR (Metro DR):**
- Synchronous replication within metro distances (<100km)
- RPO = 0 (no data loss)
- RTO = minutes
- Use case: High availability across data centers

**Multi-site DR:**
- Asynchronous replication for longer distances
- RPO = seconds to minutes
- RTO = minutes to hours
- Use case: Geographic disaster recovery

**Implementation:**
```yaml
apiVersion: ramendr.openshift.io/v1alpha1
kind: DRPolicy
metadata:
  name: odf-dr-policy
spec:
  drClusters:
  - ocp-primary
  - ocp-secondary
  schedulingInterval: 5m
  replicationClassSelector:
    matchLabels:
      replication: enabled
```

---

## Performance Comparison

| Storage Type | Latency | Throughput | IOPS | Scalability | Cost |
|--------------|---------|------------|------|-------------|------|
| **Block (Local SSD)** | <1ms | Very High | 100K+ | Node-limited | High |
| **Block (Network)** | 1-10ms | High | 10K-50K | Medium | Medium |
| **File (Distributed)** | 5-50ms | Medium | 1K-10K | High | Medium |
| **Object (S3)** | 50-200ms | High | Varies | Very High | Low |
| **Ephemeral** | <1ms | Very High | 100K+ | Node-limited | Low |

---

## Best Practices

### Storage Selection

1. **Match storage to workload requirements**
   - Use block storage for databases
   - Use file storage for shared access
   - Use object storage for archives and backups

2. **Consider access patterns**
   - Single-writer workloads: Block (RWO)
   - Multiple-reader workloads: File (RWX)
   - API-accessed data: Object (S3)

3. **Plan for growth**
   - Use dynamic provisioning
   - Set appropriate storage quotas
   - Monitor capacity regularly

### Performance Optimization

1. **Use appropriate StorageClass parameters**
   - IOPS and throughput settings for cloud storage
   - Replication settings for distributed storage

2. **Co-locate storage with compute when possible**
   - Use topology-aware provisioning
   - Consider node affinity for performance-critical workloads

3. **Monitor storage metrics**
   - IOPS, latency, throughput
   - Capacity utilization
   - Storage class performance

### Data Protection

1. **Implement backup strategies**
   - Volume snapshots for quick recovery
   - External backups for disaster recovery
   - Test restore procedures regularly

2. **Use appropriate replication**
   - Synchronous for zero data loss
   - Asynchronous for geographic DR

3. **Enable encryption**
   - Encryption at rest for sensitive data
   - Encryption in transit for network security

---

## Troubleshooting Common Issues

### PVC Stuck in Pending

**Check:**
```bash
kubectl describe pvc <pvc-name>
kubectl get events --sort-by='.lastTimestamp'
```

**Common causes:**
- No available PVs matching the claim
- StorageClass doesn't exist
- Insufficient storage capacity
- Node selector/affinity constraints

### Performance Issues

**Check:**
```bash
# View storage metrics
kubectl top nodes
kubectl describe storageclass <storage-class>

# Check pod I/O
kubectl exec -it <pod-name> -- iostat -x 1
```

**Common causes:**
- Storage backend overloaded
- Network bottlenecks
- Incorrect storage type for workload
- Resource limits too restrictive

### Data Loss

**Prevention:**
- Use replicated storage (Ceph, cloud storage)
- Regular backups and snapshots
- Test disaster recovery procedures
- Use StatefulSets for stateful applications

---

## Additional Resources

- [Kubernetes Storage Documentation](https://kubernetes.io/docs/concepts/storage/)
- [CSI Driver Specification](https://github.com/container-storage-interface/spec)
- [OpenShift Data Foundation Documentation](https://access.redhat.com/documentation/en-us/red_hat_openshift_data_foundation/)
- [Ceph Documentation](https://docs.ceph.com/)
- [Storage Performance Benchmarking](https://github.com/kubernetes/perf-tests/tree/master/fio-test)
