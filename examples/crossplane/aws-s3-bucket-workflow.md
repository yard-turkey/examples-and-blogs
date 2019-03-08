## Crossplane - AWS S3 Bucket Creation and Usage Workflow Example
This example will walk through the steps needed to provision and use 
an AWS S3 bucket using [Crossplane](https://github.com/crossplaneio/crossplane) multi-cloud 
control plane.

### Table Of Contents
1. [Assumptions](#assumptions)
1. [Install Crossplane](#install-crossplane)
1. [Create Provider](#create-provider)
1. [Create Resource Class](#create-resource-class)
1. [Create Resource Claim](#create-resource-claim)
1. [Create Photo Application](#create-photo-application)


### Assumptions
This example assumes some familiarity with Kubernetes and AWS and the following is available to use:

+ Access to AWS system with aws-cli installed and configured with proper account and credentials that have S3 permsissions
+ Kubernetes cluster is running and accessible on AWS, if not
you can install using [kops](https://kubernetes.io/docs/setup/custom-cloud/kops/) or [minikube](https://kubernetes.io/docs/setup/minikube/) or [eks](https://aws.amazon.com/eks/getting-started/), etc...
+ [Helm](https://helm.sh/docs/using_helm/) installed on the cluster.

### Install Crossplane
1. Using Helm [install](https://crossplane.io/docs/v0.1/install-crossplane.html) and enable the Crossplane repo.
1. Use master (not alpha).
1. If using Master install path, make sure to use exact version that is available, to see available versions:

```
  # helm search crossplane
  NAME                            CHART VERSION        APP VERSION          DESCRIPTION                                  
  crossplane-alpha/crossplane     0.1.0                0.1.0                Crossplane - Managed Cloud Resources Operator
  crossplane-master/crossplane    0.0.0-377.946c2c8    0.0.0-377.946c2c8    Crossplane - Managed Cloud Resources Operator
```
1. Verify Crossplane is running - you should see the Crossplane operator and CRDs.

```
  # kubectl get pods -n crossplane-system
  NAME                          READY   STATUS    RESTARTS   AGE
  crossplane-5b6686474c-72gz6   1/1     Running   0          1d

  # kubectl get crd
  NAME                                                CREATED AT
  aksclusters.compute.azure.crossplane.io             2019-03-06T18:38:22Z
  azurebuckets.storage.azure.crossplane.io            2019-03-06T18:38:22Z
  buckets.storage.crossplane.io                       2019-03-06T18:38:23Z
  s3buckets.storage.aws.crossplane.io                 2019-03-06T18:38:23Z
  ...
```

### Create Provider
A provider is a resource to manage cloud access and credentials.

1. Create your secret using the already configured .aws credentials.

```
  # kubectl create secret generic aws-provider-creds --from-file=credentials=/root/.aws/credentials -n crossplane-system
```

1. Create the provider yaml and execute it

```yaml
  apiVersion: aws.crossplane.io/v1alpha1
  kind: Provider
  metadata:
    name: aws-provider [1]
    namespace: crossplane-system
  spec:
    credentialsSecretRef:
      key: credentials
      name: aws-provider-creds [2]
    region: us-west-1 [3]
```
1. Name of the provider
1. Name of the secret
1. Specifiy the region for this provider

### Create Resource Class
The Crossplane Resource Class is like a template that defines implementation or deployment details about
a particular Resource and typically created by an administrator.

```yaml
apiVersion: core.crossplane.io/v1alpha1
kind: ResourceClass
metadata:
  name: west-s3buckets [1]
  namespace: crossplane-system
parameters:
  versioning: "false"
  predefinedACL: private [2]
  region: us-west-1
provisioner: s3bucket.storage.aws.crossplane.io/v1alpha1 [3] 
providerRef:
  name: aws-provider [4]
reclaimPolicy: Delete [5]
```
1. Name of the Resource Class to be referenced later by the Resource Claim.
1. Default Access Control List for the bucket (Private, PublicRead, PublicReadWrite, AuthenticatedRead)
1. Provisioner for s3bucket
1. Reference to the AWS Provider that was created in the previous step
1. Delete or Retain

### Create Resource Claim
The Resource Claim is a request for a specific resource type and is typically created by a developer or application owner.

1. Create and execute the following yaml.

```yaml
apiVersion: storage.crossplane.io/v1alpha1
kind: Bucket
metadata:
  name: my-bucket
spec:
  name: my-bucket [1]
  predefinedACL: Private
  localPermission: ReadWrite
  classReference:
    name: west-s3buckets [2]
    namespace: crossplane-system
```
1. Name of the S3 bucket that will be provisioned.
1. Reference to the Resource Class created in the previous step


### Create Photo Application
Now we will use the S3 Bucket we just provisioned by creating a demo photo application and running it in our cluster.

1. After the bucket is created, a secret is also created from the provisioner, this secret holds the bucket
connection information such as endpoint, user and password. We will need this secret to run our app pod.

```
  # kubectl get secret my-bucket
  NAME                   TYPE     DATA   AGE
  my-bucket              Opaque   3      20h
```

1. Create and execute the following yaml using the secret that was generated in previous step.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: photo1
  labels:
    name: photo1
spec:
  containers:
  - env:
    - name: BUCKET_NAME
      value: "my-bucket" [1]
    - name: BUCKET_ID [2]
      valueFrom:
            secretKeyRef:
              name: my-bucket
              key: username
    - name: BUCKET_PWORD [2]
      valueFrom:
            secretKeyRef:
              name: my-bucket
              key: password
    - name: OBJECT_STORAGE_S3_TYPE
      value: "aws"
    - name: OBJECT_STORAGE_CLUSTER_PORT
      value: "443"
    - name: OBJECT_STORAGE_REGION
      value: "us-west-1"
    image: docker.io/zherman/demo:latest
    imagePullPolicy: Always
    name: photo1
    ports:
    - containerPort: 3000
      protocol: TCP
```
1. Name of the bucket that was provisioned
1. Using the generated secret to get the proper credentials

Lastly, expose the pod as a service so you can access the url from a browser. In this example,
I exposed as a LoadBalancer

``` 
  # kubectl expose pod photo1 --type=LoadBalancer --name=photo1
```

To access via a url use the EXTERNAL-IP

```
  # kubectl get svc photo1
  NAME                         TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)          AGE
  photo1                       LoadBalancer   100.66.124.105   a00c53ccb3c5411e9b6550a7c0e50a2a-2010797808.us-east-1.elb.amazonaws.com   3000:32344/TCP   6d
```
 




