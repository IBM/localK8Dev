# Developing a Kubernetes Application with Local and Remote Clusters

As a developer you want to be able to rapidly iterate on your application source code locally while still mirroring a remote production environment as closely as possible.
This tutorial describes how to deploy a Kubernetes application locally using Minikube and then how to deploy it to the Kubernetes Service on the IBM Cloud.

# Objectives

Learn how to perform the following tasks.
* Create a local cluster using Minikube and a remote cluster using IBM Cloud Kubernetes Service.
* Make images available to your local and remote clusters.
* Access your running application in your local and remote clusters.
* Modify and re-deploy your application.

# Prerequisites

Before you begin, you need to install the required CLIs to create and manage your Kubernetes clusters and to deploy containerized applications to your cluster.
IBM provides an installer [here](https://clis.ng.bluemix.net/ui/home.html) to get all of these tools together.  There are instructions for how to
obtain the tools manually if desired.  The following tools are used in this tutorial.
* git
   * git is a version control system which we'll use to obtain the source of a sample application.
* Docker
   * Docker is a tool that allows developers to build and run anapplication as a lightweight, portable container.
* kubectl CLI
   * `kubectl` is a command line interface for running commands against Kubernetes clusters.
* ibmcloud CLI
   * `ibmcloud` is a command line interface for managing resources in IBM Cloud.

# Download the Sample Application
The application shown in this tutorial is a simple guestbook website where users can post messages.  You should clone it to your workstation since you will be building it locally.

```console
$ git clone https://github.com/IBM/guestbook.git
```

For the purposes of this tutorial the application is run without a backing database (i.e. data is stored in-memory).

# Deploying the application to a local cluster using Minikube

Minikube is a tool that makes it easy to run Kubernetes locally. Minikube runs a single-node Kubernetes cluster inside a VM on your workstation.

## Installing Minikube

Follow the instructions [here](https://kubernetes.io/docs/tasks/tools/install-minikube/) to install Minikube on your workstation.

## Creating a cluster

Use the `minikube start` command to start a cluster.  It creates a virtual machine on your workstation and installs Kubernetes onto it.
Note that if you are using a hypervisor other than VirtualBox then you will need to pass an additional argument `--vm-driver` to identify the appropriate VM driver.

```console
$ minikube start
Starting local Kubernetes v1.10.0 cluster...
Starting VM...
Downloading Minikube ISO
 160.27 MB / 160.27 MB  100.00% 0ss
Getting VM IP address...
Moving files into cluster...
Downloading kubelet v1.10.0
Downloading kubeadm v1.10.0
Finished Downloading kubelet v1.10.0
Finished Downloading kubeadm v1.10.0
Setting up certs...
Connecting to cluster...
Setting up kubeconfig...
Starting cluster components...
Kubectl is now configured to use the cluster.
Loading cached images from config file.
```
 
minikube configures the kubectl CLI to work with this cluster.  You can verify that by entering a `kubectl` command.

```console
$ kubectl get nodes
NAME       STATUS    ROLES     AGE       VERSION
minikube   Ready     master     1m       v1.10.0
```

If you are working with other Kubenetes clusters and change your kubectl CLI context to use another cluster, 
you can restore the minikube context by using the command `kubectl config use-context minikube`.

## Deploying the application to your local cluster

Kubernetes deployments are designed to pull images from a container registry.  However for local development you can
avoid the extra step of pushing images to a registry by pointing your docker CLI to the docker daemon running inside minikube.

```console
$ eval $(minikube docker-env)
```

If you now enter `docker ps` you will see all of the containers running inside minikube.

Note:  The effect of the `eval` command is limited to the current command window.  If you close and re-open the window
then you will need to repeat this command.

Let's go ahead and build the guestbook application.

```console
$ cd guestbook/v1/guestbook
$ docker build -t guestbook:v1 .
Sending build context to Docker daemon   16.9kB
Step 1/13 : FROM golang as builder
latest: Pulling from library/golang
05d1a5232b46: Pull complete
5cee356eda6b: Pull complete
89d3385f0fd3: Pull complete
80ae6b477848: Pull complete
94ebfeaaddf3: Pull complete
dd697213ec59: Pull complete
715d1281dfe7: Pull complete
Digest: sha256:e8e4c4406217b415c506815d38e3f8ac6e05d0121b19f686c5af7eaadf96f081
Status: Downloaded newer image for golang:latest
 ---> fb7a47d8605b
Step 2/13 : RUN go get github.com/codegangsta/negroni
 ---> Running in 6867267a6832
Removing intermediate container 6867267a6832
 ---> 41cdd0ea4dee
Step 3/13 : RUN go get github.com/gorilla/mux github.com/xyproto/simpleredis
 ---> Running in 6f928ff5ec97
Removing intermediate container 6f928ff5ec97
 ---> 2c51f4a80903
Step 4/13 : COPY main.go .
 ---> 919077d83e2b
Step 5/13 : RUN go build main.go
 ---> Running in 5959c30f5c10
Removing intermediate container 5959c30f5c10
 ---> cbe90ddb11ed
Step 6/13 : FROM busybox:ubuntu-14.04
ubuntu-14.04: Pulling from library/busybox
a3ed95caeb02: Pull complete
300273678d06: Pull complete
Digest: sha256:7d3ce4e482101f0c484602dd6687c826bb8bef6295739088c58e84245845912e
Status: Downloaded newer image for busybox:ubuntu-14.04
 ---> d16744963217
Step 7/13 : COPY --from=builder /go//main /app/guestbook
 ---> 67c829eb8d1b
Step 8/13 : ADD public/index.html /app/public/index.html
 ---> e71b4653ca17
Step 9/13 : ADD public/script.js /app/public/script.js
 ---> 1e3e52db271f
Step 10/13 : ADD public/style.css /app/public/style.css
 ---> b5b7fcfa50ca
Step 11/13 : WORKDIR /app
Removing intermediate container b982fa510f1c
 ---> 15947fedd20a
Step 12/13 : CMD ["./guestbook"]
 ---> Running in 156afadc0459
Removing intermediate container 156afadc0459
 ---> 0205392b8cbf
Step 13/13 : EXPOSE 3000
 ---> Running in 448df38be767
Removing intermediate container 448df38be767
 ---> 9708eb5366ec
Successfully built 9708eb5366ec
Successfully tagged guestbook:v1
```

Note that the image was given the tag `v1`.
You should not use the `latest` tag because this will make Kubernetes try to pull the image from a public registry.
We want it to use the locally stored image.

The guestbook image is now present in the minikube cluster so we're ready to run it.

```console
$ kubectl run guestbook --image=guestbook:v1
deployment.apps/guestbook created
```

We can use kubectl to verify that Kubernetes created a pod containing our container and that it's running.

```console
$ kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
guestbook-6cd549c68f-d6qvw   1/1       Running   0          1m
```

## Accessing the running application

In order to make the application accessible we need to create a service for it.
The guestbook application listens on port 3000.

```console
$ kubectl expose deployment guestbook --type=NodePort --port=3000
service/guestbook exposed
```

In order to access the service, we need to know the IP address of the virtual machine and the node port number.
Minikube provides a convenient way for getting this information.

```console
$ minikube service guestbook --url
http://192.168.99.104:30571
```

The IP address and port number on your workstation obviously may be different.
You can copy and paste the url that you get into your browser and the guestbook application should appear.
You can also leave off the `--url` option and minikube will open your default browser with the url for you.

## Modifying the application

Let's make a simple change to the application and redeploy it.
Open the file `public/script.js` in vi or your favorite editor.
Modify the `handleSubmission` function so that it has an additional statement to append the date to the entry value, as shown below.

```javascript
  var handleSubmission = function(e) {
    e.preventDefault();
    var entryValue = entryContentElement.val()
    if (entryValue.length > 0) {
      entryValue += " " + new Date();     // ADD THIS LINE
      entriesElement.append("<p>...</p>");
      $.getJSON("rpush/guestbook/" + entryValue, appendGuestbookEntries);
	  entryContentElement.val("")
    }
    return false;
  }
```

Now rebuild the docker image and assign it a new tag.

```console
$ docker build -t guestbook:v1.1 .
```

After the image is built, we need to tell Kubernetes to use the new image.

```console
$ kubectl set image deployment/guestbook guestbook=guestbook:v1.1
deployment.extensions/guestbook image updated
```

Refresh the guestbook application in your browser.
(You may have to tell your browser to reload the page to get the updated javascript file.)
Try entering something in the form and clicking submit.
You should see the text you entered followed by the current time appear on the page.


# Deploying the application to a remote cluster using IBM Cloud Kubernetes Service

Let's now look at continuing our application development on the IBM Cloud.
If you have not already registered for an IBM Cloud account, do so [here](https://console.bluemix.net/registration/).
The steps in this tutorial can be done with a free account.

Log in to the IBM Cloud CLI and enter your IBM Cloud credentials when prompted.

```console
ibmcloud login
```

Note: If you have a federated ID, use `ibmcloud login --sso` to log in to the IBM Cloud CLI. 

## Creating a cluster

Now let's create a Kubernetes cluster:

```console
$ ibmcloud ks cluster-create --name mycluster
The 'machine-type' flag was not specified. So a free cluster will be created.
Creating cluster...
OK
```

Cluster creation continues in the background.  You can check the status as follows.
```console
$ ibmcloud ks clusters
OK
Name        ID                                 State     Created         Workers   Location   Version
mycluster   ae7148d3c8e74d69b3ed94b6c5f02262   normal    4 minutes ago   1         hou02      1.10.7_1520
```

If the cluster state is `pending`, wait for a moment and try the command again.
Once the cluster is provisioned (state is `normal`), the kubernetes client CLI `kubectl` needs to be configured to talk to the provisioned cluster.  
Run `ibmcloud ks cluster-config mycluster` which will create a config file on your workstation.

```console
$ ibmcloud ks cluster-config mycluster
OK
The configuration for mycluster was downloaded successfully. Export environment
variables to start using Kubernetes.

export KUBECONFIG=/C/Users/IBM_ADMIN/.bluemix/plugins/container-service/clusters/mycluster/kube-config-hou02-mycluster.yml
```

Copy the `export` statement and run it.  This sets the `KUBECONFIG` environment variable to point to the kubectl config file.
This will make your `kubectl` client work with your new Kubernetes cluster.  You can verify that by entering a `kubectl` command.

```console
$ kubectl get nodes
NAME            STATUS    ROLES     AGE       VERSION
10.76.202.250   Ready     <none>    4m        v1.10.7+IKS
```

## Deploying the application to your remote cluster

In order for Kubernetes to pull images to run in the cluster, the images need to be stored in an accessible registry.
You can use the IBM Cloud Container Service to push docker images to your own private registry.

First add a namespace to create your own image repository. 
A namespace is a unique name to identify your private image registry. 
Replace <my_namespace> with your preferred namespace.

```console
$ ibmcloud cr namespace-add <my_namespace>
```

The created registry name has the format `registry.<region>.bluemix.net/<your_namespace>`
where `<region>` depends upon the region where your IBM Cloud account was created.
You can find this out by running the `ibmcloud cr region` command.

```console
$ ibmcloud cr region
You are targeting region 'us-south', the registry is 'registry.ng.bluemix.net'.

OK
```

We can now tag our current local image (which we built previously while deploying to minikube) to associate it with the private registry
and push it to the registry.  Be sure to substitute `<region>` and `<my_namespace>` with the proper values.

```console
$ docker tag guestbook:v1.1 registry.<region>.bluemix.net/<my_namespace>/guestbook:v1.1
$ docker push registry.<region>.bluemix.net/<my_namespace>/guestbook:v1.1
````

To run the application we use the same `kubectl run` command as before except now we refer to the image in the private repository.

```console
$ kubectl run guestbook --image=registry.<region>.bluemix.net/<my_namespace>/guestbook:v1.1
deployment.apps/guestbook created
```

We can use kubectl to verify that Kubernetes created a pod containing our container and that it's running.

```console
$ kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
guestbook-5986549d9-2f49g    1/1       Running   0          1m
```

## Accessing the running application

In order to make the application accessible we need to create a service for it.
The guestbook application listens on port 3000.

```console
$ kubectl expose deployment guestbook --type=NodePort --port=3000
service/guestbook exposed
```

In order to access the service, we need to know the public IP address of the node where the application is running
and the node port number that Kubernetes assigned to the service.

Get the IP address as follows:

```console
$ ibmcloud ks workers mycluster
OK
ID                                                 Public IP       Private IP    Machine Type   State    Status   Zone    Version
kube-hou02-paae7148d3c8e74d69b3ed94b6c5f02262-w1   173.193.75.82   10.76.202.250 free           normal   Ready    hou02   1.10.7_1520
```

In this case the public IP is 173.193.75.82.


Get the node port number as follows:

```console
$ kubectl describe services/guestbook
Name:                     guestbook
Namespace:                default
Labels:                   run=guestbook
Annotations:              <none>
Selector:                 run=guestbook
Type:                     NodePort
IP:                       172.21.192.189
Port:                     <unset>  3000/TCP
TargetPort:               3000/TCP
NodePort:                 <unset>  32146/TCP
Endpoints:                172.30.151.71:3000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

In this case the NodePort is 32146.

So in this case the application can be accessed from a brower using the URL `http://173.193.75.82:32146/`.