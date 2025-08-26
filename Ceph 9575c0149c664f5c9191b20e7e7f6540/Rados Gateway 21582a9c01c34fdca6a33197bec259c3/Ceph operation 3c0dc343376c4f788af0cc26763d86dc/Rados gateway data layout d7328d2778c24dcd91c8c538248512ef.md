# Rados gateway data layout

Object has Extended attributes (xattrs) and object map (OMAP)

# 1. rgw.root

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled.png)

## 1.1 Zoneinfo

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%201.png)

## 1.2 Zone.names.us-east

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%202.png)

## 1.3 period

After change any zone or zone group information, we need to run the command `radosgw-admin update period —commit` . 

Updating the period changes the epoch, and ensures that other zones will receive the updated configuration.

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%203.png)

# 2. Metadata

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%204.png)

## 2.1 User

Contain several namespaces, include root, user.keys, user.uid, user.email

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%205.png)

### 2.1.2 users.keys

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%206.png)

Each object include only UID, allow us to look up `uid` from `access_key`.

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%207.png)

### 2.1.3 user.uid

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%208.png)

## 2.2 Bucket

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%209.png)

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%2010.png)

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%2011.png)

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%2012.png)

## 2.3 Bucket.instance

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%2013.png)

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%2014.png)

# 3. Bucket Index

## 3.1 Omap

Key-value database

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%2015.png)

## 3.2 Extended Attribute (xattr)

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%2016.png)

## 3.3 Bucket Index

There are many object in the pool index, but it doesn’t store any information (it’s content is 0 byte).

Ceph doesn’t store any data in the bucket, but store its information in Omap key-value. Omap database is often run in high speed database to increase the performance of the whole system.

Object format : .dir.<bucket-id>.<shard-id>

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%2017.png)

The index information is actually kept in the key/value store in ceph (LevelDB). Each OSD has a colocated leveldb key/value store. So the object is really just acting as a place holder for ceph to find which OSD’s key/value store contains the index.

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%2018.png)

We can recognize that the omap value store the object name, uid, display name, etag (MD5Sum of the object) and tag.

Get omap header and decode, we get the statistical information of the bucket

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%2019.png)

The index object also store omap key/value of the object in a bucket. Key is the object name (hello) and value is it’s metadata information.

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%2020.png)

# 4. Log

## 4.1 gc (gabage collector)

Large file data in RGW is generally deleted in the background, and this pool is used to record file objects to be deleted.

## 4.2 lc (life cycle)

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%2021.png)

Omap value of the lc object

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%2022.png)

The marker is the name of the `bucket.instance`, which is stored in the pool `asia-sa.rgw.meta`. That means the lifecycle rule is stored in the object `.bucket.meta.bucket-3-premium:2baa6f8d-edbc-4b06-9f9f-0504a6a583cb.25757.1` .

## 4.3 reshard

# 5. Control

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%2023.png)

When the RGW is powered on, several objects are created in the control pool for watch-notify. The main function is to ensure data consistency when a zone corresponds to multiple RGWs and the cache is enabled. The basic principle is to use the object watch-notify function provided by librados. When data is updated, other RGWs are notified to refresh the cache. There will be a document specifically describing the RGW cache later.

# 6. Data

## 6.1 Single part upload

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%2024.png)

Each data object in the data pool has the following attributes

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%2025.png)

Manifest attribute

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%2026.png)

## 6.2 Multipart upload

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%2027.png)

![Untitled](Rados%20gateway%20data%20layout%20d7328d2778c24dcd91c8c538248512ef/Untitled%2028.png)