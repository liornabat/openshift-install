# Preparation

* At least 4 nodes - master1-3 node1 with NFS
* All nodes should be CentOS 7
* External LB and DNS records

* Each node should have DNS record and also the host's names should be set as the DNS record
```
# in each node
sudo hostnamectl set-hostname sub.exmaple.com 
# in your DNS 
set A record for all the nodes
set A record for the LB IP
set CNAME that forward to the LB or hosts that the infra is set
such *.apps.example.com 
```
* Make sure that NetworkAddapter is enabled and active in all nodes
```

sudo systemctl unmask NetworkManager.service
sudo systemctl start NetworkManager.service
sudo systemctl enable NetworkManager.service
sudo systemctl status NetworkManager

# check 
sudo systemctl status NetworkManager

# install if needed
sudo yum -y NetworkManager
```


# Installation
## Setting up Development host

1. Install Packages

```javascript
sudo yum install -y epel-release
sudo yum -y update
sudo yum -y install git wget ansible python-cryptography pyOpenSSL.x86_64 python-passlib java-1.8.0-openjdk-headless httpd-tools
git clone https://github.com/openshift/openshift-ansible
```
2. Create ssh key and get the public key

```javascript
ssh-keygen -t rsa -f ~/.ssh/my-ssh-key -C [USERNAME]
chmod 400 ~/.ssh/my-ssh-key
cat ~/.ssh/my-ssh-key.pub
```
3. Upload the key to each of the nodes (if needed)

```javascript
mkdir .ssh
chmod 700 .ssh
vi .ssh/authorized_keys
#add the public key 
chmod 600 .ssh/authorized_keys
```
4. Connect to each node and verify connectivity

```javascript
ssh -i ~/.ssh/my-ssh-key nodeaddress....
```

5. Create inventory hosts file
```
vi hosts
### exmaple of hosts file
[OSEv3:children]
nodes
masters
nfs
etcd
[OSEv3:vars]
openshift_master_default_subdomain=apps.exmple.com
### set root user name and set ansible_become= yes or set root as the username
ansible_ssh_user=username
ansible_become=yes
containerized=true
openshift_release=v1.5.1
openshift_image_tag=v1.5.1
openshift_master_cluster_method=native
### this is the ip/hosname of you LB
openshift_master_cluster_hostname=exmaple.com
#### this is the host name of the LB 
openshift_master_cluster_public_hostname=exmaple.com
deployment_type=origin
os_sdn_network_plugin_name='redhat/openshift-ovs-multitenant'
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true','kind':'HTPasswdPasswordIdentityProvider', 'filename': '/etc/origin/htpasswd'}]
openshift_docker_options='--selinux-enabled --insecure-registry 172.30.0.0/16'
openshift_router_selector='region=infra'
openshift_registry_selector='region=infra'
openshift_disable_check=docker_storage,disk_availability,memory_availability,package_availability,package_version
openshift_hosted_metrics_deploy=true
openshift_hosted_metrics_public_url=https://hawkular-metrics.apps.example/hawkular/metrics
openshift_hosted_logging_deploy=true
[nodes]
master[1:3].exmaple.com openshift_node_labels="{'region': 'infra','zone': 'default'}" openshift_schedulable=True
node1..exmaple.com  openshift_node_labels="{'region': 'primary','zone': 'default'}" openshift_schedulable=True
[masters]
master[1:3].exmaple.com
[nfs]
node1.exmaple.com
[etcd]
master[1:3].exmaple.com

# save and exit
```
## Running
1. Run the openshift-ansible script
```
ansible-playbook -i hosts ./openshift-ansible/playbooks/byo/config.yml --private-key ~/.ssh/my-ssh-key -v
```
2. verify that there are no errors

## Post installation

