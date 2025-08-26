# PG State

# 1. Example

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled.png)

- **105 pgs:** This indicates you have a total of 105 placement groups
- **105 active+undersized+remapped:** All 105 PGs are currently in an "active+undersized+remapped" state.
- **147 GiB / 150 GiB avail:** This indicates the available storage space on the OSDs
- **444/666 objects misplaced (66.667%)**: This is a critical piece of information. It highlights that 444 out of 666 objects (around 67%) are currently misplaced within the PGs. This means these objects aren't stored on their ideal OSDs due to remapping.

# 2. Terminologies

## 2.1 Peering

Before writing data to PG, pg must be in ***active*** state and preferably be in ***clean*** state.  For Ceph to determine the current state of a PG, peering must take place. That is, the primary OSD of the PG (that is, the first OSD in the Acting Set) must peer with the secondary and OSDs so that consensus on the current state of the PG can be established

![Untitled](Architecture%2045669077ce6843a882991c2dabaf6fa7/Untitled%203.png)

………

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%201.png)

## 2.2 Backfill

Backfilling is the process when you remove or add an OSD, CRUSH reassign the PG to others OSD and triggers migration of data.

# 3. Monitoring PG states

## 2.1 Cases when ceph health may not be OK

1. You have just created a pool and the PGs haven’t peered yet.
2. The PGs are recovering.
3. You have just added an OSD to or removed an OSD from the cluster.
4. You have just modified your CRUSH map and your PGs are migrating.
5. There is inconsistent data in different replicas of a PG.
6. Ceph is scrubbing a PG’s replicas.
7. Ceph doesn’t have enough storage capacity to complete backfilling operations.

## 2.2 Show ceph pg status

To show the statistic of the placement group, use command : `ceph pg stat`

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%202.png)

To dump all pg status, use command : `ceph pg dump` 

## 2.3 Process of creating a PG

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%203.png)

# 4. PG state

## 4.1 Active————————-

After Ceph has completed the peering process, a PG should become `active`. The `active` state means that the data in the PG is generally available for read and write operations in the primary and replica OSDs.

## 4.2 Clean———————————

When a PG is in the `clean` state, all OSDs holding its data and metadata have successfully peered and there are no stray replicas. Ceph has replicated all objects in the PG the correct number of times.

## 4.2 Degraded ———————————-

When a client writes an object to the primary OSD, the primary OSD is responsible for writing the replicas to the replica OSDs. After the primary OSD writes the object to storage, the PG will remain in a `degraded` state until the primary OSD has received an acknowledgement from the replica OSDs that Ceph created the replica objects successfully
The `degraded` state can appear along with `active` state. Active means client can still write and read data from the PG, but the quality is degraded because not all OSD is up and the placement group is not fully funtional as expexted.

If an OSD is down and the `degraded` state persists, then Ceph will mark that OSD as out of the cluster, remap the data from that OSD to another OSD. The interval for Ceph to mark a down OSD to be out of the cluster is determined by`mon_osd_down_out_interval`, which is set to `600` seconds by default.

The logs below show the pg statistic after an OSD is down. Some pg become undersized some are degraded. After `mon_osd_down_out_interval` , it is is marked as `out` and the data migration is triggered, return all PGs back to the active and clean state.

The PGs that do contain the object will be in the state `active+undersized+degraded`, and the PGs with no object inside will only be in the `active+undersize` state.

## 4.3 Recovering———————————

When an OSD goes `down`, its data might not be up to date. When the OSD is back to `up` state, the `recovering` process will be performed to recover the OSD to the latest status.

The logs bellow show the pg state of the cluster after start the OSD. Before that, the OSD is down and the all PGs settled down at `active` and `clean` state, then some object is put into the cluster through radosgw, then bring the OSD up again.

Recovery operation can be ***recovering, forced_recovery, recovery_wait, recovery_toofull, recovery_unfound.***

### 4.3.1 Recovery_toofull ——————————————

The `osd.4` has reached the `full_ratio`

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%204.png)

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%205.png)

It is the case when i put an an OSD down, then put about 80Gb data into ceph cluster, then turn that OSD back on. The recovery process will be perform, and all the PG related to `osd.4` has encounterd the `recovery_toofull` state. 

The reason is the unbalance among host. While ceph CRUSH replicate object between host, but host 1 only has 30GB, compare to 90GB and 90GB in host 2 and 3, host 1 will be full soon.

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%206.png)

To handle this situation, there are serveral ways:

- set the `full_ratio` parameter, using command `ceph osd set-full-ratio <ratio>` to a larger value, but it is a temporary solution, because the OSD will soon reach `full_ratio` .
- Resize the OSD
- Add more storage capacity into the host 1.

### 4.3.2 force-recovery——————————

Sometime, data in some pg is more important than other pgs. So, we need to instruct ceph to prioritize that pg to be recovered first. To do this, use the command

```bash
ceph pg force-recovery <pgid> <pgid> ...
```

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%207.png)

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%208.png)

## 4.4 Backfilling——————————

Ceph is scanning and synchronizing the entire contents of a placement group instead of inferring what contents need to be synchronized from the logs of recent operations. Backfill is a special case of recovery.

When a new OSD joins the cluster, CRUSH will reassign PGs from OSDs that are already in the cluster to the newly added OSD. This can lead to operational performance. To maintain performance, Ceph perform the migration of data in background with ***backfilling*** process, whichs set backfill operation to a lower priority than request to read or write data.

