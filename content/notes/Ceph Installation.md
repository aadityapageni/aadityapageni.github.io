+++
title = "Ceph Installation"
date = 2026-01-20
[taxonomies]
tags = ["ceph"]
+++

ceph clustering in 3 vm in vagrant

- install cephadm on any one node
```
apt update -y
apt install -y cephadm ceph-common
```

- make sure python, lvm2, docker and chrony is installed on other nodes, these are installed on working node auto 

- use bootstrap command to 

- Create a Monitor and a Manager daemon for the new cluster on the local host.
- Generate a new SSH key for the Ceph cluster and add it to the root user’s /root/.ssh/authorized_keys file.
- Write a copy of the public key to /etc/ceph/ceph.pub.
-  Write a minimal configuration file to /etc/ceph/ceph.conf. This file is needed to communicate with Ceph daemons.
- Write a copy of the client.admin administrative (privileged!) secret key to /etc/ceph/ceph.client.admin.keyring.
-  Add the _admin label to the bootstrap host. By default, any host with this label will (also) get a copy of /etc/ceph/ceph.conf and /etc/ceph/ceph.client.admin.keyring.

   
        `cephadm bootstrap --mon-ip <self.ip(192.168.56.101)>`
    
    - access ceph gui after this at port 8443
    
- adding hosts
` ssh-copy-id -f -i /etc/ceph/ceph.pub root@*<new-host>`

- `cephadm shell` to work with ceph cli

`ceph orch host add <host2(ubuntu2)> 10.10.0.102`

- give admin acess to multiple hosts, recommend to give more than one
  
    ` ceph orch host label add *<host>* _admin`
- consume all available devices to OSD
		`ceph orch apply osd --all-available-devices`
- check for devices with `ceph orch device ls`
- `ceph orch apply osd --all-available-devices --dry-run`

- add manager in 2 nodes
`ceph orch apply mgr node1,node2`

#### **ceph object gateway**
- this deploys 2 rgw instance within SERVICE_NAME
`ceph orch apply rgw SERVICE_NAME`
- for multi cluster config, there are realm, zone, zone-group
- [red hat docs](https://docs.redhat.com/en/documentation/red_hat_ceph_storage/5/html/object_gateway_guide/deployment#deploying-the-ceph-object-gateway-using-the-command-line-interface_rgw)

- test rgw with boto3/s3
- create s3 account with rados-admin | [sauce](https://docs.redhat.com/en/documentation/red_hat_ceph_storage/5/html/object_gateway_guide/testing#create-an-s3-user-rgw)

```
radosgw-admin user create --uid=jane --subuser=jane:swift --access=full --display-name="jane"
{
    "user_id": "jane",
    "display_name": "jane",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "subusers": [
        {
            "id": "jane:swift",
            "permissions": "full-control"
        }
    ],
    "keys": [
        {
            "user": "jane",
            "access_key": "N0QLVUM9ATZQ6ACJEEGH",
            "secret_key": "02vPmQwj6n1ZGsEX6FDpqSPhqor6kyI84cMIPYJj",
            "active": true,
            "create_date": "2025-09-13T09:36:19.387859Z"
        }
    ],
    "swift_keys": [
        {
            "user": "jane:swift",
            "secret_key": "36c8GEOp6w2HPrg81o97XvkpuCEN7HccugzSD6rl",
            "active": true,
            "create_date": "2025-09-13T09:36:19.429579Z"
        }
    ],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "default_storage_class": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": [],
    "account_id": "",
    "path": "/",
    "create_date": "2025-09-13T09:36:19.387432Z",
    "tags": [],
    "group_ids": []
}

```
- gives `access key` and `secret key`
- install `python3-boto3`
```
import boto3

endpoint = "" # enter the endpoint URL along with the port "http://URL:PORT"

access_key = 'ACCESS'
secret_key = 'SECRET'

s3 = boto3.client(
        's3',
        endpoint_url=endpoint,
        aws_access_key_id=access_key,
        aws_secret_access_key=secret_key
        )

s3.create_bucket(Bucket='my-new-bucket')

response = s3.list_buckets()
for bucket in response['Buckets']:
    print("{name}\t{created}".format(
		name = bucket['Name'],
		created = bucket['CreationDate']
))
```
#### ceph block storage
- create pool, enable rbd and  Initialize pool for use by RBD.
```
ceph osd pool create <poolname(pool1)>
ceph osd pool application enable <poolname(pool1)> rbd
rbd pool init -p <poolname(pool1)>
```

- create rbd images and list with info
```
rbd create image1 --size <size in MB> --pool <poolname(pool1)>
rbd --image <image_name> info
```


- maps block image to local device
`rbd map <image-name> --pool <poolname>`
`rbd showmapped`

##### share local device(block image ) over NFS
- exit cephadm shell
```
sudo apt install nfs-kernel-server -y
mkdir /mnt/rbd-nfs
mount /dev/rbd0 /mnt/rbd-nfs
echo "/mnt/rbd-nfs *(rw,sync,no_root_squash)" | sudo tee -a /etc/exports
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```

- on clients
```
sudo apt install nfs-common -y
sudo mount <nfs-server-ip>:/mnt/rbd-nfs /mnt
df -h | grep nfs
```

- check if rbd client is loaded
`modinfo rbd  # Should show module details`

- load rbd in client
`sudo modprobe rbd`
#### CephFs

`ceph fs volume create cephfs`

This command creates a CephFS filesystem named "cephfs" with the following:
- A metadata pool (cephfs_metadata) – Stores directory structures, file names, permissions, etc.
- A data pool (cephfs_data) – Stores actual file contents.
- An MDS daemon – Handles metadata operations (listing directories, permissions, etc.)

- to create cephfs manually
`ceph fs new cephfs cephfs_metadata cephfs_data`

- create subvolume group
`ceph fs subvolumegroup create cephfs mygroup`

- create subvolume within the group
`ceph fs subvolume create cephfs mysubvol --group mygroup`

#### ceph authentication

`ceph auth get-or-create client.libvirt mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=vm-storage' mds 'allow *'`

🔹 This creates a user client.myuser with:
Read (r) access to MONs
Read/Write (rw) access to the "vm-storage" pool

`ceph auth list`
To list all users

`ceph auth caps client.libvirt mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=vm-storage' mds 'allow *'`
- to modify existing user

`ceph auth del client.myuser`
- deletes the user, myuser

##### mount cephFS on client
`sudo apt install ceph-fuse -y`
`mkdir /mnt/cephfs`

`mount -t ceph <mon-ip>:6789:/ /mnt/cephfs -o name=admin,secret=<ceph-secret>`

`df -h | grep ceph`



## Rook

- Testing Bucket Functionality with an S3 Client

```
apiVersion: v1
kind: Pod
metadata:
  name: s3-test-pod
  namespace: rook-ceph
spec:
  containers:
    - name: s3-client
      image: registry1.dso.mil/ironbank/opensource/amazon/aws-cli:latest
      command:
      - "/bin/sh"
      - "-c"
      - |
        echo "Hello Rook" > /tmp/rookObj
        aws s3 cp /tmp/rookObj s3://ceph-bucket/rookObj
        aws s3 cp s3://ceph-bucket/rookObj /tmp/downloadedObj
      env:
        - name: AWS_ACCESS_KEY_ID
          value: N0QLVUM9ATZQ6ACJEEGH
        - name: AWS_SECRET_ACCESS_KEY
          value: 02vPmQwj6n1ZGsEX6FDpqSPhqor6kyI84cMIPYJj
```