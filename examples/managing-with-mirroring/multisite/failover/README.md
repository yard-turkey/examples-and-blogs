# Fail over from site 1 to site 2
The first step is to make a change to the blog located at wordpress.example.com(replace example.com with your GLB FQDN) this can be a comment or a new post.

NOTE: To speed this process up it may be beneficial to be logged into ArgoCD web UI or use the ArgoCD binary to force the sync to modify the replicas.

## Make changes to the repository
Make changes to your fork of the repository. 

```
cd ~/git/examples-and-blogs/examples/managing-with-mirroring/multisite-application
sed -i 's/replicas: 1/replicas: 0/g' wordpress/overlays/site1/mysql-deployment.yaml
sed -i 's/replicas: 1/replicas: 0/g' wordpress/overlays/site1/wordpress-deployment.yaml
git commit -am 'site 1 down'
git push origin master
```

## Demoting and Promoting Images During Failover
As you have seen, there is a Primary and a Secondary for Mirroring...once the Primary site goes down, it's necessary to
`demote` Site1 image and `promote` site2 image.

From Original Primary Cluster (Site1) Demote Both Images
```
  $ rbd mirror image demote <pool>/<image>
  $ rbd mirror image demote replicapool/mysql-pv-claim-rook-cephc86932ee-623b-11ea-b0b1-0a580a830212
```

From Original Secondary Cluster (Site2) Promote Both Images
```
  $ rbd mirror image promote <pool>/<image>
  $ rbd mirror image promote replicapool/mysql-pv-claim-rook-cephc86932ee-623b-11ea-b0b1-0a580a830212  
```

**[NOTE]** Even with a 1 minutes replication schedule, you will want at least 3 minutes between demotion and promotion to ensure the images, otherwise risk losing data from new Primary to Secondary. We have only seen this behavior on Failback to the Primary after we have already Failed Over once.

**[NOTE]** To help mitigate loss of data, you can always `resync` between secondary and primary to speed up the process.

- Using `resync` From current Secondary to Primary before demotion and promotion
```
  $ rbd mirror image resync replicapool/mysql-pv-claim-rook-cephc86932ee-623b-11ea-b0b1-0a580a830212
```

**[NOTE]** This flags the current Secondary to pull the current Primary latest information


## Bring site2 online
```
cd ~/git/examples-and-blogs/examples/managing-with-mirroring/multisite-application
sed -i 's/replicas: 0/replicas: 1/g' wordpress/overlays/site2/mysql-deployment.yaml
sed -i 's/replicas: 0/replicas: 1/g' wordpress/overlays/site2/wordpress-deployment.yaml
git commit -am 'site 2 up'
git push origin master
```

## Validation
Ensure that the changes to the blog are still there whether it is a comment or a new post then add yet another comment or new post.

## Back to site1
The steps above can be done in reverse order to make site1 the site running the wordpress application.
