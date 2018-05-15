Starting from a new project definition on an OpenStack installation

## Prerequisites
### Python / Ansible / Terraform
Download an up to date RC file from the project overview page and source it so
that we can authenticate to the openstack API. Terraform will need a few extra
variables as well. There's an init script in the root of this repository which
will look for openrc.sh with the correct variables for project etc. and will
prompt you for your openstack credentials

```
  . init.sh cybera
```

### Terraform

Terraform is used to define 3 VMs running CentOS 7 on a smallish flavour. The
variables.tf file defines defaults for some standard variables which can be
overridden by setting the corresponding environment variable (e.g.
TF_VAR_os_password). If you have sourced the init.sh script above then you
should be ready to go. In most cases you can just change to the terraform
directory and run `terraform plan` to see what actions will be taken, but you
may need to run `terraform init` to grab the openstack plugin (running terraform
plan should inform you if that is the case).

```
  $ terraform plan
  $ terraform apply
```

Terraform will create the resources defined in k8s-syzygy.tf and report back the
floating IP of the master node. Assign a name for this in DNS (e.g.
k8s1.syzygy.ca)

### Ansible

Once the machines are deployed we need to take care of some initialization
tasks, most of the can be done via ansible, but we will want an inventory. The
inventory construction process is described on the [kubespray
README.md](https://github.com/kubernetes-incubator/kubespray/blob/master/README.md).
Following that you should have an inventory in inventory/mycluster along with
some configuration variables.

```
$ cd kubespray
$ cp -rpf inventory/sample inventory/mycluster
# Check the IPS via openstack GUI or CLI
$ declare -a IPS=(192.168.180.126 192.168.180.124 192.168.180.125)
$ CONFIG_FILE=inventory/mycluster/hosts.ini python3
  contrib/inventory_builder/inventory.py ${IPS[@]}
```

We've decided to try NFS as a distributed storage provider so we modify the
inventory to identify an nfs-server. This server will be provisioned as part of
the plays/init.yml playbook.

```
$ vi inventory/mycluster/hosts.ini
[all]
master1     ansible_host=192.168.180.126 ip=192.168.180.126
node1       ansible_host=192.168.180.124 ip=192.168.180.124
node2       ansible_host=192.168.180.125 ip=192.168.180.125

[kube-master]
master1

[nfs-server]
master1

[kube-node]
master1
node1
node2

[etcd]
master1
node1
node2

[k8s-cluster:children]
kube-node
kube-master
```

Set the OS and an overlay networking provider.
```
$ vi inventory/mycluster/group_vars/all.yml
 +bootstrap_os: centos

$ vi inventory/mycluster/group_vars/k8s-cluster.yml
 +kube_network_plugin: flannel
```

Add the local IP addresses to .ssh/config as bastien ssh clients
```
# kubernetes on CC
Host 192.168.180.*
    User ptty2u
    ProxyCommand ssh -l ptty2u k8s1.syzygy.ca -W %h:%p
    IdentityFile ~/.ssh/id_cc_openstack
    StrictHostKeyChecking no
```

Now run the init playbook to update the hosts (N.B. the kernel update will
include a reboot task.
```
  $ cd ansible
  $ ansible-playbook -i hosts.ini  plays/init.yml
```


## kubespray

### Deploy
Our kubespray deployment is fairly vanilla, but since we've had problems with
the docker storage backend we will use overlay2 (we have a 4.4 kernel). Be
CAREFUL to use the right device in the config file below.

```
  $ vi ./inventory/mycluster/group_vars/docker-storage.yml
---
docker_container_storage_setup_version: v0.6.0
docker_container_storage_setup_profile_name: kubespray
docker_container_storage_setup_storage_driver: overlay2
docker_container_storage_setup_devs: /dev/sdb

  $ cd kubespray
  $ ansible-playbook -i inventory/mycluster/hosts.ini -b -v cluster.yml
```

The deploy can run for quite a long time (10-15 minutes) and will report status
for each of the tasks defined in the kubespray playbooks. When it is finished,
log in to the master node and check the cluster status

```
 $ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: REDACTED
    server: https://192.168.180.117:6443
  name: cluster.local
contexts:
- context:
    cluster: cluster.local
    user: admin-cluster.local
  name: admin-cluster.local
current-context: admin-cluster.local
kind: Config
preferences: {}
users:
- name: admin-cluster.local
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED

  $ kubectl get nodes
NAME      STATUS    ROLES         AGE       VERSION
master1   Ready     master,node   2m        v1.9.5+coreos.0
node1     Ready     node          2m        v1.9.5+coreos.0
node2     Ready     node          2m        v1.9.5+coreos.0
```

## Distributed Storage

A lot of what kubernetes does relies on having distributed storage. The trick is
that the storage has to be under kubernetes' direct control so that it can
allocate/deallocate and otherwise manipulate volumes. The most common storage
providers are the cloud based ones, (gcePersistentDisk, awsElasticBlockStore
etc.) but nfs and gluster are also possible.

### NFS

In the hosts.ini above, we have listed an nfs-server so init.yaml should have
partitioned a volume and configured /etc/exports on that machine. On top of the
raw storage we need a provider to allow kubernetes to control volume
lifecycling. We will use the nfs-client provisioner from the
[kubernetes/external-storage](https://github.com/kubernetes-incubator/external-storage)
project. On master1, clone out that repository.

```
 $ git clone https://github.com/kubernetes-incubator/external-storage
```

The kubespray installation uses RBAC so we need to follow the [instructions for
authentication](https://github.com/kubernetes-incubator/external-storage/blob/master/nfs/docs/authorization.md).
N.B. These instructions are actually for the nfs provider, the documentation for
nfs-client is missing.
```
  $ cd external-storage/nfs-client/deploy
  $ kubectl create -f auth/serviceaccount.yaml
  
  $ vi auth/clusterrole.yaml
  - apiVersion: rbac.authorization.k8s.io/v1
  + apiVersion: rbac.authorization.k8s.io/v1beta1
  $ kubectl create -f auth/clusterrole.yaml

  $ vi auth/clusterrolebinding.yaml
  - apiVersion: rbac.authorization.k8s.io/v1
  + apiVersion: rbac.authorization.k8s.io/v1beta1
  $ kubectl create -f auth/clusterrolebinding.yaml

  $ vi deployment.yaml
             - name: PROVISIONER_NAME
               value: fuseim.pri/ifs
             - name: NFS_SERVER
-              value: 10.10.10.60
+              value: 192.168.180.126 # Check the IP address of your NFS server
             - name: NFS_PATH
-              value: /ifs/kubernetes
+              value: /export
       volumes:
         - name: nfs-client-root
           nfs:
-            server: 10.10.10.60
-            path: /ifs/kubernetes
+            server: 192.168.180.126 # Check the IP address of your NFS server
+            path: /export


  $ kubectl create -f deployment.yaml
```

Add a storageClass and make it the default
```
  $ vi class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
  annotations:
    storageclass.beta.kubernetes.io/is-default-class: "true"
provisioner: fuseim.pri/ifs # or choose another name, must match deployment's env PROVISIONER_NAME'

  $ kubectl create -f class.yml
```

You should be able to see the nfs-client-provisioner pod come up.
```
  $ kubectl get pods
NAME                                      READY     STATUS    RESTARTS   AGE
nfs-client-provisioner-7c878d4d49-hf6gr   1/1       Running   0          1h
```
If you run into problems, check `kubect logs nfs-client...

#### Testing - Make a PVC
We can test a claim

```
  $ vi test-pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myclaim-1
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

  $ kubectl create -f test-pvc.yaml

  $ kubectl get pvc
NAME        STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
myclaim-1   Bound     pvc-36f2aa39-57d3-11e8-8af6-fa163e2d1d82   1Gi        RWO            managed-nfs-storage   6s

  $ kubectl delete -f test-pvc.yaml
```