1. Add user 
```
# in all master nodes
sudo cd /etc/origin/master
sudo htpasswd -c -b /etc/origin/htpasswd admin 1223123
```
2. Enable the user to be cluster admin
```
#in all nodes
#oc adm policy add-cluster-role-to-user cluster-admin admin
```
## Adding NFS support
1. In the NFS node
```
#create sharefolder
mkdir /nfs
chmod 777 /nfs

#Edit or create /etc/exports
vi /etc/exports

#exmaple:
/nfsfileshare *(rw,sync,root_squash)
/nfs *(rw,sync,root_squash)

#then

chown nfsnobody:nfsnobody /nfs
chmod 755 /nfs

exportfs -r

#Make sure that the firewall is not blocking
iptables -I INPUT 1 -p tcp --dport 2049 -j ACCEPT
```
2. In the cluster admin
```
oc login -u username -n the-project-you-want-to-add
# or
oc login -u system:admin

#create pv ymal file like this

apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0001
spec:
  capacity:
    storage: 15Gi
  accessModes:
  - ReadWriteOnce
  nfs:
    path: /nfs
    server: 134.191.54.158
  persistentVolumeReclaimPolicy: Recycle

#save 
#create the pv in
oc create -f nfs-pv2.yaml 

#verify by 
oc get pv
```
## Adding new host
1. Edit your inventory file like that
```
[OSEv3:children]
masters
nodes
new_nodes # <-add this

[OSEv3:vars]
ansible_ssh_user=username
ansible_become=yes
deployment_type=origin
openshift_release=v1.5.0
openshift_image_tag=v1.5.0
......
the same
[masters]
master.exaple.com openshift_public_hostname="master.exmaple.com"
[nodes]
# master needs to be included in the node to be configured in the SDN
master.openshift.tradency-ext.com
infra.example.com openshift_node_labels="{'region': 'infra', 'zone': 'default'}"
node1.example.com.openshift_node_labels="{'region': 'primary', 'zone': 'default'}"
node2.example.com openshift_node_labels="{'region': 'primary', 'zone': 'default'}"

# add this 
[new_nodes]
node3.example.com openshift_node_labels="{'region': 'primary', 'zone': 'default'}"
```
2.
```
# run again
ansible-playbook -i hosts ./openshift-ansible/playbooks/byo/openshift-node/scaleup.yml  --private-key ~/.ssh/my-ssh-key -v
```
2. After the installation
```
# the hosts inventory should look like this
[OSEv3:children]
masters
nodes
new_nodes # <-add this

[OSEv3:vars]
ansible_ssh_user=username
ansible_become=yes
deployment_type=origin
openshift_release=v1.5.0
openshift_image_tag=v1.5.0
......
the same
[masters]
master.exaple.com openshift_public_hostname="master.exmaple.com"
[nodes]
# master needs to be included in the node to be configured in the SDN
master.openshift.tradency-ext.com
infra.example.com openshift_node_labels="{'region': 'infra', 'zone': 'default'}"
node1.example.com.openshift_node_labels="{'region': 'primary', 'zone': 'default'}"
node2.example.com openshift_node_labels="{'region': 'primary', 'zone': 'default'}"
node3.example.com openshift_node_labels="{'region': 'primary', 'zone': 'default'}"
# add this 
[new_nodes]
## leave this blank
```

# Possible problems and solutions
1. Cannot run containers that need root access
```
oc login 

#switch to project 
 oc project nginx
 oadm policy add-scc-to-user anyuid -z default

 #verify at 
 oc edit scc anyuid

 #maker sure you see such example
 upplementalGroups:
  type: RunAsAny
users:
- system:serviceaccount:default:default
- system:serviceaccount:default:dev-fullertone
- system:serviceaccount:default:demo
- system:serviceaccount:ngins:default
volumes:
- configMap
```
2. Metrics is not working - cassandra pod is crashing 
```
the problem is because all the pods are using the tag:latest , change the pods images to tag:your_open_shift_verion
```

3. You don't have root and need to create user with root rights
```
adduser username
gpasswd -a username wheel
# adding to password less sudo
visudo -f /etc/sudoers

in the file uncomment 

## Same thing without a password
%wheel  ALL=(ALL)       NOPASSWD: ALL
```


# Small Utils to debug
1. Check logs
```
journalctl -u origin-master -l
```

