# MongoDB in Kubernetes

This repository describes architectural approaches for running MongoDB with Kubernetes and demonstrates a production architecture with **controlled storage topology**.

The goal is to explain engineering trade-offs and operational considerations when running databases inside Kubernetes — not to be a tutorial on installation.

---

## Why This Repository Exists

Running MongoDB in Kubernetes is widely discussed,
but many examples focus only on deployment mechanics.

This repository focuses on **storage topology and operational stability**.

It demonstrates a production-oriented architecture where:

- Kubernetes manages Pods
- storage topology remains predictable
- MongoDB ReplicaSet handles node failures
- storage is not dynamically relocated by the orchestrator

The goal is to show a **practical compromise between Kubernetes flexibility and database stability**.

---

## Design Goals

This architecture is designed with the following priorities:

1. **Predictable storage topology**

   MongoDB data must remain attached to known nodes.

2. **Operational simplicity during incidents**

   Engineers must immediately understand where data is located.

3. **Controlled failover**

   MongoDB ReplicaSet should handle node failures,
   while storage placement remains deterministic.

4. **Minimal reliance on cloud storage orchestration**

   The system should not depend on slow or unpredictable
   attach/detach operations.

5. **Clear separation of responsibilities**

   Kubernetes manages **process lifecycle**,
   operators manage **storage topology**.

---

> **Key Engineering Principle**
>
> MongoDB handles node failures well.
> However, MongoDB does **not** handle unpredictable storage relocation well.
>
> MongoDB ReplicaSets are designed to handle **node failures**,
> but they assume that **storage location remains stable**.

Because of this, running MongoDB in Kubernetes often requires controlling **storage topology**, not only Pod orchestration.

---

## MongoDB + Kubernetes: Possible Architectures

There are several ways MongoDB can be used together with Kubernetes.
Each approach has its own advantages, limitations, and appropriate use cases.

| Scenario | Implementation Complexity | Storage Reliability | Operational Predictability | Scaling Flexibility |
|---|---|---|---|---|
| MongoDB outside Kubernetes | Medium | Very High | Very High | Low |
| Single MongoDB instance in Kubernetes | Low | Low | Medium | None |
| MongoDB ReplicaSet with dynamic storage | Medium | Medium | Low | High |
| MongoDB ReplicaSet with controlled storage | High | High | High | Medium |

---

## Reality of Production Systems

Most production incidents involving MongoDB in Kubernetes do not originate in the application layer.
They occur at the intersection of storage, scheduling, and node failure: a volume that cannot reattach, a Pod rescheduled to an unexpected node, or a disk left mounted on a host that never recovered.
MongoDB replication handles node unavailability — but it cannot compensate for a volume that is physically inaccessible or attached to the wrong host.
Understanding where the orchestrator's responsibility ends and the database's begins is the central operational challenge this architecture addresses.

---

### 1. MongoDB Outside Kubernetes

In this architecture MongoDB runs on separate servers or virtual machines.
The Kubernetes cluster connects to the database over the network.

```mermaid
flowchart LR
    A[Kubernetes Cluster] --> B[MongoDB Servers]
```

## Pros

* fully independent storage infrastructure
* predictable storage behavior
* simpler backup and recovery strategies
* less dependency on Kubernetes scheduling
* easier operational management

## Cons

* separate infrastructure
* separate database lifecycle
* additional network hop between application and database
* possible increase in query latency

## Use Cases

This is **the most common and recommended approach for production workloads**, even when applications run inside Kubernetes.

It works best when:

* the database is critical
* storage predictability is important
* infrastructure is separated into **application layer (Kubernetes)** and **database layer (dedicated servers)**
* Kubernetes is used primarily for **stateless workloads**
* the team wants to avoid turning Kubernetes into a **fully stateful platform**

Kubernetes works best when it remains **lightweight and elastic**.

When a cluster starts hosting large stateful systems (especially databases), operations often become:

