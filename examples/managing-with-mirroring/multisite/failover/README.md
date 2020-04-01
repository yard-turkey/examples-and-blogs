# Fail over from site 1 to site 2
The first step is to make a change to the blog located at wordpress.demo-sysdeseng.com this can be a comment or a new post.

NOTE: To speed this process up it may be beneficial to be logged into ArgoCD web UI or use the ArgoCD binary to force the sync to modify the replicas.

## Make changes to the repository
Make changes to the fork of https://github.com/cooktheryan/multisite-application or to the repository if you have access. 

```
cd ~/git/multisite-application/
sed -i 's/replicas: 1/replicas: 0/g' wordpress/overlays/site1/mysql-deployment.yaml
sed -i 's/replicas: 1/replicas: 0/g' wordpress/overlays/site1/wordpress-deployment.yaml
git commit -am 'site 1 down'
git push origin master
```

## Modify the primary storage


## Bring site2 online
```
cd ~/git/multisite-application/
sed -i 's/replicas: 0/replicas: 1/g' wordpress/overlays/site2/mysql-deployment.yaml
sed -i 's/replicas: 0/replicas: 1/g' wordpress/overlays/site2/wordpress-deployment.yaml
git commit -am 'site 2 up'
git push origin master
```

## Validation
Ensure that the changes to the blog are still there whether it is a comment or a new post then add yet another comment or new post.

## Back to site1
The steps above can be done in reverse order to make site1 the site running the wordpress application.
