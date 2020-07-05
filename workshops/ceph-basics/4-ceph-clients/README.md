# Ceph clients 

This exercise will walk you through the different access methods Ceph exposes (File, Object, Block storage). In the end of this exercise you will be familiar with Ceph as a unified SDS solution for all storage types. 

## Ceph RADOS Gateway 

Connect to one of your mon hosts using the `vagrant ssh` command: 

```bash 
vagrant ssh mon0
```

Create a S3 user using the `radosgw-admin` command, This command will create a user for you to interact with Ceph's S3 interface: 

```bash
$ radosgw-admin user create --uid=test --display-name=test --access-key=test --secret-key=test
```

Install the `awscli` package to interact with the S3 interface: 

```bash
$ pip3 install --upgrade awscli 
```

Add your AWS credenials as env vars so that `awscli` could fetch those credentials: 

```bash
$ export AWS_ACCESS_KEY_ID=test
$ export AWS_SECRET_ACCESS_KEY=test
```

Create a bucket using awscli tool 

```bash
$ aws s3 mb s3://test --endpoint-url http://192.168.42.10:8080
```

Upload a file to the created bucket using awscli tool 

```bash
$ aws s3 cp /etc/hosts s3://test --endpoint-url http://192.168.42.10:8080
```

Verify that the object was uploaded successfully 

```bash
$ aws s3 ls s3:// --endpoint-url http://192.168.42.10:8080
```

List both data and metadata pools to see objects were created

```bash
$ rados ls -p default.rgw.buckets.data
97b3ffd5-ffa7-48cd-80e4-d0d76e34af25.4209.1_hosts
```

```bash
$ rados ls -p default.rgw.buckets.index
.dir.97b3ffd5-ffa7-48cd-80e4-d0d76e34af25.4209.1
```

As you see, in the index pool you have one object with the bucket ID as the key. This is made so that Ceph will be able to to track which objects are located in 
which bucket. In the data pool we have the object itself with the bucket ID as the prefix.


Get the metadata of the object created (from the index pool):

```bash
$ rados -p default.rgw.buckets.index listomapkeys .dir.97b3ffd5-ffa7-48cd-80e4-d0d76e34af25.4209.1
hosts
```

As you see, if we list the keys for this particular objects (holds key value store) we could see that it contains the object name that we have uploaded. 

Let's do the same to list the value for the index object: 

```bash
$ rados -p default.rgw.buckets.index listomapvals .dir.97b3ffd5-ffa7-48cd-80e4-d0d76e34af25.4209.1
hosts
value (199 bytes) :
00000000  08 03 c1 00 00 00 05 00  00 00 68 6f 73 74 73 03  |..........hosts.|
00000010  00 00 00 00 00 00 00 01  07 03 5a 00 00 00 01 88  |..........Z.....|
00000020  00 00 00 00 00 00 00 ce  64 80 5e 1f 66 c9 28 20  |........d.^.f.( |
00000030  00 00 00 63 63 33 32 34  64 64 30 30 33 35 65 37  |...cc324dd0035e7|
00000040  39 66 65 35 66 30 64 30  61 61 30 39 35 37 36 39  |9fe5f0d0aa095769|
00000050  34 39 39 04 00 00 00 74  65 73 74 04 00 00 00 74  |499....test....t|
00000060  65 73 74 00 00 00 00 88  00 00 00 00 00 00 00 00  |est.............|
00000070  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000080  01 01 02 00 00 00 0d 03  02 2c 00 00 00 39 37 62  |.........,...97b|
00000090  33 66 66 64 35 2d 66 66  61 37 2d 34 38 63 64 2d  |3ffd5-ffa7-48cd-|
000000a0  38 30 65 34 2d 64 30 64  37 36 65 33 34 61 66 32  |80e4-d0d76e34af2|
000000b0  35 2e 34 32 30 37 2e 31  32 00 00 00 00 00 00 00  |5.4207.12.......|
000000c0  00 00 00 00 00 00 00                              |.......|
000000c7
```

## Ceph RADOS Block Device 

Create a replicated pool called `rbd` with 32 pgs: 

```bash
$ ceph osd pool create rbd 32 32 
pool 'rbd' created
```

Enable rbd application on the specific pool: 

```bash
$ ceph osd pool application enable rbd rbd
enabled application 'rbd' on pool 'rbd'
```

Create an 10GB size rbd image called `block`, This image will be stored in the `rbd` pool:

```bash
$ rbd create --size 10G rbd/block
```

Verify the image was created successfully:

```bash
$ rbd ls -p rbd
block
```

Check for the image's size using rhe `rbd du` command, this shows how much space is consumed with this RBD image: 