During the backfill operations, you might see one of several states: `backfill_wait` indicates that a backfill operation is pending, but is not yet underway; `backfilling` indicates that a backfill operation is currently underway; and `backfill_toofull` indicates that a backfill operation was requested but couldn’t be completed due to insufficient storage capacity. When a PG cannot be backfilled, it might be considered `incomplete`.

In the logs bellow, I add an OSD to the cluster, then the cluster will perform rebalancing. Some PG need to peering, and some are added to backfilling state.

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%209.png)

## 4.5 Remapped————————

The placement group is temporarily mapped to a different set of OSDs from what CRUSH specified

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2010.png)

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2011.png)

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2012.png)

After add a new osd, PG ***11.7f*** is remapped into a new actiing set. and the `last_change` and `last_peered` change as well

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2013.png)

## 4.6 Peer————————————

- `Peering` : the PG is peering
- `Peered` : The placement group has peered, but cannot serve client IO due to not having enough copies to reach the pool’s configured ***min_size*** parameter. Recovery may occur in this state, so the pg may heal up to min_size eventually.

**PG temp**

A temporary placement group acting set that is used while backfilling the primary OSD. Assume that the acting set is `[0,1,2]` and we are `active+clean`. Now assume that something happens and the acting set becomes `[3,1,2]`. Under these circumstances, OSD `3` is empty and can’t serve reads even though it is the primary. `osd.3` will respond by requesting a *PG temp* of `[1,2,3]` to the monitors using a `MOSDPGTemp` message, and `osd.1` will become the primary temporarily. `osd.1` will select `osd.3` as a backfill peer and will continue to serve reads and writes while `osd.3` is backfilled. When backfilling is complete, *PG temp* is discarded. The acting set changes back to `[3,1,2]` and `osd.3` becomes the primary.

We can use the command `ceph pg repeer <pgid>`  to force a pg to repeer.

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2014.png)

After peering, the perring time will be updated in the ceph pg info (use `ceph pg query` to check)

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2015.png)

**Peered**

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2016.png)

## 4.6 Stale  ————————————

The placement group is in an unknown state - the monitors have not received an update for it since the placement group mapping changed.

This often appears when you **remove an OSD**, then the Monitor will not know any thing about that OSD. It will become `stale` in a short period, until an OSD report that the OSD is down and it is marked as `out` and trigger the migration operation. 

When an OSD is put down

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2017.png)

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2018.png)

And the **last_unstale** time will be updated. 

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2019.png)

If we query   `ceph pg dump` fast enough, then we can capture the stale state of the pg.

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2020.png)

When all OSDs (both primary and replica) are down, then PGs will become stale as well.

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2021.png)

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2022.png)

## 4.7 Undersized ———————

The placement group has fewer copies than the configured pool replication level. It appear when some OSD is down.

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2023.png)

Remove `osd.3`

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2024.png)

## 4.8 Scrubbing ——————

Scrubbing is a mechanism in Ceph to maintain data integrity, similar to ***fsck*** in the file system, that will help in finding if the existing data is inconsistent.
Ceph's OSD (Objet Storage Daemon) will regularly start the Scrub thread to scan some objects, and compare them with other replicas to find out whether they are consistent. If there is an inconsistency, Ceph will throw this exception and hand it over to the user to solve.

Scrub is performed so quick so it is hard to capture that state of the pg.

To order a pg to scrub or deep scrubbing, use command

```bash
ceph pg scrub <pgid>
ceph pg deep-scrub <pgid>
```

To check the last scrubbing time, use `ceph pg dump` and check the last scrubbing column.

Use ceph query to check the scrubbing information

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2025.png)

Last scrub time in query logs

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2026.png)

Scrub result. any scrub error will make pg inconsistent.

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2027.png)

Or use ceph pg dump to check th last scrub time.

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2028.png)

- [ ]  Thuật toán scrubbing nào, efect của nó lên hệ thống, làm thế nào để tối ưu, các tham số liên quan đến scrub và deep scrub

## 4.9 Laggy

[Docs for laggy and wait](https://docs.ceph.com/en/reef/dev/osd_internals/stale_read/)

Each replica can serve read request within an interval called a ***lease***. Primary and replicate track the property `readable_until`  and `readable_until_ub`, which determine the interval that PG are readable. After `readable_until` passes, the ***lease*** is considered expired and if client request for reading in this expired time, the PG will be in `laggy` state. After ***readable_until,*** ceph renew this time by the message *pg_lease_t* and  *pg_lease_ack_t.* 

## 4.10 Wait

If peering completes but the prior interval may still be readable, then PG will be in `wait` state. This state will remain until the existing lease expires. 

During this interval, PG will block new request to pg. 

## 4.11 Down ———————————

In case your cluster has 3 copies of data, and it requires 2 of them to be available to operate, then if the number of copies is fewer than 2, the PG is considered down.

Normal state of pg 11.1d

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2029.png)

## 4.12 I**nconsistent ———————————**

Ceph detects inconsistencies in the one or more replicas of an object in the placement group (e.g. objects are the wrong size, objects are missing from one replica *after* recovery finished, etc.).

In most cases, errors during scrubbing cause inconsistency within placement group.

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2030.png)

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2031.png)

Corrupt data

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2032.png)

Instruct ceph to deep scrub the inconsistent pg and find the srubbing error.

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2033.png)

- [ ]  Kiểm tra các cách khác corrupt data, kiểu xóa object file, fill lại object với same data size nhưng data khác.
- [ ]  Nếu OSD bị lỗi data 2/3 OSD và

## 4.13 Repair———————————

Ceph is checking the placement group and repairing any inconsistencies it finds (if possible)

![Untitled](PG%20State%2009cf553db6e34441b7c643dcdae2a8a0/Untitled%2034.png)