* more complex
* more rigid
* less predictable

For this reason many production systems keep the database **outside Kubernetes**, even when all applications run inside the cluster.

---

### 2. Single MongoDB Instance in Kubernetes

MongoDB runs as a single Pod inside Kubernetes.

```mermaid
flowchart TD
    A[Mongo Pod] --> B[PVC]
    B --> C[Storage]
```

## Pros

* extremely simple architecture
* very quick deployment
* convenient for development and testing

## Cons

* no high availability
* Pod failure makes the database unavailable
* potential risk of data loss
* not suitable for production workloads

## Use Cases

Typically used for:

* development environments
* staging environments
* testing setups
* small internal tools

---

### 3. MongoDB ReplicaSet with Dynamic Storage

MongoDB runs as a StatefulSet.
PersistentVolumeClaims are created automatically through a StorageClass.

```mermaid
flowchart LR
    A[Mongo-0] --> B[PVC] --> C[Dynamic PV]
    D[Mongo-1] --> E[PVC] --> F[Dynamic PV]
    G[Mongo-2] --> H[PVC] --> I[Dynamic PV]
```

## Pros

* Kubernetes automatically manages storage
* less manual configuration
* fast infrastructure deployment
* easy environment creation
* convenient horizontal scaling

## Cons

* reduced control over data placement
* volumes may be attached to unexpected nodes
* attach/detach operations can take significant time
* node failures may create complex recovery scenarios

In real clusters situations may occur where:

* a node fails
* the Pod is rescheduled to another node
* the volume cannot be reattached quickly
* the Pod enters **CrashLoopBackOff**

These issues complicate database operations.

A specific and well-known failure mode is the **Multi-Attach error**:

* a node becomes unavailable
* the disk remains attached to the old node at the cloud/storage level
* Kubernetes attempts to schedule the Pod on a new node
* the disk cannot be mounted because it is still associated with the previous node
* the Pod stays in **ContainerCreating** state indefinitely, waiting for the volume to detach

This situation requires manual intervention and can result in significant downtime for the affected ReplicaSet member.

Overall stability of this architecture heavily depends on the quality and reliability of the CSI driver provided by the underlying platform (e.g. AWS EBS, Google Persistent Disk, Azure Disk).

## Use Cases

This approach is used when **speed and infrastructure flexibility are more important than storage determinism**.

It works best when:

* environments need to be created quickly
* infrastructure is frequently recreated
* systems scale both up and down frequently
* the team accepts the risk of storage-related issues

The operational philosophy here is often:

```
something breaks → recreate pod → recreate storage → continue operating
```

This model prioritizes **speed of infrastructure** over **predictability of storage topology**.

---

### 4. MongoDB ReplicaSet with Controlled Storage

MongoDB runs as a StatefulSet, but storage is managed manually.

Static PersistentVolumes are created in advance and Pod placement is controlled.

```mermaid
flowchart LR
    A[worker-1] --> B[Mongo-0] --> C[PV-1]
    D[worker-2] --> E[Mongo-1] --> F[PV-2]
    G[worker-3] --> H[Mongo-2] --> I[PV-3]
```

```mermaid
flowchart TB

subgraph Kubernetes Cluster
    Pod1["MongoDB Pod (mongo-0)"]
    Pod2["MongoDB Pod (mongo-1)"]
    Pod3["MongoDB Pod (mongo-2)"]
end

subgraph Persistent Storage Layer
    PVC1["PersistentVolumeClaim"]
    PVC2["PersistentVolumeClaim"]
    PVC3["PersistentVolumeClaim"]
end

subgraph Node Storage
    PV1["PersistentVolume → node-1 disk"]
    PV2["PersistentVolume → node-2 disk"]
    PV3["PersistentVolume → node-3 disk"]
end

Pod1 --> PVC1 --> PV1
Pod2 --> PVC2 --> PV2
Pod3 --> PVC3 --> PV3
```

