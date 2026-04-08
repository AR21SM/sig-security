# Etcd \- Technical scope of the assessment 

## **Table of Contents**

1. **[Metadata](#metadata)**

2. **[Overview](#overview)**
* [Impact](#impact)
* [Scope](#scope)
* [Process level](#process-level)
* [Technical](#technical)
* [Not in Scope](#not-in-scope)

3. **[Communication Channels](#communication-channels)**
* [Slack channels in Kubernetes Workspace](#slack-channels-in-kubernetes-workspace)
* [Mailing lists](#mailing-lists)
* [GitHub tracking](#github-tracking)
* [Primary Community Contact](#primary-community-contact)

4. **[Project Overview](#project-overview)**

	4.1 [Project Goals](#project-goals)

	4.2 [Project Non-goals](#project-non-goals)

	4.3 [Intended Uses of the Project](#intended-uses-of-the-project)

	4.4 [Personas](#personas)

	4.5 [Primary Components](#primary-components)

      * [Boundaries and control plane interactions](#boundaries-and-control-plane-interactions)

	4.6 [Data Stores and Data Flows](#data-stores-and-data-flows)

      * [Data Flow diagrams](#data-flow-diagrams)
      * [Common Flows](#common-flows)

	4.7 [3rd Party Requirements](#3rd-party-requirements-source-libraries-services-apis)

	4.8 [Secure development practices](#secure-development-practices)

	4.9 [Development Workflow](#development-workflow)

      * [Identified gaps in development workflow](#identified-gaps-in-development-workflow)
      * [Core Infrastructure Initiative](#core-infrastructure-initiative)

5. **[Threat Modeling with STRIDE](#threat-modeling-with-stride)**

	5.1 [Spoofing](#spoofing)

	5.2 [Tampering](#tampering)

	5.3 [Repudiation](#repudiation)

	5.4 [Information Disclosure](#information-disclosure)

	5.5 [Denial of Service](#denial-of-service)

	5.6 [Elevation of Privilege](#elevation-of-privilege)

	5.7 [Security issue resolution](#security-issue-resolution)

6. **[References](#references)**
	* [Docs](#docs)
	* [Talks](#talks)
	* [Links](#links)

----

## Metadata

| Status |  In progress |
| :---- | :---- |
| Software | [etcd](https://github.com/etcd-io/etcd) |
| Security Provider | NO |
| Languages | Go  |
| SBOM | Link to [go.mod](https://github.com/etcd-io/etcd/blob/main/go.mod) |

## Version History

| Date | Version | Author(s) | Changes |
| :---- | :---- | :---- | :---- |
| 2025-09-21 | 0.1 | Carol Valencia | Initial draft, Created formal Initiative from this version: [https://github.com/kubernetes/sig-security/issues/152](https://github.com/kubernetes/sig-security/issues/152)  |
|  |  | Ronald Ngounou | Added the reference section and updated documentation on etcd peers. |
|  |  | Victor Lu |  |
| 2026-01-21 |  | Tiago Turquette | Stride components |
| 2026-04-08 |  | Ashish Mahajan ||


## Reviewers

| Date | Name | Contact |
| ----- | ----- | ----- |
|  | Benjamin Wang |  |
|  | Sherine Khoury |  |

# Overview

A security [self-assessment](https://github.com/kubernetes/sig-security/tree/main/sig-security-assessments) is a subproject, part of the [kubernetes sig-security](https://github.com/kubernetes/sig-security) initiatives. 

The sig-security team will work with project maintainers to create a threat model and assess the current security instance of all or a portion of the project. The resulting assessment will allow maintainers to determine gaps in their project's security and develop a strategy to work toward improving it.

See all the self assessments write up here: [https://github.com/kubernetes/sig-security/tree/main/sig-security-assessments](https://github.com/kubernetes/sig-security/tree/main/sig-security-assessments)

- [Cluster API Security Self-Assessment](https://github.com/kubernetes/sig-security/blob/main/sig-security-assessments/cluster-api/self-assessment.md)  
- [vSphere CSI Driver Security Self-Assessment](https://github.com/kubernetes/sig-security/blob/main/sig-security-assessments/vsphere-csi-driver/self-assessment.md)

Meetings notes: [Etcd Self Assessment Meeting Notes](https://docs.google.com/document/d/1AMhGlDyeKrMRUYHxMl7s2cA3vcMrYXC0wQJgaq10hW4/edit?tab=t.0#heading=h.z480elyyrn7n)  
Folder: [etcd-self-assesments-cncf](https://drive.google.com/drive/folders/1f4zL8xFjAS8tkOD-CCoGnJ92CMVDhZp6?usp=sharing)

Issue: [https://github.com/kubernetes/sig-security/issues/152](https://github.com/kubernetes/sig-security/issues/152)

Example of a SIG Security self-assessment, final output: [https://github.com/kubernetes/sig-security/blob/main/sig-security-assessments/cluster-api/self-assessment.md](https://github.com/kubernetes/sig-security/blob/main/sig-security-assessments/cluster-api/self-assessment.md)

etcd is a distributed, reliable, and consistent key-value store designed to hold critical configuration data for distributed systems. In Kubernetes, etcd serves as the backing store for all cluster state, including configuration, metadata, and control-plane coordination. It exposes a strongly consistent gRPC-based API and uses the Raft consensus algorithm to maintain data consistency across multiple nodes. etcd supports features such as leader election, watches for change notifications, and transactional operations, enabling components to coordinate reliably in dynamic environments. To ensure availability and durability, etcd is typically deployed as an odd-sized cluster with quorum-based decision-making, secured via mutual TLS authentication, access controls, and optional data-at-rest encryption. Its correctness, performance, and resilience are foundational to the stability and behavior of the Kubernetes control plane.

# Communication Channels

### **Slack channels in Kubernetes Workspace**

Visit [https://slack.k8s.io](https://slack.k8s.io) to request an invite

* [\#sig-security-assess-etcd](https://kubernetes.slack.com/archives/C06KZSTBY1L) \- Working channel for this review

# Project Overview

## Project Goals

* To provide a distributed, reliable key-value store for the most critical data in distributed systems.  
* To ensure strong consistency and durability guarantees through quorum-based consensus.  
* To expose a simple, stable, and well-defined API supporting key-value operations, watches, and transactions.  
* To enable high availability through replicated state, leader election, and fault tolerance.  
* To support secure communication and access control, including transport encryption and authentication.  
* To serve as a reliable backing store for coordination and configuration data, including Kubernetes control plane state.  
* To support safe operational procedures such as cluster membership changes, backup and restore, compaction, and upgrades.

## Project Non-goals

* To serve as a general-purpose or application-facing database  
* To provide eventual consistency or relaxed consistency semantics  
* To automatically manage or orchestrate systems that depend on etcd  
* To replace higher-level systems for messaging, service discovery, or application coordination  
* To provide transparent low-latency, multi-region replication  
* To eliminate the need for operator involvement in sizing, tuning, or disaster recovery planning  
* To support unbounded data growth or sustained high-throughput write workloads

## Personas

### System clients

System clients interact with etcd through the v3 API surface. These clients use clientv3 API semantics, either directly through the clientv3 library or via equivalent language-specific implementations, to store and watch critical system state such as object definitions, configurations, and cluster metadata.

Representative system clients include orchestration and control plane components that rely on etcd as a source of truth.

### Administrative Operators

Administrative operators interact with etcd to perform operational and maintenance tasks.

These actors typically use command-line tools such as etcdctl to conduct health checks, manage cluster membership, perform backups and restorations, and carry out other administrative operations against the etcd cluster.

### Application Developer

Application developers build systems that integrate with etcd as a distributed key-value store.

These actors use etcd client libraries implementing the v3 API to support application-level coordination, configuration management, or service discovery. Application developers do not interact with internal etcd components directly and rely on the exposed API surface for all interactions.

# Primary Components

![][image1]

## Client

Clients include administrative tools such as etcdctl, application and control-plane components using the clientv3 gRPC client library, and other systems that rely on etcd for coordination or state storage.

## API

Exposes etcd functionality to external clients. Provides a stable, versioned API surface implemented over gRPC Remote Procedure Calls (gRPC), and HTTP where applicable. It is responsible for request admission, basic validation, and routing requests into the etcd server core for processing.

## etcd Server Core

Coordinates request processing and integrates the API layer with the consensus and storage subsystems.

It is responsible for orchestrating read and write flows, applying committed state changes, enforcing internal limits, and managing background maintenance activities.

## Raft Consensus

Provides distributed consistency and fault tolerance across the etcd cluster. It ensures that all state changes are agreed upon by a quorum of cluster members before being committed. Raft defines the ordering, replication, and commitment of log entries that represent changes to the key-value store.

### Leader election

Establishes a single active leader responsible for coordinating write operations. Leader election ensures progress in the presence of failures while maintaining safety guarantees defined by the Raft protocol.

### Log replication

Orders state changes as log entries and replicates them from the leader to follower members. Replication ensures that committed entries are durably recorded on a quorum of nodes before being applied to the state machine.

### Membership management

Controls the addition and removal of cluster members using Raft’s membership change mechanisms. This ensures that configuration changes do not violate quorum or consistency guarantees during cluster reconfiguration.

### Read index

Provides a mechanism for serving linearizable read requests by verifying that the serving node is sufficiently up to date with the committed Raft log. This avoids stale reads without requiring all reads to go through the leader’s log.

## MVCC Key-Value Store

The MVCC (Multi-Version Concurrency Control) Key-Value Store defines the logical data model of etcd.

It maintains versioned key-value data, supports transactional semantics, and enables consistent reads and watch semantics by tracking revisions over time. This layer represents etcd’s logical state independent of how it is persisted on disk.

## Persistence and Storage

The Persistence and Storage layer provides durable storage and crash recovery for etcd state.

It ensures that committed state survives process restarts and node failures by persisting Raft logs and snapshots to disk. This layer underpins both Raft and MVCC by providing reliable on-disk storage primitives.

# Define the scope

## What workflows are used most?

* **Kubernetes Resource Management:** The most frequent workflows involve kubectl operations (creating, updating, deleting Pods, Services, Deployments, ConfigMaps, Secrets, etc.) which all funnel through the API Server to etcd.  
* **Controller Operations:** kube-controller-manager and kube-scheduler constantly watch the API Server for changes and write status updates back, constituting a high volume of read-and-write operations.  
* **State Synchronization/Watches:** Components watching the API Server for changes in cluster state.

## What workflows does the community want a security assessment of?

 From a security perspective, the community is most concerned with:

* **Unauthorized Access to etcd:** Preventing direct read/write access to etcd from anything other than authorized kube-apiserver instances.  
* **Data Tampering:** Ensuring the integrity of data stored in etcd against malicious modification.  
* **Data Confidentiality:** Protecting sensitive data (e.g., Secrets) at rest in etcd and in transit.  
* **Denial of Service (DoS):** Protecting etcd from being overwhelmed or brought down, as it's a single point of failure for the Kubernetes control plane.  
* **Backup and Recovery:** Secure and reliable processes for etcd data snapshots and restoration.  
* **Upgrade/Downgrade Resilience:** Ensuring data consistency and security during etcd cluster lifecycle events.

## What expertise does the team doing the assessment have? Does it match any of the potential flows? Try to get matching expertise for the scope\!

* **Distributed Systems & Consensus Algorithms:** Deep understanding of Raft, distributed state management, and fault tolerance. (Matches etcd core functionality, high availability flows).  
* **Kubernetes Architecture & Control Plane:** Intimate knowledge of how Kubernetes components (API Server, controllers, scheduler, kubelet) interact and rely on etcd. (Matches all defined workflows and technical questions).  
* **Network Security (TLS/Authentication/Authorization):** Expertise in securing gRPC communication, mutual TLS, and IAM policies. (Matches API Server \<-\> etcd communication, client access).  
* **Data Security (at Rest and in Transit):** Knowledge of encryption, key management, and data integrity. (Matches etcd storage and replication).  
* **Linux/Container Security:** Understanding of file system permissions, process isolation within containers, and secure base images. (Matches etcd deployment and backend storage).  
* **Performance & Scalability:** While not strictly security, understanding performance implications can reveal attack vectors (e.g., DoS) or configuration weaknesses.

# Technical Questions to Guide the Diagram

* **What operations does the API server perform on etcd?** The API Server performs **CRUD (Create, Read, Update, Delete) operations** on Kubernetes resources (Pods, Services, ConfigMaps, Secrets, Deployments, etc.) and also issues **Watch operations** to subscribe to changes. These operations are typically via gRPC.  
* **How does etcd store Kubernetes state?** Etcd stores Kubernetes state as a **key-value store**. Each Kubernetes object is represented as a key-value pair, typically organized under paths like /registry/pods/default/nginx, /registry/services/default/my-service, /registry/configmaps/default/my-config, etc.  
* **Who queries etcd directly?** In a standard Kubernetes deployment, **only the kube-apiserver directly queries and writes to etcd**. No other Kubernetes component should directly access etcd.  
*  **Is data ever pushed directly to etcd from outside the API Server?** **No**. Access to etcd for Kubernetes state is **strictly via the kube-apiserver**. Any external tool or component attempting direct access to etcd would bypass Kubernetes's authentication, authorization, and admission control layers, which is a major security risk.

* **How is high availability handled (multi-node etcd)?** High availability is handled by running **multiple etcd nodes (typically 3 or 5\)** in a cluster. They use the **Raft consensus algorithm** to elect a leader, replicate data across all members, and ensure that all committed data is consistent across a majority of nodes. If the leader fails, a new one is elected.  
    
* etcd's role and data scope: etcd is the primary data store for all Kubernetes cluster data, including configurations, state, and secrets. Its security is critical because access to etcd is equivalent to root permissions in the cluster.

* Access control and authentication: Evaluate the strictness of mutual TLS (mTLS) authentication between etcd clients (like the Kubernetes API server) and etcd servers. Verify usage of strong, certificate-based authentication to ensure only authorized components can access etcd.  
* Data encryption:  
  * Encryption in transit: All communication between etcd clients and servers and among etcd cluster members must be encrypted using TLS to prevent eavesdropping or man-in-the-middle attacks.  
  * Encryption at rest: Stored etcd data must be encrypted to protect sensitive information, especially Kubernetes Secrets, from unauthorized access if physical storage is compromised.  
* Backup and recovery data flows: Ensure secure handling of etcd backups, including encryption and restricting access to backup data to prevent exfiltration of sensitive cluster state data.

* Audit and logging: Verify audit logs capture etcd access and changes, providing traceability and forensic capability in case of security incidents

# Data Stores and Data flow Diagram (DFD) of etcd in Kubernetes

🎯 **Objective**: Visualize how etcd interacts with Kubernetes components, mainly the API Server, for reads/writes and system state management.

📌 **Scope**:

- **Include:** etcd, kube-apiserver, kube-controller-manager, kube-scheduler, kubectl (user interaction).  
- **Exclude:** Pod-level communication, worker-node internals (for simplicity, only high-level kubelet interaction is shown for node registration).

Understand what data moves where:  
	•	API server → etcd (write/read cluster state)  
	•	etcd → API server (returns data)  
	•	API server ←→ user (via kubectl)  
	•	controller-manager ← API server (watch events)  
	•	scheduler ← API server (unscheduled Pods)  
	•	controller-manager → API server (updates status)

SD: Sequence diagram  
DF: Data flow diagram

## 01-DF: Etcd High availability

![01-DF Etcd High Availability](images/01-Etcd%20High%20availability%20Data-Flow.png)

The data flow diagram showing high availability architecture in Kubernetes with multiple etcd nodes, load balancing, and failover scenarios.

This data flow diagram illustrates etcd high availability in Kubernetes with the following key components and flows:

**High Availability Architecture:**

1. **etcd Cluster (3 nodes minimum), etcd has 3 nodes minimum to guarantee the smallest quorum with tolerance for failure.:**  
   * One Leader node (handles all writes)  
   * Two+ Follower nodes (replicate data)  
   * Uses Raft consensus algorithm  
2. **Multiple API Servers:**  
   * Each apiserver instance can connect to any etcd node  
   * Load balanced for client requests  
   * Provide redundancy for API access

**Key Data Flows:**

**Write Operations:**

* All writes go through the Leader  
* Leader replicates to majority of followers (2/3)  
* Uses Raft consensus for consistency  
* Only commits when majority acknowledges

**Read Operations:**

* Can read from any node (eventual consistency)  
* Leader reads for linearizable consistency  
* API servers can load balance reads

**Failure Scenarios:**

1. **etcd Node Failure:**  
   * If follower fails: Cluster continues with reduced redundancy  
   * If leader fails: New leader elected automatically  
   * Requires majority quorum (2/3) to remain operational  
2. **API Server Failure:**  
   * Load balancer redirects to healthy API servers  
   * Each API server can connect to multiple etcd nodes

**High Availability Features:**

* **Quorum-based decisions** (need majority for writes)  
* **Automatic leader election** via Raft  
* **Data replication** across all nodes  
* **Service discovery** via DNS  
* **Optional proxy/load balancer** for etcd endpoints  
* **Multiple API server instances** for redundancy

**Best Practices Shown:**

* Odd number of etcd nodes (3, 5, 7): quorum uses an odd number of nodes to prevent "split-brain" scenarios where network issues could make the two halves of a cluster think they are the majority. An odd number ensures that one partition always has more than half the votes.Geographic distribution of nodes  
* Multiple API server instances  
* Client-side load balancing capabilities  
* Network redundancy and service discovery

This architecture ensures that the Kubernetes cluster can survive individual component failures while maintaining data consistency and availability.


## 02-DF: Critical etcd access in Kubernetes

![02-DF Critical etcd access in Kubernetes](images/02%20Critical%20etcd%20access%20in%20Kubernetes%20Data-Flow.png)

This comprehensive data flow diagram illustrates the critical security implications of etcd in Kubernetes:

**etcd's Complete Data Scope:**

1. **Configuration Data:**  
   * All cluster configurations and settings  
   * Network policies and feature gates  
   * Component configurations  
2. **Cluster State:**  
   * Real-time status of all resources  
   * Node health and capacity information  
   * Service endpoints and networking  
3. **Sensitive Data:**  
   * All Kubernetes Secrets (passwords, tokens, keys)  
   * TLS certificates and registry credentials  
   * ServiceAccount tokens  
4. **Security Policies:**  
   * RBAC rules and permissions  
   * ClusterRoles and RoleBindings  
   * Admission control policies

**Critical Security Principle: etcd Access \= Root Privileges**

**Why Direct etcd Access is Dangerous:**

* **Bypasses all Kubernetes security layers** (authentication, authorization, admission control)  
* **Exposes all cluster secrets** in potentially unencrypted form  
* **Allows arbitrary resource manipulation** without audit trails  
* **Enables privilege escalation** by modifying RBAC policies  
* **Can completely destroy cluster state** with delete operations

**Attack Scenarios:**

1. **Compromised etcd node** → Complete cluster takeover  
2. **Stolen etcd backup** → All secrets and configurations exposed  
3. **Network access to etcd** → Silent bypass of all Kubernetes security  
4. **Malicious etcd client** → Undetected data manipulation

**Essential Protection Measures:**

1. **Network Security:**  
   * TLS encryption for all communications  
   * Client certificate authentication  
   * Firewall restrictions on etcd ports  
2. **Storage Security:**  
   * Encryption at rest for etcd data  
   * Secure file system permissions  
   * Encrypted backup storage  
3. **Access Control:**  
   * Strict network isolation  
   * Regular security audits  
   * Monitoring and alerting for etcd access

**Key Takeaway:** The API Server acts as the crucial security gateway, providing authentication, authorization, and audit logging that are completely bypassed with direct etcd access. This is why etcd security is paramount \- compromising etcd is equivalent to having unlimited administrative access to the entire Kubernetes cluster.

## 03-DF: Cryptographic Data lifecycle and Integrity

Diagram image: 

This section illustrates **how data is protected from birth to death** and how we make sure it was never altered along the way.

1.  **In-Transit**: Secures data while it is moving across the network to prevent interception.

	

2. **At-Rest**:  Protects data while it is stored on physical disks to prevent unauthorized access to the database files.  
     
   While etcd does not natively encrypt data, it facilitates a multi-layered security model to prevent unauthorized access to its database files:  
     
   // Work in progress  
     
   

	

## 01-SD: Pod lifecycle dataflow

![01-SD Pod lifecycle dataflow](images/01-SD%20Pod%20lifecycle%20dataflow.png)

This dataflow shows the key interactions between etcd and Kubernetes components during a typical pod lifecycle

**Key Interactions Shown:**

1. **Pod Creation**: User creates a pod through kubectl, which goes to the API server, gets validated, and is stored in etcd  
2. **Watch Mechanisms**: Controllers and schedulers watch the API server, which reads from etcd to provide real-time updates  
3. **Scheduling**: The scheduler receives unscheduled pods, makes decisions, and updates the pod specification in etcd via the API server  
4. **Execution**: Kubelet watches for pods assigned to its node and updates pod status back to etcd via the API server.  
5. **Status Queries**: Users can query pod status, which is retrieved from etcd through the API server

**Important Notes:**

* etcd serves as the single source of truth for all cluster state  
* All Kubernetes components interact with etcd exclusively through the API server  
* The API server handles authentication, validation, and authorization before persisting to etcd  
* Watch mechanisms enable real-time updates without constant polling  
* etcd's consistency guarantees ensure all components see the same cluster state

This pattern extends to all Kubernetes resources (Services, Deployments, ConfigMaps, etc.) \- they all follow similar interaction patterns with etcd as the persistent storage backend.

## 02-SD: CRUD operations between the API Server and etcd using gRPC

![02-SD CRUD operations between API server and etcd](images/02-SD%20CRUD%20operations.png)

This sequence diagram shows the detailed CRUD and Watch operations between the Kubernetes API Server and etcd using gRPC:

**Key gRPC Operations Demonstrated:**

**CREATE:**

* API Server uses `gRPC Put()` to store new resources  
* etcd returns a revision number for the operation

**READ:**

* Single resource: `gRPC Range()` with specific key  
* Multiple resources: `gRPC Range()` with `WithPrefix()` option

**UPDATE:**

* Uses `gRPC Put()` with the same key to overwrite  
* API Server validates ResourceVersion for optimistic concurrency

**DELETE:**

* Uses `gRPC DeleteRange()` to remove resources  
* Handles Kubernetes-specific logic like finalizers

**WATCH:**

* Uses `gRPC Watch()` with streaming response  
* Supports prefix watching and starting from specific revisions  
* Transforms etcd events (PUT/DELETE) into Kubernetes events (ADDED/MODIFIED/DELETED)

**Advanced Features:**

* **Transactions**: `gRPC Txn()` for atomic compare-and-swap operations  
* **Compaction**: `gRPC Compact()` to cleanup old revisions  
* **Revision-based consistency**: All operations return revision numbers for ordering

**Resource Storage Pattern:** All Kubernetes resources are stored in etcd with hierarchical keys like:

* `/registry/pods/namespace/name`  
* `/registry/services/namespace/name`  
* `/registry/configmaps/namespace/name`

The consistency/atomicity is guaranteed by etcd's consensus protocol (raft). This architecture ensures strong consistency, atomic operations, and real-time change notifications across the entire Kubernetes cluster.

## Choose Data Flow Diagram Level

Start with:  
	•	Level 0: High-level overview (API Server ↔ etcd)  
	•	Level 1: Detailed flow between all control plane components

⬅️➡️ Flows:  
	•	kubectl → API Server: REST call  
	•	API Server → etcd: Read/Write (gRPC)  
	•	etcd → API Server: State data  
	•	Controller Manager → API Server: Watches/Updates  
	•	Scheduler → API Server: Watches, schedules Pods

## Tools for Drawing

Use one of the following:  
	•	Draw.io: Rich shapes, ready for DFDs  
	•	Excalidraw: Good for sketch/prototype  
	•	Lucidchart: If you want pre-made Kubernetes templates

## Optional Enhancements

If needed, include:  
	•	TLS/Authentication flows  
	•	Leader election in controller-manager/scheduler  
	•	Metrics or observability components (e.g., Prometheus)

# Threat Modeling with STRIDE

In Kubernetes, **etcd** is the key-value store that serves as the **single source of truth** for cluster state. It stores critical information such as:

* API objects (Pods, ConfigMaps, Secrets, etc.)Cluster configuration and policies

* Authentication and authorization data

Because etcd holds the **single source of truth**, it becomes a high-value target. A compromise of etcd can mean full compromise of the cluster since etcd stores all cluster state, configuration data, and metadata, providing a centralized, consistent view of the cluster's health and desired state. Threat modeling helps us systematically understand and mitigate risks.

One of the most widely used frameworks is **STRIDE**, which categorizes threats into six classes: **Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege**.

## 1\. Spoofing

### STRIDE-ETCD-SPOOF-1 – Spoofing of etcd client identity

An attacker that obtains etcd client TLS certificates (from disk, backup, snapshot, or compromised control plane node) can impersonate a legitimate etcd client (e.g., kube-apiserver) and read or modify cluster state directly, bypassing Kubernetes RBAC entirely

**Impact**

* Full read/write access to Secrets, RBAC, CRDs  
* Cluster-wide compromise without API server logs

**Recommended Mitigations**

* Use **mutual TLS** for etcd client access  
* Restrict file permissions on etcd client certificates  
* Use separate certs per component (API server, controller-manager)  
* Rotate etcd client certificates regularly  
* Monitor etcd auth logs for unexpected client identities  
   **Status:** End user guidance / partially implemented

### STRIDE-ETCD-SPOOF-2 – Spoofing of etcd peer node

An attacker-controlled node could impersonate an etcd peer during cluster bootstrapping or reconfiguration, potentially injecting or manipulating cluster state.

**Recommended Mitigations**

* Mutual TLS for peer traffic  
* Static peer membership where possible  
* Disable dynamic peer discovery  
* Network-level isolation of peer traffic

Threat:  
	•	An attacker pretends to be an authorized client or peer.  
	•	Exploits weak authentication between Kubernetes API server and etcd.  
	•	It can influence leader election in Raft by sending a higher term number to force      re-election and get the cluster into constant state of election

Mitigations:  
	•	Enforce mutual TLS (mTLS) between peers and clients.  
	•	Rotate and revoke certificates regularly.  
	•	Use Role-Based Access Control (RBAC) to restrict who can access etcd directly.

* \--peer-client-cert-auth=true [https://docs.datadoghq.com/security/default\_rules/5be-7yq-bjy/](https://docs.datadoghq.com/security/default_rules/5be-7yq-bjy/)  
* \--peer-auto-tls=false https://docs.datadoghq.com/security/default\_rules/t6p-v9r-6k8/  
  


## 2\. Tampering

### STRIDE-ETCD-TAMPER-1 – Tampering with etcd data at rest

An attacker with filesystem access to etcd data directories can directly manipulate the datastore, WAL files, or snapshots, altering cluster state without leaving Kubernetes audit logs.

**Recommended Mitigations**

* Enable **encryption at rest** for Secrets  
* Restrict filesystem access on control plane nodes  
* Monitor integrity of etcd data directories  
* Alert on unexpected etcd restarts  
   **Status:** Partially implemented / End user guidance

### STRIDE-ETCD-TAMPER-2 – Tampering via etcd snapshot restore

A malicious or compromised backup/restore pipeline can inject altered etcd snapshots, reintroducing deleted RBAC bindings, Secrets, or CRDs.

**Recommended Mitigations**

* Sign and verify etcd snapshots  
* Restrict who can perform restore operations  
* Audit snapshot creation and restore events  
* Store backups encrypted and access-controlled

Threat:  
	•	Malicious modification of stored objects (e.g., changing a ConfigMap, altering RBAC roles).  
	•	Manipulating etcd data during transit if TLS is misconfigured.

Malicious modification of etcd snapshots during backup/restore.

Mitigations:  
	•	Enable encryption in transit with TLS.  
	•	Enable encryption at rest for sensitive objects (e.g., Secrets).  
	•	Restrict direct access to etcd (only API server should talk to it).  
	•	Audit policies for data integrity validation.  
Protect etcd by verifying snapshot hashes, optionally signing backups, and storing them encrypted, immutable, and off-cluster. Restrict direct access with RBAC, isolate etcd on a trusted network, and regularly test restores to detect tampering early.

## 3\. Repudiation

### STRIDE-ETCD-REPUDIATE-1 – Direct etcd access bypassing audit trails

Actions performed directly against etcd are not recorded in Kubernetes audit logs, allowing attackers to modify cluster state without attribution.

**Recommended Mitigations**

* Restrict etcd access to API server only  
* Enable etcd request logging  
* Centralize logs and correlate with node-level events

Threat:  
	•	Actions against etcd (writes, deletes) without proper audit trails.  
	•	Difficulty proving malicious changes were made.

Mitigations:  
	•	Enable Kubernetes audit logging at the API server.  
	•	Use etcd audit logs (in recent versions) or external logging.  
	•	Integrate with a SIEM to correlate suspicious activity.

[https://palospublishing.com/supporting-audit-logs-with-cryptographic-verification](https://palospublishing.com/supporting-audit-logs-with-cryptographic-verification)

## 4\. Information Disclosure

### STRIDE-ETCD-INFODISCLOSE-1 – Disclosure of Kubernetes Secrets from etcd

etcd stores Kubernetes Secrets (base64 \+ optionally encrypted). Compromise of etcd or its backups leads to disclosure of:

* Application credentials  
* Cloud provider keys  
* Service account tokens

**Recommended Mitigations**

* Enable strong encryption-at-rest providers  
* Protect encryption keys (HSM/KMS-backed)  
* Rotate encryption keys regularly  
* Secure etcd backups with encryption and access control

### STRIDE-ETCD-INFODISCLOSE-2 – Disclosure via etcd metrics and health endpoints

Misconfigured etcd metrics or health endpoints can leak cluster metadata, keys, or internal topology.

**Recommended Mitigations**

* Protect metrics endpoints with TLS and auth  
* Restrict network access to metrics ports  
* Disable unauthenticated health checks  
   

Threat:  
	•	Unauthorized read of sensitive objects (Secrets, service tokens).  
	•	Network sniffing if traffic is unencrypted.  
	•	Backups of etcd not properly secured.

Mitigations:  
	•	Use etcd encryption at rest with strong key management (KMS provider).  
	•	Limit access to etcd data directories and snapshot files.  
	•	Protect etcd backups (encryption \+ access control).  
	•	Isolate etcd in a dedicated, hardened network segment.

[https://docs.cloud.google.com/kubernetes-engine/docs/how-to/rotate-etcd-kcp-encryption-keys](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/rotate-etcd-kcp-encryption-keys)

## 5\. Denial of Service (DoS)

### STRIDE-ETCD-DOS-1 – Exhaustion of etcd storage

An attacker can create excessive objects (CRDs, events, leases) causing etcd disk exhaustion, leading to API server unavailability.

**Recommended Mitigations**

* Enforce resource quotas and object limits  
* Limit CRD size and count  
* Monitor etcd disk usage and compaction health  
   

### STRIDE-ETCD-DOS-2 – Compaction and performance abuse

Abuse of frequent writes or large objects can increase compaction pressure, causing latency spikes or outages.

**Recommended Mitigations**

* Tune compaction settings  
* Monitor etcd latency and write amplification  
* Alert on abnormal write rates  
   

Threat:  
	•	Flooding etcd with requests to exhaust CPU, memory, or disk I/O.  
	•	Large object storage (e.g., big ConfigMaps, Secrets) causing performance degradation.

Mitigations:  
	•	Apply resource requests/limits and monitor etcd health.  
	•	Enforce object size limits and quotas in Kubernetes.  
	•	Use PodSecurity/NetworkPolicies to restrict access to etcd.  
	•	Scale etcd cluster with 3–5 nodes for resilience.

[https://www.clouddefense.ai/cve/2020/CVE-2020-15112](https://www.clouddefense.ai/cve/2020/CVE-2020-15112)  
..attacker to trigger runtime panics during consensus by manipulating entry indexes.  
[https://etcd.io/docs/v3.5/op-guide/data\_corruption/](https://etcd.io/docs/v3.5/op-guide/data_corruption/)

https://github.com/etcd-io/etcd/issues/16837  
[https://tangcong.info/post/2020-09-19-etcd-qos-proposal/](https://tangcong.info/post/2020-09-19-etcd-qos-proposal/)

Compared with other storage backends, is etcd more prone to Dos Attack ?   
[https://deepwiki.com/kubernetes-sigs/apiserver-builder-alpha/5.2-alternative-storage-backends](https://deepwiki.com/kubernetes-sigs/apiserver-builder-alpha/5.2-alternative-storage-backends)  
[https://docs.k3s.io/datastore](https://docs.k3s.io/datastore)

⸻

## 6\. Elevation of Privilege

STRIDE-ETCD-EOP-1 – Elevation of privilege via direct etcd write

An attacker with write access to etcd can:

* Grant themselves `cluster-admin`  
* Inject malicious admission configs  
* Modify API server flags indirectly

**Recommended Mitigations**

* Ensure etcd is not reachable from workloads  
* Strict node-level hardening  
* Separate etcd from general-purpose control plane workloads

STRIDE-ETCD-EOP-2 – Compromise of encryption-at-rest keys

If encryption provider keys stored on disk are compromised, attacker can decrypt historical and future secrets.

**Recommended Mitigations**

* Use external KMS providers  
* Restrict key access and rotation  
* Audit key usage  
   

Threat:  
	•	Attackers with limited cluster role gains access to etcd.  
	•	Direct read of Secrets leads to cluster-wide privilege escalation.

Mitigations:  
	•	Restrict etcd access to control plane nodes only.  
	•	Enforce least privilege RBAC in Kubernetes.  
	•	Use network policies/firewalls to block non-API access to etcd.  
	•	Monitor for suspicious privilege escalation attempts.

 It allows a remote attacker to authenticate as a valid RBAC user by exploiting a TLS certificate issue.  
[https://www.clouddefense.ai/cve/2018/CVE-2018-16886](https://www.clouddefense.ai/cve/2018/CVE-2018-16886)

# References

## Docs

- [https://etcd.io/docs/v3.6/](https://etcd.io/docs/v3.6/)  
- Raft in etcd: [https://static.sched.com/hosted\_files/kccncosschn19eng/ea/KubeCon%20China%202019\_%20Raft%20in%20etcd.pdf?\_gl=1\*8xqhls\*\_gcl\_au\*NDQ4NjY4MjYyLjE3NTczMTE1MDU.\*FPAU\*NDQ4NjY4MjYyLjE3NTczMTE1MDU](https://static.sched.com/hosted_files/kccncosschn19eng/ea/KubeCon%20China%202019_%20Raft%20in%20etcd.pdf?_gl=1*8xqhls*_gcl_au*NDQ4NjY4MjYyLjE3NTczMTE1MDU.*FPAU*NDQ4NjY4MjYyLjE3NTczMTE1MDU).   
- [https://www.microsoft.com/en-us/security/blog/2020/04/02/attack-matrix-kubernetes/](https://www.microsoft.com/en-us/security/blog/2020/04/02/attack-matrix-kubernetes/)

## Talks

- [https://www.youtube.com/watch?v=DrtdrdwDpZE](https://www.youtube.com/watch?v=DrtdrdwDpZE) Deep Dive:etcd \- Jingyi Hu

## Links

- PRs that support raft learners in etcd: [\#10725](https://github.com/etcd-io/etcd/pull/10725), [\#10727](https://github.com/etcd-io/etcd/pull/10727), [\#10730](https://github.com/etcd-io/etcd/pull/10730).

## Others

- [https://d3fend.mitre.org/resources/](https://d3fend.mitre.org/resources/)  
- [https://d3fend.mitre.org/resources/ontology/](https://d3fend.mitre.org/resources/ontology/)  
- [https://mlsec.org/docs/2025-ccsw.pdf](https://mlsec.org/docs/2025-ccsw.pdf)

[https://www.startupdefense.io/cyberattacks/kubernetes-etcd-exploitation](https://www.startupdefense.io/cyberattacks/kubernetes-etcd-exploitation)  
[https://www.wallarm.com/cloud-native-products-101/what-is-etcd](https://www.wallarm.com/cloud-native-products-101/what-is-etcd)