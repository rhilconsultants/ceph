## Ceph's Cluster components 

This wxercise will walk you through the different cluster components, it will cover the distruction and creation of OSDs, will show how to interact 
with it using the `ceph-volume` tool, and will show how versatile is Ceph as a storage system. 

Install jq package to parse ceph commands output:

```bash
$ yum install -y jq
```

Create a replicated pool with 32 pgs: 

```bash
$ ceph osd pool create replication 32 32 
```
Upload an object to Ceph's object store using the `rados put` command:

```bash
# Upload an object to the pool 
$ rados put -p replication hosts /etc/hosts
```

List the pool to see that the object was successfully created:

```bash
$ rados ls -p replication
hosts
```

Log-in to one of your OSD hosts with `vagrant ssh`:

```bash 
vagrant ssh osd0
```

Collect the OSDs lvm paths for this specific host, each OSD is created on a given lvm, with this command we will use the `ceph-volume` tool 
to filter out the lvm paths for every OSD in this particular host:

```bash
$ HOST_LV_PATHS=$(for OSD_INFO in $(ceph-volume lvm list --format json | jq -c '.[]');do echo ${OSD_INFO} | jq -r '.[].lv_path' ;done)
```

Stop the OSDs and mark them out, that way those OSDs will not get any IO and the cluster will enter a **degraded state** because 33.33% of it's data is currently 
not part of the cluster: 

```bash
$ for OSD_ID in `ceph osd ls-tree $HOSTNAME`;do systemctl stop ceph-osd@${OSD_ID}; ceph osd out ${OSD_ID};done
marked out osd.<ID>. 
marked out osd.<ID>.
```

Now you should have 2 down OSDs and ⅓ of your data should be in a degraded state (data will not replicate to other locations).
Try downloading the object once again to verify that although ⅓ of the data is unavailable, you can still access it:

```bash
$ rados get -p replication hosts hosts_file
```

Purge the OSDs from the cluster, which will remove them from the CRUSH map and will zap the lvms (data is lost for this host), you should see 4 OSDs now:

```bash
$ for OSD_ID in `ceph osd ls-tree $HOSTNAME`;do ceph osd purge ${OSD_ID} --yes-i-really-mean-it;done
purged osd.<id>
purged osd.<id>
```


Zap the OSDs lvms, this task unmounts the OSD dir and deletes all data from the block device:

```bash
$ for OSD_ID in $HOST_LV_PATHS;do ceph-volume lvm zap ${OSD_ID};done 

--> Zapping: /dev/ceph-82331c89-47bc-46dd-9619-4649e9a89a0c/osd-data-bf59da4b-d0ba-4427-aeab-e154c2fb1ee9
--> Unmounting /var/lib/ceph/osd/ceph-2
.
.
.
--> Zapping successful for: <LV: /dev/ceph-82331c89-47bc-46dd-9619-4649e9a89a0c/osd-data-bf59da4b-d0ba-4427-aeab-e154c2fb1ee9>
--> Zapping: /dev/ceph-1687deb2-a264-42e4-8095-989772370e67/osd-data-91d5148d-0ef8-4f42-8b80-68562bf7f799
.
.
.
```

Verify with ceph -s that all pgs are in active state, and that you have now 4 OSDs out of 4 (it means that the other two were deleted). 

Create the OSDs once again given the previously collected lvm paths (this process uses `ceph-volume` tool to create new OSDs with no data): 

```bash
$ for OSD_ID in $HOST_LV_PATHS;do ceph-volume lvm create --bluestore --data ${OSD_ID};done
.
.
.
--> ceph-volume lvm activate successful for osd ID: <id>
--> ceph-volume lvm create successful for: ceph-82331c89-47bc-46dd-9619-4649e9a89a0c/osd-data-bf59da4b-d0ba-4427-aeab-e154c2fb1ee9

.
.
.
--> ceph-volume lvm activate successful for osd ID: <id>
--> ceph-volume lvm create successful for: ceph-1687deb2-a264-42e4-8095-989772370e67/osd-data-91d5148d-0ef8-4f42-8b80-68562bf7f799
```

The OSDs will be created and added to the cluster, Ceph will recognize automatically that it has more locations to replicate data, and data will 
start replicating to the new OSDs. 
Verify with ceph -s that all pgs are in active state, and all data was entirely replicated (no degraded data) 

Verify you can still read the object the you have uploaded once again: 

```bash
$ rados get -p replication hosts hosts_file
```
