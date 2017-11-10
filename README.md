About
-----

In this doc I will explain how we can use dynamic volume provisioning in Kubernetes with [GlusterFS](https://www.gluster.org/) and [Heketi API](https://github.com/heketi/heketi)

We're going to create heketi api as pod on kubernetes and we're going to use ssh executor for heketi api to communicate with glusterfs nodes, all glusterfs servers are going to be installed on baremetal hosts directly, and that's why we'll be using ssh instead of kubernetes executor.

It is very important that only one replica of heketi is created and running at any time, and this is required because of heketi db on which only one instance of heketi can have read-write permissions (lock acquired), also if pod is terminated, kubernetes replication controller will make sure that new heketi pod is up and running.


Install
-------

#### GlusterFS

GlusterFS server has to be installed on all Kubernetes nodes (excluding master nodes)

Add latest glusterfs repo
```
apt install software-properties-common
add-apt-repository ppa:gluster/glusterfs-3.10
```

Install dependencies
```
apt install thin-provisioning-tools
```

Finally install `glusterfs-server`
```
apt install glusterfs-server glusterfs-client
```

Check `glusterfs-server` status
```
systemctl status glusterfs-server.service
```

Add label on all Kubernetes nodes with GlusterFS
```
kubectl label node node1 storagenode=glusterfs
kubectl label node node2 storagenode=glusterfs
kubectl label node node3 storagenode=glusterfs
```

#### Heketi CLI

Heketi CLI can be installed wherever you want, preferably on Kubernetes master nodes or on your local machine

```bash
HEKETI_BIN="heketi-cli"      # heketi or heketi-cli
HEKETI_VERSION="5.0.0"       # latest heketi version => https://github.com/heketi/heketi/releases
HEKETI_OS="linux"            # linux or darwin

curl -SL https://github.com/heketi/heketi/releases/download/v${HEKETI_VERSION}/heketi-v${HEKETI_VERSION}.${HEKETI_OS}.amd64.tar.gz -o /tmp/heketi-v${HEKETI_VERSION}.${HEKETI_OS}.amd64.tar.gz && \
tar xzvf /tmp/heketi-v${HEKETI_VERSION}.${HEKETI_OS}.amd64.tar.gz -C /tmp && \
rm -vf /tmp/heketi-v${HEKETI_VERSION}.${HEKETI_OS}.amd64.tar.gz && \
cp /tmp/heketi/${HEKETI_BIN} /usr/local/bin/${HEKETI_BIN}_${HEKETI_VERSION} && \
rm -vrf /tmp/heketi && \
cd /usr/local/bin && \
ln -vsnf ${HEKETI_BIN}_${HEKETI_VERSION} ${HEKETI_BIN} && cd

unset HEKETI_BIN HEKETI_VERSION HEKETI_OS
```


Configure
---------

#### Heketi API on Kubernetes

Create required directories on all glusterfs servers
```
mkdir -p /data/heketi/{db,.ssh} && chmod 700 /data/heketi/.ssh
```

Generate heketi ssh keys that will be used by heketi api for password-less login to glusterfs servers
```
ssh-keygen -t rsa -b 2048 -f /data/heketi/.ssh/id_rsa
```

Copy `.ssh` dir to all glusterfs servers
```
for NODE in node1 node2 node3; do scp -r /data/heketi/.ssh root@${NODE}:/data/heketi; done
```

Import ssh public key to all glusterfs servers (node1, node2 and node3 in our example)
```
for NODE in node1 node2 node3; do cat /data/heketi/.ssh/id_rsa.pub | ssh root@${NODE} "cat >> /root/.ssh/authorized_keys"; done
```

There are several parameters in [heketi-deployment.json](./kubernetes/heketi-deployment.json) you have to pay attention to before provisioning heketi on Kubernetes

Pod will be instantly terminated, which will lead to instant heketi db lock release and this is exactly what we want
```json
"terminationGracePeriodSeconds": 0,
```

Heketi pod will be scheduled to run only on GlusterFS nodes (if you recall we already created a label for this)
```json
"nodeSelector": {
  "storagenode": "glusterfs"
}
```

Heketi admin key (password) will be stored in secret object that we must create before heketi deployment and storageclass objects
```json
{
  "name": "HEKETI_ADMIN_KEY",
  "valueFrom": {
    "secretKeyRef": {
      "name": "heketi-secret",
      "key": "key"
    }
  }
}
```

First we have to create Secret object that will store password for heketi authentication  
Make sure to replace value of `key` with based64 encoded password in [heketi-secret.yaml](./kubernetes/heketi-secret.yaml) file  
This password will be later on collected directly from this secret and defined as value of `HEKETI_ADMIN_KEY` environment variable, see [heketi-deployment.json](./kubernetes/heketi-deployment.json) file and also it will be called from storageclass object as well  
This is the only place where you have to set this password
```
kubectl apply -f kubernetes/heketi-secret.yaml
```

Create heketi api deployment and service objects
```
kubectl apply -f kubernetes/heketi-deployment.json
```

Find exposed port to access heketi api  
**Note:** Replace `resturl` in [heketi-storageclass.yaml](./kubernetes/heketi-storageclass.yaml)
```
kubectl get svc -l glusterfs=heketi-service
```

Test heketi api (replace node name and port number)
```
curl -s http://node1:30348/hello
```

Test heketi api with heketi cli (replace node name and port number)  
**Note:** output should be empty because we haven't created any cluster yet
```
heketi-cli --user admin --secret password --server http://node1:30348 cluster list
```

#### Heketi topology

Create a cluster from topology file  
**Note:** change parameters in the topology file to mirror your environment
```
heketi-cli --user admin --secret password --server http://node1:30348 topology load --json heketi-topology.json
```

Replace `clusterid` in [heketi-storageclass.yaml](./kubernetes/heketi-storageclass.yaml)
```
heketi-cli --user admin --secret password --server http://node1:30348 cluster list
```

Show current cluster topology
```
heketi-cli --user admin --secret password --server http://node1:30348 topology info
```

#### Heketi DB

Create glusterfs shared volume
```
heketi-cli --user admin --secret password --server http://node1:30348 setup-openshift-heketi-storage --listfile heketi-storage.json
```

Migrate Heketi database to previously created glusterfs volume
```
kubectl apply -f heketi-storage.json
```

If job is successful you can remove all created migration objects, otherwise you should try recreating them until success status is reached

Check status
```
kubectl get job -o wide
```

Delete temporary objects if success status is reached
```
kubectl delete -f heketi-storage.json
```

or try again
```
kubectl delete -f heketi-storage.json
kubectl apply -f heketi-storage.json
```

Now we have heketi database migrated to `heketidbstorage` glusterfs replicated volume and now we can remove local heketi database from all glusterfs servers and mount glusterfs volume to shared location `/data/heketi/db`
```
for NODE in node1 node2 node3; do ssh root@${NODE} "rm -f /data/heketi/db/heketi.db && mount.glusterfs ${NODE}:/heketidbstorage /data/heketi/db"; done
```

In order to make mount persistent we have to append following lines into the `/etc/rc.local` file on all glusterfs nodes (in our example node1, node2 and node3)

Example how it should look on node1
```bash
# Mount GlusterFS on boot
sleep 5
mount.glusterfs node1:/heketidbstorage /data/heketi/db
```

#### Kubernetes StorageClass object

Create SC with glusterfs provisioner
Make sure to replace `resturl` and `clusterid` parameters as mentioned above
```
kubectl apply -f kubernetes/heketi-storageclass.yaml
```

#### Kubernetes PersistentVolumeClaim

Creation of PVC will automatically trigger creation of PV
```
kubectl apply -f kubernetes/heketi-pvc.yaml
kubectl get pvc test-claim
kubectl get pv
```

Show volume list
```
heketi-cli --user admin --secret password --server http://node1:30348 volume list
```

#### Pod with PVC

```
kubectl apply -f kubernetes/nginx-pod.yaml
```
