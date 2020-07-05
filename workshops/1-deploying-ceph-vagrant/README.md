## Deploying your RHCS4 Vagrant cluster 

From your computer, unzip the zip file into your local directory 

```bash
$ wget https://github.com/shonpaz123/ceph-tools/raw/master/ceph-ansible-rhcs_4.0.zip
```

```bash 
$ unzip ceph-ansible-rhcs_4.0.zip
```

Change directory into the unzipped directory 

```bash
$ cd ceph-ansible-rhcs-4-vagrant 
```
As root, Start cluster provisioning (the `RHN_USRNAME` and `RHN_PASSWORD` are your RedHat login credentials)

```bash
(root) $ ANSIBLE_ARGS='--extra-vars "rh_username=${RHN_USERNAME} rh_password=${RHN_PASSWORD}"' vagrant up --provision --provider ${VAGRANT_PROVIDER}
```