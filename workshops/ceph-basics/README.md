# Ceph Basics Workshop

This workshop will walk through Ceph's basics in all levels (from the RADOS level to the application level) to create basic knowledge across all Ceph offerings.

## Prerequisites

* Vagrant 2.2.7
  * Install Virtualbox/Libvirt/Parallels provider
  * `Vagrant box add generic/rhel7` (will be faster)

* Python3
  * `Python3 --version`

* ansible >= 2.8
  * For RHEL/Fedora:
    * `subscription-manager repos --enable ansible-2.8-for-rhel-8-x86_64-rpms && dnf install -y ansible`
    * `ansible --version`

  * For mac:
    * `pip3 install --upgrade ansible==2.8`
    * `ansible --version`
  
* Code Ready Containers
  * https://cloud.redhat.com/openshift/install/crc/installer-provisioned
  * Download tar.xz file and the pull secret
  * Add binary file to your PATH