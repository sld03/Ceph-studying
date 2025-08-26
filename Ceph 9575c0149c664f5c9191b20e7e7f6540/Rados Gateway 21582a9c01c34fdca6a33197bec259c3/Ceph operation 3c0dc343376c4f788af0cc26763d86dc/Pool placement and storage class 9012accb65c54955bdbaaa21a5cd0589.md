# Pool placement and storage class

# 1. Placement Target

Placement target control which pools are associated with a particular bucket.

Bucket placement target is selected on creation and can not be modified.

The zonegroup configuration contains a list of placement targets with an initial target named `default-placement`. The zone configuration then maps each zonegroup placement target name onto its local storage. This zone placement information includes the `index_pool` name for the bucket index, the `data_extra_pool` name for metadata about incomplete multipart uploads, and a `data_pool` name for each storage class.

# 2. Storage class

Storage classes are used to customize the placement of object data. S3 Bucket Lifecycle rules can automate the transition of objects between storage classes.

Storage classes are defined in terms of placement targets. Each zonegroup placement target lists its available storage classes with an initial class named `STANDARD`. The zone configuration is responsible for providing a `data_pool` pool name for each of the zonegroup’s storage classes

# 3. Labs

Add new Storage class

```
radosgw-admin zonegroup placement add --rgw-zonegroup default --placement-id default-placement --storage-class PREMIUM
```

Create 3 OSD with device class ssd 

![Untitled](Pool%20placement%20and%20storage%20class%209012accb65c54955bdbaaa21a5cd0589/Untitled.png)

Create Pool with SSD device class

Create storage class **PREMIUM** for the default placement

![Untitled](Pool%20placement%20and%20storage%20class%209012accb65c54955bdbaaa21a5cd0589/Untitled%201.png)

Assign zone placement info for that storage class

```bash
radosgw-admin zone placement add --rgw-zone default --placement-id default-placement --storage-class GLACIER --data-pool default.rgw.glacier.data --compression lz4
```

![Untitled](Pool%20placement%20and%20storage%20class%209012accb65c54955bdbaaa21a5cd0589/Untitled%202.png)

Change default storage class of an user

```bash
radosgw-admin user modify --uid <user-id> --placement-id <default-placement-id --storage-class <default-storage-class> --tags <tag1,tag2>
```

![Untitled](Pool%20placement%20and%20storage%20class%209012accb65c54955bdbaaa21a5cd0589/Untitled%203.png)

After change the default permission for the user `mrwick` then create his bucket and put object to that bucket, the data pool will be created automatically.

![Untitled](Pool%20placement%20and%20storage%20class%209012accb65c54955bdbaaa21a5cd0589/Untitled%204.png)

In practice, we need to place data of premium pool into the ssd device class, so we create a replicated rule to place data, then assign that rule into the pool

![Untitled](Pool%20placement%20and%20storage%20class%209012accb65c54955bdbaaa21a5cd0589/Untitled%205.png)

![Untitled](Pool%20placement%20and%20storage%20class%209012accb65c54955bdbaaa21a5cd0589/Untitled%206.png)

![Untitled](Pool%20placement%20and%20storage%20class%209012accb65c54955bdbaaa21a5cd0589/Untitled%207.png)

![Untitled](Pool%20placement%20and%20storage%20class%209012accb65c54955bdbaaa21a5cd0589/Untitled%208.png)

## Transition bucket from the bucket

![Untitled](Pool%20placement%20and%20storage%20class%209012accb65c54955bdbaaa21a5cd0589/Untitled%209.png)

Object will be stored in `asia-sa.rgw.meta` pool, namespace `lc`, and can only be retrieved with the command `rados -p asia-sa.rgw.log --namespace=lc getomapheader lc.1` . The content is empty.

![Untitled](Pool%20placement%20and%20storage%20class%209012accb65c54955bdbaaa21a5cd0589/Untitled%2010.png)

There are multiple `lc` stored in that namespace, andthe number of `lc` is defined by a property. 

Object expiration

![Untitled](Pool%20placement%20and%20storage%20class%209012accb65c54955bdbaaa21a5cd0589/Untitled%2011.png)