## Architecture Characteristics

* each Pod uses a pre-created PersistentVolume
* each volume is bound to a specific node
* Pod placement is controlled using node affinity
* storage topology is fully predictable

## Pros

* full control over data placement
* deterministic storage topology
* easier troubleshooting
* faster recovery during incidents
* clear understanding of where data physically resides

## Cons

* more manual configuration
* scaling requires preparing new volumes
* higher operational responsibility

## Use Cases

This architecture is a **compromise between two worlds**:

* running the database **inside Kubernetes**
* maintaining **strict control over storage placement**

It is appropriate when:

* the database must run inside Kubernetes
* the team needs to know exactly **where data is stored**
* incident investigation must be straightforward
* recovery operations must be predictable
* the team does not want to rely entirely on dynamic storage systems

The core design principle is:

```
Pod → specific PVC → specific PV → specific Node
```

Kubernetes manages the **process lifecycle**,
while storage topology remains **explicitly controlled by the operator**.

---

# Repository Structure

Each environment contains all resources required to run a MongoDB ReplicaSet inside Kubernetes.

```text
k8s/
├── ENV/
│   ├── dev/
│   │   ├── mongo-statefulset.yaml
│   │   ├── mongo-headless-svc.yaml
│   │   └── mongo-client-svc.yaml
│   │
│   ├── stage/
│   │   ├── mongo-statefulset.yaml
│   │   ├── mongo-headless-svc.yaml
│   │   └── mongo-client-svc.yaml
│   │
│   ├── prod-1/
│   │   ├── mongo-statefulset.yaml
│   │   ├── mongo-headless-svc.yaml
│   │   └── mongo-client-svc.yaml
│   │
│   └── prod-2/
│       ├── mongo-statefulset.yaml
│       ├── mongo-headless-svc.yaml
│       └── mongo-client-svc.yaml
│
└── PV/
    ├── pv-dev-mongo-worker-1.yaml
    ├── pv-dev-mongo-worker-2.yaml
    ├── pv-dev-mongo-worker-3.yaml
    │
    ├── pv-stage-mongo-worker-1.yaml
    ├── pv-stage-mongo-worker-2.yaml
    ├── pv-stage-mongo-worker-3.yaml
    │
    ├── pv-prod-1-mongo-worker-1.yaml
    ├── pv-prod-1-mongo-worker-2.yaml
    ├── pv-prod-1-mongo-worker-3.yaml
    │
    ├── pv-prod-2-mongo-worker-1.yaml
    ├── pv-prod-2-mongo-worker-2.yaml
    └── pv-prod-2-mongo-worker-3.yaml
```

Each environment runs a MongoDB ReplicaSet using a StatefulSet.

* The **Headless Service** is used for internal ReplicaSet communication.
* The **Client Service** provides stable access for applications inside the cluster.
* PersistentVolumes are defined separately and pinned to specific nodes.

This separation allows Kubernetes to manage Pod lifecycle while keeping database storage topology predictable.

### Cluster Assumptions

This architecture assumes a Kubernetes cluster with **at least three worker nodes**.

MongoDB ReplicaSet consists of three Pods:

- `mongo-0`
- `mongo-1`
- `mongo-2`

Each Pod is scheduled on a **different worker node**.

To guarantee stable data placement, each MongoDB Pod uses a **dedicated storage volume** mapped through Kubernetes storage objects:

```
node-1 → storage → PV → PVC → mongo-0
node-2 → storage → PV → PVC → mongo-1
node-3 → storage → PV → PVC → mongo-2
```

This results in the following rule:

**1 storage → 1 PersistentVolume → 1 PersistentVolumeClaim → 1 MongoDB Pod**

This mapping guarantees deterministic data placement across cluster nodes.

This design prevents shared-storage issues and keeps MongoDB data placement stable even when Pods are restarted or rescheduled.
