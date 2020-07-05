# rook-ceph 

This exercise will walk you through rook-ceph as a containerized orchestrator for Openshift and Kubernetes environments. In this exercise you'll repeat all Ceph's basics in a containerized Openshift CRC cluster. 

## rook-ceph RADOS 

Add oc command into your path: 

```bash
$ eval $(./crc oc-env)
```

Login the the OCP cluster, login command will show up after crc start command: 

```bash
$ oc login -u kubeadmin -p <password> https://api.crc.testing:6443
```

Clone the rook 1.2-release branch (where we have all the needed manifests):

```bash
$ git clone --single-branch -b release-1.2 https://github.com/rook/rook.git && cd rook/cluster/examples/kubernetes/ceph
```

Create common definitions and the openshift operator for ceph deployments:

```bash
$ oc create -f common.yaml && oc create -f operator-openshift.yaml 
```

This command creates all the needed resources for the Operator to run successfuly in our Openshift CRC cluster. 

Change the namespace context for convenience:

```bash
$ oc project rook-ceph 
```

Verify you have a running operator before moving on (pod should be in running state):

```bash
$ oc get pods -w | grep operator 
```

Run the cluster configuration to create a Ceph cluster, this CR creates a Ceph cluster with a given parameters:

```bash
$ oc create -f cluster-test.yaml 
```

Verify the cluster has created successfully by using the cephcluster CRD (health should be `HEALTH_OK`):

```bash
$ oc get cephclusters 
```

Deploy the toolbox pod used for interacting with the ceph cluster: 

```bash
$ oc create -f toolbox.yaml 
```

The toolbox pod contains all the needed binaries for us to interact with the containerized Ceph cluster. 

Connect into the toolbox pod, and run ceph status (ignore the warning):

```bash
$ oc rsh $(oc get pods | grep tools | awk '{print $1}') ceph status 
```

Create a generic pool with replica 1: 

```bash
$ oc create -f pool-test.yaml 
```

Run rados lspools to verify a pool `replicapool` was created:

```bash
$ oc rsh $(oc get pods | grep tools | awk '{print $1}') rados lspools 
```

Connect to the toolbox pod and verify you can upload/download objects to the cluster: 

```bash  
$ oc rsh  $(oc get pods | grep tools | awk '{print $1}')
$ rados put -p replicapool hosts /etc/hosts
$ rados get -p replicapool hosts /tmp/hosts
$ cat /tmp/hosts
```

Scale out the monitor number to 3 by patching the cephcluster CRD:

```bash
$ oc patch cephcluster rook-ceph -p '{"spec":{"mon":{"count":3}}}' --type merge
```

This way we patch the Cephcluster deployment to add more mons to our clusters (could e useful when adding more servers the the Openshift cluster).

Verify you have 3 mons now: 

```bash
$ oc get pods | grep mon 
```

Delete the created pool 

```bash
$ oc delete -f pool-test.yaml
```

## rook-ceph RADOS Gateway 

Create an object storage pool using the pre-created manifest: 

```bash
$ oc create -f object-test.yaml 
```

Watch the objectstore CRDs to verify successful creation:

```bash
$ oc get cephobjectstores
```

Create an object storage user: 

```bash
$ oc create -f object-user.yaml 
```

This command creates a user automatically to interact with the S3 interface (as we did with the `radosgw-admin` command). 

Expose the rgw service (will create an ingress route so that we could access the S3 service outside of our Oppenshift cluster): 

```bash
$ oc expose svc/$(oc get svc | grep rgw | awk '{print $1}')
```

Set user credentials to access the S3 service using `awscli`:

```bash
$ export AWS_ACCESS_KEY_ID=`oc get secret rook-ceph-object-user-my-store-my-user -o 'jsonpath={.data.AccessKey}' | base64 --decode`
$ export AWS_SECRET_ACCESS_KEY=`oc get secret rook-ceph-object-user-my-store-my-user -o 'jsonpath={.data.SecretKey}' | base64 --decode`
```

Validate that you get the XML and that the S3 service is responsive: 

```bash
$ curl $(oc get route | grep rgw | awk '{print $2}') && export ENDPOINT_URL=`echo $(oc get route | grep rgw | awk '{print $2}')`
```

Create a bucket and upload an object using `awscli`: 

```bash
$ aws s3 mb s3://test --endpoint-url http://${ENDPOINT_URL}
$ aws s3 cp --acl public-read /etc/hosts  s3://test --endpoint-url http://${ENDPOINT_URL}
```

Download the object and verify its valid: 

```bash
$ wget http://${ENDPOINT_URL}/test/hosts
$ cat hosts 
```

Scale the rgw instance number by patching the cephobjectstore CRD: 

```bash
$ oc patch cephobjectstore my-store  -p '{"spec":{"gateway":{"instances":3}}}' --type merge
```

Verify you have 3 RGW pods: 

```bash
$ oc get pods | grep rgw
```

Upload a file again to verify S3 service is valid: 

```bash
$ aws s3 cp --acl public-read /etc/hosts  s3://test --endpoint-url http://${ENDPOINT_URL}
```

## rook-ceph RADOS Block Device 

Change directory to the rbd provisioning dir:

```bash
$ cd csi/rbd 
```

Create a storage class used for RWO access:

```bash
$ oc create -f storageclass-test.yaml 
```

This CR integrates the creation of a RBD pool in our Ceph cluster (using the Operator), and the creation of a Storage class that uses the Ceph RBD pool. 

Verify the Storage Class was created: 

```bash
$ oc get sc 
```

Create a PVC from that storage class:

```bash
$ oc create -f pvc.yaml 
```

Check pvs is in bound state: 

```bash
$ oc get pvc 
```

Create a pod and mount the created PVC to /var/lib/www/html dir: 

```bash
$ oc create -f pod.yaml 
```

Wait for the pod to spin up, then check if the rbd volume is mounted:

```bash
$ oc rsh csirbd-demo-pod lsblk -l
```

## rook-ceph Filesystem

Change directory to the rbd provisioning dir: 

```bash
$ cd csi/cephfs 
```

Add the following cephfilesystem config to storageclass.yaml file before the StorageClass config: 

```bash 
oc create -f - <<EOF
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  metadataPool:
    failureDomain: host
    replicated:
      size: 1
  dataPools:
    - failureDomain: host
      replicated:
        size: 1
  preservePoolsOnDelete: true
  metadataServer:
    activeCount: 1
    activeStandby: true

---
EOF 
```

Verify you have 2 MDS pods running in your cluster: 

```bash
$ oc get pods | grep mds 
```

Create a storage class used for RWX access to be accessed via the created cephfilesystem:

```bash
$ oc create -f storageclass.yaml 
```

Verify a Storage Class was created:

```bash
$ oc get sc
```

Create a PVC from that storage class:

```bash
$ oc create -f pvc.yaml 
```

Check pvc is in bound state: 

```bash
$ oc get pvc 
```

Create a pod and mount the created PVC to /var/lib/www/html dir:

```bash
$ oc create -f pod.yaml 
```

Wait for the pod to spin up, then check if the cephfs volume is mounted:

```bash
$ oc rsh csicephfs-demo-pod df -h
```