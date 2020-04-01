# Installation
Two clusters must be created. The install config of one of those clusters can be default while the other needs the following changes.

## Site1
For site1 default values can be used.

```
openshift-install create install-config --dir site1
```

## Site 2

For site2 the values for networking must be modified.
```
openshift-install create install-config --dir site2

vi site2/install-config.yaml

networking:
  clusterNetwork:
  - cidr: 10.132.0.0/14
    hostPrefix: 23
  machineCIDR: 10.2.0.0/16
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.32.0.0/16
```

## Machine sizes
Before creating the clusters the worker machine sizes should be increased

```
openshift-install create manifests --dir site1
openshift-install create manifests --dir site2
sed -i 's/m4.large/m4.4xlarge/g' site1/openshift/99_openshift-cluster-api_worker-machineset-*
sed -i 's/m4.large/m4.4xlarge/g' site2/openshift/99_openshift-cluster-api_worker-machineset-*
```

## Kubeconfig
Once the installation is completed modify the kubeconfig so that the context names are not the same
```
sed -i 's/admin/site1/g' kubeconfig/site1/kubeconfig 
sed -i 's/admin/site2/g' kubeconfig/site2/kubeconfig 
```
## Peering
Once the changes have been made to the kubeconfig export KUBECONFIG and setup the peering connection
```
export KUBECONFIG=/home/user/git/multisite/kubeconfig/site1/kubeconfig:/home/user/git/multisite/kubeconfig/site2/kubeconfig
./create-peer.sh
```

This should establish a VPC peer connection between the two installations.
