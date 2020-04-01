# Argo Deployment 
The steps below assume an OpenShift cluster has been deployed. The following steps will need to be done on both clusters.


## Install the ArgoCD binary

```
wget https://github.com/argoproj/argo-cd/releases/download/v1.4.2/argocd-linux-amd64
mv argocd-linux-amd64 ~/bin/argocd
chmod +x ~/bin/argocd
```

## Deploy ArgoCD

### Site 1

Export the KUBECONFIG that relates to your site1 environment.
```
export KUBECONFIG=/home/rcook/git/multisite/installation/site1/auth/kubeconfig
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
ARGOCD_SERVER_PASSWORD=$(oc -n argocd get pod -l "app.kubernetes.io/name=argocd-server" -o jsonpath='{.items[*].metadata.name}')
PATCH='{"spec":{"template":{"spec":{"$setElementOrder/containers":[{"name":"argocd-server"}],"containers":[{"command":["argocd-server","--insecure","--staticassets","/shared/app"],"name":"argocd-server"}]}}}}'
oc -n argocd patch deployment argocd-server -p $PATCH
oc -n argocd create route edge argocd-server --service=argocd-server --port=http --insecure-policy=Redirect
ARGOCD_ROUTE=$(oc -n argocd get route argocd-server -o jsonpath='{.spec.host}')
sleep 1m
argocd --insecure --grpc-web login ${ARGOCD_ROUTE}:443 --username admin --password ${ARGOCD_SERVER_PASSWORD}
argocd --insecure --grpc-web login ${ARGOCD_ROUTE}:443 --username admin --password ${ARGOCD_SERVER_PASSWORD}
argocd --insecure --grpc-web --server ${ARGOCD_ROUTE}:443 account update-password --current-password ${ARGOCD_SERVER_PASSWORD} --new-password 100Root-
```

### Site 2

Export the KUBECONFIG that relates to your site2 environment.
```
export KUBECONFIG=/home/rcook/git/multisite/installation/site2/auth/kubeconfig
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
ARGOCD_SERVER_PASSWORD=$(oc -n argocd get pod -l "app.kubernetes.io/name=argocd-server" -o jsonpath='{.items[*].metadata.name}')
PATCH='{"spec":{"template":{"spec":{"$setElementOrder/containers":[{"name":"argocd-server"}],"containers":[{"command":["argocd-server","--insecure","--staticassets","/shared/app"],"name":"argocd-server"}]}}}}'
oc -n argocd patch deployment argocd-server -p $PATCH
oc -n argocd create route edge argocd-server --service=argocd-server --port=http --insecure-policy=Redirect
ARGOCD_ROUTE=$(oc -n argocd get route argocd-server -o jsonpath='{.spec.host}')
sleep 1m
argocd --insecure --grpc-web login ${ARGOCD_ROUTE}:443 --username admin --password ${ARGOCD_SERVER_PASSWORD}
argocd --insecure --grpc-web login ${ARGOCD_ROUTE}:443 --username admin --password ${ARGOCD_SERVER_PASSWORD}
argocd --insecure --grpc-web --server ${ARGOCD_ROUTE}:443 account update-password --current-password ${ARGOCD_SERVER_PASSWORD} --new-password 100Root-
```

## Adding repository
NOTE: This must be done on both servers. If you do not have your pub key loaded into GitHub use https://github.com/cooktheryan/multisite-application.

### Site1
```
export KUBECONFIG=/home/rcook/git/multisite/installation/site1/auth/kubeconfig
ARGOCD_ROUTE=$(oc -n argocd get route argocd-server -o jsonpath='{.spec.host}')
argocd --insecure --grpc-web login ${ARGOCD_ROUTE}:443 --username admin --password ${ARGOCD_SERVER_PASSWORD}
argocd repo add git@github.com:cooktheryan/multisite-application.git --ssh-private-key-path ~/.ssh/id_rsa
```

### Site2
```
export KUBECONFIG=/home/rcook/git/multisite/installation/site2/auth/kubeconfig
ARGOCD_ROUTE=$(oc -n argocd get route argocd-server -o jsonpath='{.spec.host}')
argocd --insecure --grpc-web login ${ARGOCD_ROUTE}:443 --username admin --password ${ARGOCD_SERVER_PASSWORD}
argocd repo add git@github.com:cooktheryan/multisite-application.git --ssh-private-key-path ~/.ssh/id_rsa
```
