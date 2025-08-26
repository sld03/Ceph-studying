# Architecture

# 1. Ceph Storage cluster

Ceph uniquely delivers **object, block, and file system** in one unified system

**Ceph Monitors** maintain the master copy of the cluster map, which they provide to Ceph clients. The existence of multiple monitors in the Ceph cluster ensures availability if one of the monitor daemons or its host fails.

**A Ceph OSD Daemon** checks its own state and the state of other OSDs and reports back to monitors.

**A Ceph Manager** serves as an endpoint for monitoring, orchestration, and plug-in modules.

**A Ceph Metadata Server (MDS)** manages file metadata when CephFS is used to provide file services.

Ceph OSD Daemons use CRUSH algorithm to compute information about the location data. This algorithm helps avoud bottlenecked by a central lookup table.

# 2. Storing data

The data come from anywhere (object storage, file system or block device), it will be stored as **RADOS object.**

Each object is stored on an **Object Storage Device (OSD). OSD** control read, write and replication operation of storage device.

![Untitled](Architecture%2045669077ce6843a882991c2dabaf6fa7/Untitled.png)

# 3. High Availability and Scalability

Ceph apply CRUSH algorithm to avoid single point of falure.

## 3.2 Cluster Map

Cluster Map stores the information of the cluster’s topology.

Cluster Map is stored in all Ceph Client and Ceph OSDs

**There are 5 Maps in the Cluster Map**

### **3.2.1 The Monitor Map**

Contains the cluster `fsid`, the position, the name, the address, and the TCP port of each monitor. The monitor map specifies the current epoch, the time of the monitor map’s creation, and the time of the monitor map’s last modification. To view a monitor map, run `ceph mon dump`.

![Untitled](Architecture%2045669077ce6843a882991c2dabaf6fa7/Untitled%201.png)

**Remove a monitor node from a cluster**

### **3.2.2 The OSD Map**

Contains the cluster `fsid`, the time of the OSD map’s creation, the time of the OSD map’s last modification, a list of pools, a list of replica sizes, a list of PG numbers, and a list of OSDs and their statuses (for example, `up`, `in`). To view an OSD map, run `ceph osd dump`.

### **3.2.3 The PG Map**

Contains the PG version, its time stamp, the last OSD map epoch, the full ratios, and the details of each placement group. This includes the PG ID, the Up Set, the Acting Set, the state of the PG (for example, `active + clean`), and data usage statistics for each pool.

### **3.2.4 The CRUSH Map:**

Contains a list of storage devices, the failure domain hierarchy (for example, `device`, `host`, `rack`, `row`, `room`), and rules for traversing the hierarchy when storing data. To view a CRUSH map, run `ceph osd getcrushmap -o {filename}` and then decompile it by running `crushtool -d {comp-crushmap-filename} -o {decomp-crushmap-filename}`. Use a text editor or `cat` to view the decompiled map.

### **3.2.5 The MDS Map:**

Contains the current MDS map epoch, when the map was created, and the last time it changed. It also contains the pool for storing metadata, a list of metadata servers, and which metadata servers are `up` and `in`. To view an MDS map, execute `ceph fs dump`.

Each Map maintains a history of changes to its operating state.

# Quay lai tim hieu sau khi cai dat ceph

## 3.1 High Availability monitor

If a cluster has only 1 monitor node, then it will be a single point of falure. So, Ceph leverage a Monitor cluster.

Ceph uses Paxos algorithm to establish consensus of the cluster state. This algorithm requires the acceptance of more than a half of the number of monitor node in the cluster (for example 3 over 5, 3 over 4, not 2 over 4,…)

So, it is recommended to setup the number of node in a cluster to be an odd number, to prevent  the split brain phenomenom.

## 3.2 High availability authentication

cephx authentication system is used by ceph to protect against man-in-the-middle-attack.

Cephx use secret keys for authentication. this means both client and monitor cluster keeps a copy of the key.

Ceph doesn’t have a centralized interface between client and object store, because it might lead to a bottleneck. To solve this problem, client have to interact directly with OSD, and this require authentication.

### Authentication Scheme

![Untitled](Architecture%2045669077ce6843a882991c2dabaf6fa7/Untitled%202.png)

## 3.3 Smart daemon enable hyperscale

Many storage cluster use employs a centralized interface to keepj track of the nodes that users are permitted to access. It might be the botteneck for petabyte to exabyte scale.

To solve this problem, in Ceph, Ceph clients, Ceph Monitor and Ceph OSD interact with each other **directly.** This offers some benefits

- Clients access content directly, help to improve performance and capacity (because some network interface allow a finite number of concurrent connection)
- Often, to check the availability of an OSD, it will send the message periodically to monitor node. After a period of no responding, monitor will consider that OSD to be ***down.*** But normally in Ceph, an neighbor OSD will notice that that node is down and inform to the monitor node. This make the operation of the monotor node more lightweight.
- Replication
    
    ![Untitled](Architecture%2045669077ce6843a882991c2dabaf6fa7/Untitled%203.png)
    

## 3.4 Pools

Pools are logical partitions for storing objects.

Ceph Clients retrieve a [Cluster Map](https://docs.ceph.com/en/reef/architecture/#cluster-map) from a Ceph Monitor, and write RADOS objects to pools. The way that Ceph places the data in the pools is determined by the pool’s size or number of replicas, the CRUSH rule, and the number of placement groups in the pool.

### 3.4.1 Placement group

![Untitled](Architecture%2045669077ce6843a882991c2dabaf6fa7/Untitled%204.png)

Each pool has a number of placement groups within it. Placement group performs functions of placing object into OSD. Placement groups (PGs) are an internal implementation detail of how Ceph distributes data. CRUSH dynamically map PGs to OSDs and objects to PGs.

Placement group acts as a “*layer of indirection*”, abstract the OSD topology bellow. This helps the pool to rebalance when when a new OSD come online. 

### 3.4.2 Calculate PG ID

The client requires only the object ID and the name of the pool in order to compute the object location.

Ceph stores data in named pools (for example, “liverpool”). When a client stores a named object (for example, “john”, “paul”, “george”, or “ringo”) it calculates a placement group by using the object name, a hash code, the number of PGs in the pool, and the pool name. 

# Xem lai cach tinh pg id sau khi tim hieu thuatj toan crush

### 3.4.3 Peering and Set

Introduce Active set and Upset

![Untitled](Architecture%2045669077ce6843a882991c2dabaf6fa7/Untitled%203.png)

As mentioned in the section **Smart daemon enable hyperscale,** The data is write into OSD will go through the primary OSD and then replicate to 2 or more OSDs. The primary OSD refers to the first OSD in the Active set, which is responsible for oschestrating the peering process of each placement group. Primary OSD is the only OSD in a placement group that is allowed client to initiate a write operation.

***Up set*** is the set of all the OSDs with state up. ***Up set*** is a subset of the ***Active set.*** If an OSD down, it will be removed from ***Up set***, but not ***Active Set.*** 

## 3.5 Rebalancing