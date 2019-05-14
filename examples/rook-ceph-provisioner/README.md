## Rook-Ceph S3 Provisioner - Dynamically Create _New_ Bucket OR Access Existing Bucket OBC Examples
This example will walk through the basic steps needed to dynamically provision
a new AWS S3 Bucket. The end result is that application pods have read-write access to the new bucket via a Kubernetes Secret and ConfigMap.

The Rook-Ceph S3 provisioner utilizes a Kubernetes [CustomResourceDefinition](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
that defines the ObjectBucket (OB) and ObjectBucketClaim (OBC) resources for your cluster. This design
pattern mimics the Kubernetes PersistentVolume and PersistentVolumeClaim model.

### Table Of Contents
1. [Assumptions](#assumptions)
1. [Deploy or Run AWS S3 Provisioner on Cluster](#deploy-or-run-aws-s3-provisioner-on-cluster)
1. [Administrator Creates Secret](#administrator-creates-secret)
1. [Administrator Creates StorageClass](#administrator-creates-storageclass)
1. [User Creates ObjectBucketClaim](#user-creates-objectbucketclaim)
1. [Results and Recap](#results-and-recap)
1. [User Creates Pod](#user-creates-pod)

### Assumptions
This example assumes some familiarity with Kubernetes and Rook-Ceph and that a Kubernetes
cluster is available to use. This example also breaks the work flow operations into 
two basic use cases; Administrator and Developer/Application Owner.

### Deploy or Run Rook-Ceph S3 Provisioner on Cluster

1. Follow [these instructions](https://rook.io/docs/rook/v1.0/ceph-quickstart.html) on how to quickly deploy the Rook-Ceph Cluster and Operator framework on your cluster.

```
# kubectl get pods -n rook-ceph
NAME                                                       READY   STATUS      RESTARTS   AGE
rook-ceph-agent-dmfnj                                      1/1     Running     0          1d
rook-ceph-mgr-a-659b66f777-7tph9                           1/1     Running     0          1d
rook-ceph-mon-a-7f756f6df-t7rzv                            1/1     Running     0          4d
rook-ceph-mon-b-7d786c4cbc-l8b5g                           1/1     Running     0          4d
rook-ceph-mon-c-84bdc7554c-w25ws                           1/1     Running     0          4d
rook-ceph-operator-7d8c75665d-b8pr9                        1/1     Running     0          1d
rook-ceph-osd-0-855b5cf8c9-jpvtq                           1/1     Running     0          1d
rook-ceph-osd-prepare-ip-172-20-55-80.ec2.internal-zzkhz   0/2     Completed   0          1d
rook-ceph-rgw-screeley-store-86b7c7bb4f-fjph5              1/1     Running     0          1d
rook-discover-5v92n                                        1/1     Running     0          1d

```

2. Create a ClusterRoleBinding for the rook-ceph service account.
```
# kubectl create clusterrolebinding cluster-admin-rook --clusterrole=cluster-admin --user=system:serviceaccount:rook-ceph:rook-ceph-system
```

3. Create the ObjectStore and ObjectStoreUser.

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: screeley-store
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 1
  dataPool:
    replicated:
      size: 1
  gateway:
    type: s3
    port: 80
    securePort:
    instances: 1
    allNodes: false

```

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStoreUser
metadata:
  name: screeley
  namespace: rook-ceph
spec:
  store: screeley-store
  displayName: "screeley's object store"
```


### Administrator Creates StorageClass
The StorageClass defines the name of the provisioner and holds other properties that are needed to provision a new bucket, including
the Owner Secret and Namespace, and the AWS Region.

#### Greenfield Example:
For Greenfield, a new, dynamic bucket will be generated.

1. Create the Kubernetes StorageClass for the Provisioner.
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: rook-buckets [1]
provisioner: ceph.rook.io/bucket [2]
parameters:
  objectStoreNamespace: rook-ceph [3]
  objectStoreName: screeley-store [4]
reclaimPolicy: Delete [5]
```
1. Name of the StorageClass, this will be referenced in the User ObjectBucketClaim.
1. Provisioner name
1. Namespace of the objectStore created above
1. Name of the objectStore created above
1. reclaimPolicy (Delete or Retain) indicates if the bucket can be deleted when the OBC is deleted.

**NOTE:** the absence of the `bucketName` Parameter key in the storage class indicates this is a new bucket and its name is based on the bucket name fields in the OBC.

```
 # kubectl create -f storageclass-greenfield.yaml
storageclass.storage.k8s.io/rook-buckets created
```

#### Brownfield Example:

For brownfield, the StorageClass defines the name of the provisioner and the name of the existing bucket. It also includes other properties needed by the target
provisioner, including: the Owner Secret and Namespace, and the AWS Region

1. Create the Kubernetes StorageClass for the Provisioner.
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: ceph-rook-brown-class <1>
provisioner: ceph.rook.io/bucket
parameters:
  BUCKET_NAME: screeley-bucket <2>
  objectStoreNamespace: rook-ceph <3>
  objectStoreName: screeley-store <4>
reclaimPolicy: Retain

```
1. Name of the StorageClass, this will be referenced in the User ObjectBucketClaim.
1. Provisioner name
1. Name of the existing bucket
1. Namespace of the objectStore
1. Name of the objectStore created above

**NOTE:** the storage class's `reclaimPolicy` is ignored for existing buckets.

```
 # kubectl create -f storageclass-brownfield.yaml
storageclass.storage.k8s.io/s3-buckets created
```

### User Creates ObjectBucketClaim
An ObjectBucketClaim follows the same concept as a PVC, in that
it is a request for Object Storage, the user doesn't need to
concern him/herself with the underlying storage, just that
they need access to it. The user will work with the cluster/storage
administrator to get the proper StorageClass needed and will
then request access via the OBC.

#### Greenfield Request Example:

1. Create the ObjectBucketClaim.
```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: myobc [1]
  namespace: s3-provisioner [2]
spec:
  generateBucketName: mybucket [3]
  bucketName: my-awesome-bucket [4]
  storageClassName: s3-buckets [5]
```
1. Name of the OBC
1. Namespace of the OBC
1. Name prepended to a random string used to generate a bucket name. It is ignored if bucketName is defined
1. Name of new bucket which must be unique across all AWS regions, otherwise an error occurs when creating the bucket. If present, this name overrides `generateName`
1. StorageClass name

**NOTE:** if both `generateBucketName` and `bucketName` are omitted, and the storage class does _not_ define a bucket name, then a new, random bucket name is generated with no prefix.
```
 # kubectl create -f obc-brownfield.yaml
objectbucketclaim.objectbucket.io/myobc created
```
#### Brownfield Request Example:

1. Create the ObjectBucketClaim.
```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: myobc [1]
  namespace: s3-provisioner [2]
spec:
  storageClassName: s3-existing-buckets [3]
```
1. Name of the OBC
1. Namespace of the OBC
1. StorageClass name

**NOTE:** in the OBC here there is no reference to the bucket's name. This is defined in the storage class and is not a concern of the user creating the claim to this bucket.  An OBC does have fields for defining a bucket name for greenfield use only.
```
 # kubectl create -f obc-brownfield.yaml
objectbucketclaim.objectbucket.io/myobc created
```

### Results and Recap
Let's pause for a moment and digest what just happened.
After creating the OBC, and assuming the S3 provisioner is running, we now have
the following Kubernetes resources:
.  a global ObjectBucket (OB) which contains: bucket endpoint info (including region and bucket name), a reference to the OBC, and a reference to the storage class. Unique to S3, the OB also contains the bucket Amazon Resource Name (ARN).Note: there is always a 1:1 relationship between an OBC and an OB.
.  a ConfigMap in the same namespace as the OBC, which contains the same endpoint data found in the OB.
.  a Secret in the same namespace as the OBC, which contains the AWS key-pairs needed to access the bucket.

And of course, we have a *new* AWS S3 Bucket which you should be able to see via the AWS Console.


*ObjectBucket*
```yaml
 # kubectl get ob obc-s3-provisioner-my-awesome-bucket -o yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucket
metadata:
  creationTimestamp: "2019-04-03T15:42:22Z"
  generation: 1
  name: obc-s3-provisioner-my-awesome-bucket
  resourceVersion: "15057"
  selfLink: /apis/objectbucket.io/v1alpha1/objectbuckets/obc-s3-provisioner-my-awesome-bucket
  uid: 0bfe8e84-576d-4c4e-984b-f73c4460f736
spec:
  Connection:
    additionalState:
      ARN: arn:aws:iam::<accountid>:policy/my-awesome-bucket-vSgD5 [1]
      UserName: my-awesome-bucket-vSgD5 [2]
    endpoint:
      additionalConfig: null
      bucketHost: s3-us-west-1.amazonaws.com
      bucketName: my-awesome-bucket [3]
      bucketPort: 443
      region: us-west-1
      ssl: true
      subRegion: ""
  claimRef: null [4]
  reclaimPolicy: null
  storageClassName: s3-buckets [5]
```
1. The AWS Policy created for this user and bucket.
1. The new user generated by the Provisioner to access this existing bucket.
1. The bucket name.
1. The reference to the OBC (not filled in yet).
1. The reference to the StorageClass used.


*ConfigMap*
```yaml
 # kubectl get cm myobc -n s3-provisioner -o yaml
apiVersion: v1
data:
  BUCKET_HOST: s3-us-west-1.amazonaws.com [1]
  BUCKET_NAME: my-awesome-bucket [2]
  BUCKET_PORT: "443"
  BUCKET_REGION: us-west-1
  BUCKET_SSL: "true"
  BUCKET_SUBREGION: ""
kind: ConfigMap
metadata:
  creationTimestamp: "2019-04-01T19:11:38Z"
  finalizers:
  - objectbucket.io/finalizer
  name: my-awesome-bucket
  namespace: s3-provisioner
  resourceVersion: "892"
  selfLink: /api/v1/namespaces/s3-provisioner/configmaps/my-awesome-bucket
  uid: 2edcc58a-aff8-4a29-814a-ffbb6439a9cd
```
1. The AWS S3 host.
1. The name of the new bucket we are gaining access to.


*Secret*
```yaml
 # kubectl get secret my-awesome-bucket -n s3-provisioner -o yaml
apiVersion: v1
data:
  AWS_ACCESS_KEY_ID: *the_new_access_id* [1]
  AWS_SECRET_ACCESS_KEY: *the_new_access_key_value* [2]
kind: Secret
metadata:
  creationTimestamp: "2019-04-03T15:42:22Z"
  finalizers:
  - objectbucket.io/finalizer
  name: my-awesome-bucket
  namespace: s3-provisioner
  resourceVersion: "15058"
  selfLink: /api/v1/namespaces/s3-provisioner/secrets/screeley-provb-5
  uid: 225c71a5-9d75-4ccc-b41f-bfe91b272a13
type: Opaque
```
1. The new generated AWS Access Key ID.
1. The new generated AWS Secret Access Key.

What happened in AWS? The first thing we do on any OBC request is
create a new IAM user and generate Access ID and Secret Keys.
This allows us to also better control ACLs. We also create a policy
in IAM which we then attach to the user and bucket. We also created a new bucket, called *my-awesome-bucket*. 

When the OBC is deleted all of its Kubernetes and AWS resources will also be deleted, which includes:
the generated OB, Secret, ConfigMap, IAM user, and policy.
If the _retainPolicy_ on the StorageClass for this bucket is *"Delete"*, then, in addition to the above cleanup, the physical bucket is also deleted.  

**NOTE:** The actual bucket is only deleted if the Storage Class's _reclaimPolicy_ is "Delete".

### User Creates Pod
Now that we have our bucket and connection/access information, a pod
can be used to access the bucket. This can be done in several different
ways, but the key here is that the provisioner has provided the proper
endpoints and keys to access the bucket. The user then simply references
the keys.

1. Create a Sample Pod to Access the Bucket.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: photo1
  labels:
    name: photo1
spec:
  containers:
  - name: photo1
    image: docker.io/screeley44/photo-gallery:latest
    imagePullPolicy: Always
    envFrom:
    - configMapRef:
        name: my-awesome-bucket <1>
    - secretRef:
        name: my-awesome-bucket <2>
    ports:
    - containerPort: 3000
      protocol: TCP

```
1. Name of the generated configmap from the provisioning process
1. Name of the generated secret from the provisioning process

*[Note]* Generated ConfigMap and Secret are same name as the OBC!

Lastly, expose the pod as a service so you can access the url from a browser. In this example,
I exposed as a LoadBalancer

``` 
  # kubectl expose pod photo1 --type=LoadBalancer --name=photo1 -n your-namespace
```

To access via a url use the EXTERNAL-IP

```
  # kubectl get svc photo1
  NAME                         TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)          AGE
  photo1                       LoadBalancer   100.66.124.105   a00c53ccb3c5411e9b6550a7c0e50a2a-2010797808.us-east-1.elb.amazonaws.com   3000:32344/TCP   6d
```

**NOTE:** This is just one example of a Pod that can utilize the bucket information,
there are several ways that these pod applications can be developed and therefore
the method of getting the actual values needed from the Secrets and ConfigMaps
will vary greatly, but the idea remains the same, that the pod consumes the generated
ConfigMap and Secret created by the provisioner.



