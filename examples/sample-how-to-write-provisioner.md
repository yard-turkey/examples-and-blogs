## Developer Notes: How to Write a Provisioner

### Summary
This example will walk through the steps on how to create your own provisioner. For this
example we will be building the start of a Google Cloud Storage (GCS) Provisioner to help
show a real working example.

#### Prerequisites and Key Components
1. Access to a Kuberenetes Cluster
- Run local with [Minikube](https://kubernetes.io/docs/setup/minikube/).
- Run local with [hack/local-up-cluster.sh](https://github.com/kubernetes/kubernetes/blob/master/hack/local-up-cluster.sh) from [Kubernetes repo](https://github.com/kubernetes/kubernetes)
- Cloud Cluster running on AWS using [Kops](https://kubernetes.io/docs/setup/custom-cloud/kops/)
- etc...

2. Docker
- Best to install latest Docker, but this should all work with minimum of Docker 1.13.1

3. Access to a public image/application repository
- Personal Account/Repository on either [docker.io](https://hub.docker.com/) or [quay.io](https://quay.io/repository/)

4. GoLang/IDE
- go1.11.4 or higher should be fine



### Project Layout
A project can be structured in several different ways, this is just one example of a 
project model that was used with the [AWS S3 Provisioner](https://github.com/yard-turkey/aws-s3-provisioner).

#### Create a GitHub Repo
1. Login to Github and create a personal repository for your project. Our team uses a generic repo where we
typically put dev projects called [yard-turkey](https://github.com/yard-turkey). And before you ask...yard-turkey has
no special meaning other than the guy who created it had a bunch of wild turkeys in his back yard during that time.

```
   Good repo should be easy to remmember and find:
   https://github.com/<team repo>/<specific app/project name> i.e. /yard-turkey/aws-s3-provisioner
   
   You can also use your personal repo for dev if you don't have a formal or team repo to use
   https://github.com/<github user>/<specific app/project name> i.e. /screeley44/aws-s3-provisioner
```


#### Sample Directory Structure

1. Build your repo locally or in github
- From your local GOPATH build the common src/github.com directory structure
```
  # mkdir -p <$GOPATH>/src/github.com/<repo>/<app>
```

2. Create the basic directory structure from the Repo Root directory. The below example is by no means the
only way to structure a provisioner, the key is to do what makes sense for you and your project.

```

        --> Repo Root dir
            --> cmd (source directory for main package go files)
                --> awss3provisioner.go
                --> util.go
                --> etc...
            --> bin (binary directory for builds)
            --> hack (scripts and utilities directory if needed)
            --> examples or docs (directory for documentation if needed)
            README.md (start up README)
            
```

3. Create a generic .gitignore

```
# Logs and archives
*.log
*.tar.*
*.zip

# Binaries for programs and plugins
*.exe
*.exe~
*.dll
*.so
*.dylib
bin/*

# Test binary, build with `go test -c`
*.test

# Output of the go coverage tool, specifically when used with LiteIDE
*.out

# IDE config
.idea*
```


### Code the Provisioner
Now the basic project structure is in place, you can begin building the provisioner. We will add a
<provisioner>.go file in the <Repo Root>/cmd/ directory.

#### Sample Code Template



#### Dependency Management
The AWS S3 Provisioner used go modules [vgo](https://github.com/golang/vgo) for dependency management. You can also use [Dep](https://golang.github.io/dep/docs/installation.html), but I think
VGO is more aligned with the long term direction of Go Dependency Management.

1. Install vgo
```
 # go get -u golang.org/x/vgo.
```

2. Initialize vgo
```
 # vgo build
 # vgo update
```

3. Create and Manage the vendor directory
```
# vgo mod vendor
```
**[NOTE]** This will pull in all the project dependencies and create the <Repo Root/*vendor* directory.
If your imports and dependencies change, just rerun the above command.


### Testing Provisioner
As mentioned above, if you don't have a test cluster available and running, now is the time to get one running.
See links above for some guidance on how one might go about doing that.

#### Build and Test Local Binary

1. Build the provisioner binary.
```
 # go build -a -o ./bin/<provisioner name>  ./<source dir>/...

 i.e.
 # go build -a -o ./bin/aws-s3-provisioner  ./cmd/...
```

2. Install the [OB/OBC CRDs](https://github.com/yard-turkey/lib-bucket-provisioner/blob/master/deploy/customResourceDefinitions.yaml) on your cluster.


3. Push the binary to a remote cluster to test or run it on your local cluster if you have one.
```
# scp /bin/awss3provisioner <user>@<kube-cluster-host>:~
```

Run the provisioner (after the CRD's are created) passing in *master* and *kubeconfig* parameters. (assumes a simple local-up-cluster.sh implemenation)
```
 # ./awss3provisioner -master https://localhost:6443 -kubeconfig /var/run/kubernetes/admin.kubeconfig -alsologtostderr -v=2

I0403 10:30:40.881043   16396 aws-s3-provisioner.go:458] AWS S3 Provisioner - main
I0403 10:30:40.881264   16396 aws-s3-provisioner.go:459] flags: kubeconfig="/var/run/kubernetes/admin.kubeconfig"; masterURL="https://localhost:6443"
I0403 10:30:40.883873   16396 manager.go:75] objectbucket.io "level"=0 "msg"="new provisioner"  "name"="aws-s3.io/bucket"
I0403 10:30:40.884624   16396 manager.go:87] objectbucket.io "level"=2 "msg"="generating controller manager"  
I0403 10:30:40.923627   16396 manager.go:94] objectbucket.io "level"=2 "msg"="adding schemes to manager"  
I0403 10:30:40.923742   16396 reconiler.go:54] objectbucket.io/reconciler/aws-s3.io/bucket "level"=0 "msg"="constructing new reconciler"  "provisioner"="aws-s3.io/bucket"
I0403 10:30:40.923766   16396 reconiler.go:59] objectbucket.io/reconciler/aws-s3.io/bucket "level"=2 "msg"="retry loop setting"  "RetryBaseInterval"=10000000000
I0403 10:30:40.923779   16396 reconiler.go:63] objectbucket.io/reconciler/aws-s3.io/bucket "level"=2 "msg"="retry loop setting"  "RetryTimeout"=360000000000
I0403 10:30:40.923791   16396 manager.go:132] objectbucket.io "level"=0 "msg"="building controller manager"  
I0403 10:30:40.924741   16396 aws-s3-provisioner.go:472] main: running aws-s3.io/bucket provisioner...
I0403 10:30:40.924763   16396 manager.go:150] objectbucket.io "level"=0 "msg"="Starting manager"  "provisioner"="aws-s3.io/bucket"
```

If using a real cluster, like Kops, passing in the *master* and *kubeconfig* parameters.
```
 # kops export kubecfg --name=<kops cluster name from output>
 i.e.
 # kops export kubecfg --name=screeley-s3prov.screeley.sysdeseng.com
 #./awss3provisioner -master https://api.screeley-s3prov.screeley.sysdeseng.com -kubeconfig /home/centos/.kube/config -alsologtostderr -v=2
```


#### Build and Test Docker Image
1. Add a simple Dockerfile to the project in the <Repo Root> directory.

```
FROM fedora:29

COPY ./bin/aws-s3-provisioner /usr/bin/

ENTRYPOINT ["/usr/bin/aws-s3-provisioner"]
CMD ["-v=2", "-alsologtostderr"]
```

2. Login to docker and quay.io.
```
 # docker login
 # docker login quay.io
```

3. Build the image and push it to quay.io or docker.io.
```
 # docker build . -t quay.io/<your_quay_account>/aws-s3-provisioner:v1.0.0
 # docker push quay.io/<your_quay_account>/aws-s3-provisioner:v1.0.0
```

i.e.

```
 # docker build . -t quay.io/screeley44/aws-s3-provisioner:v1.0.0
 # docker push quay.io/screeley44/aws-s3-provisioner:v1.0.0
```

4. You can now create a deployment or pod to test your image and provisioner.
