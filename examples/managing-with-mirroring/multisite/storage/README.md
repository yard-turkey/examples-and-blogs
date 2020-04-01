# Storage setup

## AWS
On AWS an extra disk needs to be created and attached to the worker or OCS instances.

## AWS security groups
The following ports must be opened up or all traffic must be allowed between workers at site1 to site2 and vice versa.

The traffic must be opened up to the specific networks. For example site1 worker security group would need to be allowed from source 10.2.0.0/16 to the ports below. The same would be for site2 to 10.0.0.0/16	
* 6789
* 3300
* 6800-7300

## Install Rook-Ceph On Each OCP Cluster
Install [common.yaml](../workflow1/examples/common.yaml), [operator-openshift.yaml](../workflow1/examples/operator-openshift.yaml) and [cluster.yaml](../workflow1/examples/cluster.yaml), in that order.

After successful installation of the cluster (give it a few minutes and make sure all components are healthy and running) 
you will need to grab some additional information to properly execute RBD commands. Because this is a development setup for now, the toolbox.yaml pod
will not have proper access to the cluster and rbd daemon and monitors, so instead we will use the Mgr pod to interact with the cluster and RBD. Keep this information
handy as you will need it in later steps when setting up Mirroring and/or `alias` the rbd command

### Gather the Ceph Connection information

- Get Mon list from configs
```
$ oc describe cm rook-ceph-mon-endpoints
"10.0.172.36:6789","10.0.138.164:6789","10.0.156.143:6789"
```

- Get the admin key from configs and base64 decode it
```
$ oc get secrets rook-ceph-mon -o yaml
```

From this take the admin-secret field and base64 decode it
```
$ echo <the admin-secret key> | base64 -d
```

- cluster 1 Mon List and key
```
-m 10.0.172.36:6789,10.0.138.164:6789,10.0.156.143:6789
```
```
--id admin --key AQAJfmJeR00wIxAANFLosHgnLLSf6TqGZkegBA==
```

- cluster 2 Mon List and key
```
-m 10.2.136.208:6789,10.2.157.234:6789,10.2.173.142:6789
```

```
--id admin --key AQDar2JeTyRbLBAA8UZQAcS+Ao2uCns93rI7Lw==
```

## Update the Provisioner DaemonSet and Deployment with Latest Dev Images
For this setup we are using some special dev images and to get these to take
you need to update a DS and Deployment

Follow these steps for the Provisioner deployment
```
$ oc edit deploy csi-rbdplugin-provisioner
Add the following respectively to both csi-attacher and csi-provisioner containers 

Provisioner container image:
quay.io/k8scsi/csi-provisioner:canary

And also add this argument to the csi-provisioner
       --extra-create-metadata=true

attacher container image:
quay.io/k8scsi/csi-attacher:canary


And add this image to the csi-rbdplugin container
      quay.io/shyamsundarr/cephcsi:rbdmirror

```

And then update the Plugin DaemonSet
```
$ oc edit ds csi-rbdplugin
Add the following image to the csi-rbdplugin container:
      quay.io/shyamsundarr/cephcsi:rbdmirror

-- Restart the DS
$ oc rollout restart ds csi-rbdplugin
daemonset.extensions/csi-rbdplugin restarted
```

**NOTE** Wait until all the components are successfully restarted




## Create the Image Pool and StorageClass on Each Cluster
On each cluster execute the [pool.yaml](../workflow1/examples/pool.yaml), changing the pool name as you see fit.

```
  oc create -f pool.yaml
```

Also on each cluster execute the [sc-mirror.yaml](../workflow1/examples/sc-mirror.yaml).

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: rook-ceph-block-mirror <1>
provisioner: rook-ceph.rbd.csi.ceph.com <2>
parameters:
  clusterID: rook-ceph
  pool: replicapool  <3>
  imageFeatures: layering
  mirrored: "true" <4>
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
  csi.storage.k8s.io/fstype: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
  - discard
```
<1>. StorageClass name - use this in your PVC

<2>. Provisioner name (standard)

<3>. The pool name from the pool.yaml

<4>. Mirrored feature, but be true and must use quotes

```
  # oc create -f sc-mirror.yaml
```
