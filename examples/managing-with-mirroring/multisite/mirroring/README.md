# DR Demo Example - Ceph RBD Mirroring Steps
This is some short-hand commands and steps to setup mirroring and demonstrate
the commands needed to perform the failover demo.

For more information see [official rbd mirroring documentation](https://docs.ceph.com/docs/master/rbd/rbd-mirroring/)


**[NOTE]** This assumes you have [Setup the Storage](../storage/README.md) including the StorageClass and Pool! and that the Primary Cluster (Site1) has the WP + MySQL App running
and has generated the proper rbd images!


## Using Mgr Pod and Setting an Alias
Each time you login to the Mgr pod, you can set the alias of the rbd command
so you don't have to keep including the parameters with the commands.
See [Setup Storage](../../storage/README.md) section to gather this information.

- Login and set alias in Mgr Pod
```
  # oc exec -ti <mgr pod> /bin/sh
  $ alias rbd="rbd -m 10.2.136.208:6789,10.2.157.234:6789,10.2.173.142:6789 --id admin --key AQDar2JeTyRbLBAA8UZQAcS+Ao2uCns93rI7Lw=="
```

## Enable Mirroring on the Image Pool and Bootstrap the Cluster Peers
*Assumes shelled into the Mgr pod and Alias has been set and pool is created*

- On each cluster enable `image` mirroring
```
  $ rbd mirror pool enable replicapool image
  $ rbd pool init replicapool
```
**[NOTE]** Ignore the ugly messages after each rbd command - they are result of dev images and not using the toolbox pod

- Bootstrap the Cluster Peers - from Primary Cluster (Site1)
```
  $ rbd mirror pool peer bootstrap create --site-name site1 replicapool
  <produces key output which you will copy and use in next step>
```

- Import Bootstrapped Peer Key - from Secondary Cluster (Site2)
```
  -- Create token file from Key
  cat <<EOF > token
<The Key Here>
EOF

  -- Import
  $ rbd mirror pool peer bootstrap import --site-name site2 replicapool token


  -- Check status from each cluster
  $ rbd mirror pool info replicapool --all

```

## Enable Mirroring on the Image
For this demo we used `snapshot` mirroring which is a point in time that
can be scheduled down to every 1 minute. This also assumes that the WP + MySQL 
app is running on Site1 and the images have been created.

- To see images

```
  $ rbd ls replicapool
  mysql-pv-claim-rook-cephc86932ee-623b-11ea-b0b1-0a580a830212   
  wp-pv-claim-rook-cephc8dfdae7-623b-11ea-b0b1-0a580a830212
```

- Enable Snapshot Mirroring on both images from Primary Cluster (Site1)

```
  rbd mirror image enable replicapool/mysql-pv-claim-rook-cephc86932ee-623b-11ea-b0b1-0a580a830212 snapshot
  rbd mirror image enable replicapool/wp-pv-claim-rook-cephc8dfdae7-623b-11ea-b0b1-0a580a830212 snapshot
```

**[NOTE]** DO NOT enable the images on the Secondary Cluster (Site2)

- Set a Replication Schedule on BOTH clusters

```
  $ rbd mirror snapshot schedule add --pool replicapool --image wp-pv-claim-rook-cephc8dfdae7-623b-11ea-b0b1-0a580a830212 1m
  $ rbd mirror snapshot schedule add --pool replicapool --image mysql-pv-claim-rook-cephc8dfdae7-623b-11ea-b0b1-0a580a830212 1m
```

- View Schedule

```
  $ rbd mirror snapshot schedule ls --pool replicapool --recursive
POOL         NAMESPACE  IMAGE                                                         SCHEDULE
demopool             mysql-pv-claim-rook-cephc86932ee-623b-11ea-b0b1-0a580a830212  every 1m
demopool             wp-pv-claim-rook-cephc8dfdae7-623b-11ea-b0b1-0a580a830212     every 1m
```