```bash
$ rbd du rbd/block

NAME  PROVISIONED USED 
block      10 GiB  0 B
```

Create a user and save it’s key to the /etc/ceph directory, if you are using your client machine, please copy the key to the wanted directory:

```bash
$ ceph auth get-or-create client.rbd mon 'profile rbd' osd 'profile rbd pool=rbd'
$ ceph auth get client.rbd -o /etc/ceph/ceph.client.rbd.keyring
``` 

Disable rbd features to work with krbd kernel module (those features are not supported using the krbd kernel module):

```bash
$ rbd feature disable block object-map fast-diff deep-flatten
```

Verify the created user can access the rbd pool by listing the created pool:

```bash
$ rbd ls --user rbd -p rbd
block
```
This command lists the pool remotly to see if the created image is indeed located in the pool and can be accessed. 

Map the rbd image to your local machine (This will create a block device on your local machine, verify by running `lsblk -l`).

```bash
$ rbd map --user rbd rbd/block 

/dev/rbd0
```

Make an ext4 filesystem on the given block device on your local machine: 

```bash
$ mkfs.ext4 /dev/rbd0 
```

Mount the created block device to your local machine: 

```bash
$ mkdir /mnt/myrbd && mount -t ext4 /dev/rbd0 /mnt/myrbd
```

Verify the directory is mounted: 

```bash
$ df -h | grep /mnt/myrbd
```

Check the metadata for this filesystem using the `stat` commmand: 

```bash
$ stat -f /mnt/myrbd
  File: "/tmp/mount/"
    ID: 10c4bc2acf586785 Namelen: 255     Type: ext2/ext3
Block size: 4096       Fundamental block size: 4096
Blocks: Total: 2547525    Free: 2538303    Available: 2403135
Inodes: Total: 655360     Free: 655348
```

## Ceph Filesystem 

**Note:** CephFS will be automatically created for you within the cluster deployment.

Verify that the MDS is active (this daemon holds the metadata regarding the location of the CephFS files created):

```bash
$ ceph mds stat

cephfs:1 {0=mon0=up:active}
```

Verify you have CephFS filesystem running:

```bash
$ ceph fs ls

name: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data ]
```

Verify the filesystem's status: 

```bash
$ ceph fs status

cephfs - 0 clients
======
+------+--------+------+---------------+-------+-------+
| Rank | State  | MDS  |    Activity   |  dns  |  inos |
+------+--------+------+---------------+-------+-------+
|  0   | active | mon0 | Reqs:    0 /s |   10  |   13  |
+------+--------+------+---------------+-------+-------+
+-----------------+----------+-------+-------+
|       Pool      |   type   |  used | avail |
+-----------------+----------+-------+-------+
| cephfs_metadata | metadata | 1536k | 90.8G |
|   cephfs_data   |   data   |    0  | 90.8G |
+-----------------+----------+-------+-------+
+-------------+
| Standby MDS |
+-------------+
+-------------+
```

As you see, we have an active filesystem with one MDS (which is mon0) and we have 2 pools that were created in the deployment, one for the data store
and the second for the metadata. 

Create a client for interacting with the filesystem: 

```bash
$ ceph auth get-or-create client.cephfs mon 'allow r' mds 'allow rw' osd 'allow rw pool=cephfs_data'
```

Get the created key and save only base64 key to a file:

```bash
$ ceph auth get client.cephfs 
[client.cephfs]
	key = AQAnhYBe088jDxAAPpnpG2T3kOCKRvdkhWxpGw==
``` 

```bash
$ echo "AQAnhYBe088jDxAAPpnpG2T3kOCKRvdkhWxpGw==" > /etc/ceph/ceph.client.cephfs.secret
```

Change the file's ownership: 

```bash 
$ chmod 644 /etc/ceph/ceph.client.cephfs.secret
```

Create a data directory locally:

```bash
$ mkdir /mnt/mycephfs
```

Mount the filesystem using the created user and the created secret file: 

```bash
$ mount -t ceph 192.168.42.10:6789:/ /mnt/mycephfs -o name=cephfs,secretfile=/etc/ceph/ceph.client.cephfs.secret
```

Verify the directory is mounted:

```bash

$ df -h | grep /mnt

192.168.42.10:6789:/          91G     0   91G   0% /mnt/mycephfs
```

Verify the directory has ceph filesystem type by using the `stat` command:

```bash
$ stat -f /mnt/mycephfs/
  File: "/mnt/mycephfs/"
    ID: 950d0baa0f734639 Namelen: 255     Type: ceph
Block size: 4194304    Fundamental block size: 4194304
Blocks: Total: 23266      Free: 23266      Available: 23266
Inodes: Total: 0          Free: -1
```