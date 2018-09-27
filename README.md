# Developing a Kubernetes Application with Local and Remote Clusters

As a developer you want to be able to rapidly iterate on your application source code locally while still mirroring a remote production environment as closely as possible.
This tutorial describes how to set up a single-node Kubernetes cluster locally using Minikube and deploy an application to it and then do the same steps with the IBM Kubernetes Service on the IBM Cloud.
It then describes how to set up a multi-node cluster locally using kubeadm-dind-cluster and remotely with the IBM Kubernetes Service.

# Objectives

Learn how to perform the following tasks.
* Create a local single-node cluster using Minikube.
* Create a local multi-node cluster using kubeadm-dind-cluster.
* Create remote single-node and multi-node clusters using IBM Cloud Kubernetes Service.
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
   * Docker is a tool that allows developers to build and run an application as a lightweight, portable container.
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

# Setting up a local single-node cluster using Minikube

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

If you are working with other Kubernetes clusters and change your kubectl CLI context to use another cluster,
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

The guestbook application listens on port `3000` inside the pod.  In order to make the application externally accessible
we need to create a Kubernetes service of type NodePort for it.  Kubernetes will allocate a port in the range `30000-32767`
and the node will proxy that port to the pod's target port.

```console
$ kubectl expose deployment guestbook --type=NodePort --port=3000
service/guestbook exposed
```

In order to access the service, we need to know the IP address of Minikube's virtual machine and the node port number.
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


# Setting up a remote single-node cluster using IBM Cloud Kubernetes Service

Let's now look at continuing our application development on the IBM Cloud.
If you have not already registered for an IBM Cloud account, do so [here](https://console.bluemix.net/registration/).
The steps in this section can be done with a free Lite account.  (The cluster that you create will expire after one month.)

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

export KUBECONFIG=/home/gregd/.bluemix/plugins/container-service/clusters/mycluster/kube-config-hou02-mycluster.yml
```

Copy the `export` statement and run it.  This sets the `KUBECONFIG` environment variable to point to the kubectl config file.
This will make your `kubectl` client work with your new Kubernetes cluster.  You can verify that by entering a `kubectl` command.

```console
$ kubectl get nodes
NAME            STATUS    ROLES     AGE       VERSION
10.76.202.250   Ready     <none>    4m        v1.10.7+IKS
```

## Deploying the application to your remote cluster
<a id="create-registry"></a>

In order for Kubernetes to pull images to run in the cluster, the images need to be stored in an accessible registry.
You can use the IBM Cloud Container Service to push docker images to your own private registry.

First add a namespace to create your own image repository.
A namespace is a unique name to identify your private image registry.
Replace `<my_namespace>` with your preferred namespace.

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

Before we can push an image into the registry, we need to run the `ibmcloud cr login` command
to log your local Docker daemon into IBM Cloud Container Registry.

```console
ibmcloud cr login
```

<a id="push-registry"></a>
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

The guestbook application listens on port `3000` inside the pod.  In order to make the application externally accessible
we need to create a Kubernetes service of type NodePort for it.  Kubernetes will allocate a port in the range `30000-32767`
and the node will proxy that port to the pod's target port.

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

In this case the public IP is `173.193.75.82`.

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

In this case the NodePort is `32146`.

So in this case the application can be accessed from a browser using the URL `http://173.193.75.82:32146/`.


# Setting up a local multi-node cluster using kubeadm-dind-cluster

More advanced application development may require a multi-node cluster.
We are going to use the project `kubernetes-sigs/kubeadm-dind-cluster` to set up a multi-node Kubernetes cluster.
It uses "Docker in Docker" to simulate multiple Kubernetes nodes in a single machine environment.

## Installing kubeadm-dind-cluster

kubeadm-dind-cluster is written to run in Linux.  If you have a Windows or Mac workstation, you need to set up a
virtual machine running Linux.  The recommended approach is to use
[VirtualBox](https://www.virtualbox.org/wiki/Downloads) and run an
[Ubuntu](https://www.ubuntu.com/download/desktop) guest virtual machine.
You also need to install [Docker](https://docs.docker.com/install/) in the guest virtual machine.

kubeadm-dind-cluster provides preconfigured scripts to set up Kubernetes clusters at various version levels.
For this tutorial we'll use the `dind-cluster-v1.10.sh` script.  Obtain the script as follows.

```console
wget https://cdn.rawgit.com/kubernetes-sigs/kubeadm-dind-cluster/master/fixed/dind-cluster-v1.10.sh
```

## Creating a cluster

Start a cluster by running the `dind-cluster-v1.10.sh` script with the `up` option.

```console
./dind-cluster-v1.10.sh up
```

By default the script creates a Kubernetes master node and two worker nodes.
Each node is running inside a separate Docker container.

The script configures the kubectl CLI to work with this cluster.
You first have to add the kubectl CLI to your path.
Then you can verify your cluster nodes by entering the `kubectl get nodes` command.

```console
$ export PATH="$HOME/.kubeadm-dind-cluster:$PATH"
$ kubectl get nodes
NAME          STATUS    ROLES     AGE       VERSION
kube-master   Ready     master    5m        v1.10.5
kube-node-1   Ready     <none>    3m        v1.10.5
kube-node-2   Ready     <none>    4m        v1.10.5
```

## Deploying the application to your cluster

The Kubernetes cluster needs to be able to pull images from somewhere to run in the cluster.
With minikube we could point the Docker CLI to the minikube's docker daemon and avoid the need
for a registry.  However we now have multiple docker daemons running nested within docker containers
and synchronizing images across them is challenging and error-prone.
It makes more sense to work within the Kubernetes model and use a registry.

Let's see how to set up the cluster to use the private registry created using the IBM Cloud Container Service.
Earlier you learned [how to create your own image repository using the `ibmcloud` CLI](#create-registry).
Now you need to use the ibmcloud CLI to create a token to grant access to your IBM Cloud Container Registry namespaces.

```console
$ ibmcloud cr token-add --description "token for kubeadm dind cluster access" --non-expiring --readwrite
Requesting a registry token...

Token identifier   58669dd6-3ddd-5c78-99f9-ad0a5aabd9ad
Token              <token_value>
```

This creates a non-expiring token that has read and write access to all namespaces.
The actual token appears in place of `<token_value>`.
Every user in possession of this token can push and pull images to and from your namespaces.

Next create a Kubernetes secret to hold the token value.
Secrets are intended to hold sensitive information.
You will need to substitute the following values into this command:
* \<region>
    * You can find this out by running the `ibmcloud cr region` command.

    ```console
    $ ibmcloud cr region
    You are targeting region 'us-south', the registry is 'registry.ng.bluemix.net'.
    OK
    ```
    In this case you would substitute `ng` into the registry address.
* \<token_value>
    * This is the `<token_value>` that was output when you created the registry token above.
* \<email>
    * You can provide any email address.  This argument is required by this `kubectl` command but is not used for anything.

```console
$ kubectl --namespace default create secret docker-registry registrysecret --docker-server=registry.<region>.bluemix.net --docker-username=token --docker-password=<token_value> --docker-email=<email>
secret "registrysecret" created
```

Finally we need to have Kubernetes use this secret when pulling images.  The easiest way to do so is to add this secret
to the default Kubernetes service account.  A service account represents an identity for processes that run in a pod.
If a pod doesn't have an assigned service account, it uses the default service account.

```console
kubectl patch -n default serviceaccount/default -p '{"imagePullSecrets":[{"name": "registrysecret"}]}'
```

This command tells Kubernetes to use the `registrysecret` that you created to authenticate with your private registry
when it needs to pull an image.  Let's try it now.

```console
$ kubectl run guestbook --image=registry.<region>.bluemix.net/<my_namespace>/guestbook:v1.1
deployment.apps/guestbook created
```

Use kubectl to verify that Kubernetes created a pod containing our container and that it's running.

```console
$ kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
guestbook-5986549d9-h5n74    1/1       Running   0          1m
```

## Accessing the running application

The guestbook application listens on port `3000` inside the pod.  In order to make the application externally accessible
we need to create a Kubernetes service of type NodePort for it.  Kubernetes will allocate a port in the range `30000-32767`
and the node will proxy that port to the pod's target port.

```console
$ kubectl expose deployment guestbook --type=NodePort --port=3000
service/guestbook exposed
```

In order to access the service, we need to know the IP address of one of the worker nodes
and the node port number that Kubernetes assigned to the service.

Get the IP address as follows:

```console
$ docker exec -it kube-node-1 ip addr show eth0
15: eth0@if16: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:12:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.18.0.3/16 brd 172.18.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

This command displays the eth0 interface in the `kube-node-1` container.
The container's IP address appears following `inet` and in this case is `172.18.0.3`.

Get the node port number as follows:

```console
$ kubectl describe services/guestbook
Name:                     guestbook
Namespace:                default
Labels:                   run=guestbook
Annotations:              <none>
Selector:                 run=guestbook
Type:                     NodePort
IP:                       10.100.222.15
Port:                     <unset>  3000/TCP
TargetPort:               3000/TCP
NodePort:                 <unset>  32345/TCP
Endpoints:                10.244.2.3:3000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

In this case the NodePort is 32345.

So in this case the application can be accessed from a browser running on the Linux host
using the URL `http://172.18.0.3:32345/`.
Note that although we are using the IP address of kube-node-1, the application does not need
to be running on that node for this URL to work.  Each node proxies the NodePort
into the Service.

## Scaling the number of pods

Of course the reason for having a cluster is to increase capacity and improve availability by placing copies of an
application on multiple nodes.  Kubernetes calls these copies "replicas".  Let's tell Kubernetes that we want two replicas
of the application.

```console
$ kubectl scale --replicas=2 deployment guestbook
deployment.extensions "guestbook" scaled
```

Kubernetes works in the background to start an additional pod to make the total number of pods equal to 2.
You can check the status of this by running the `kubectl get deployments` command.

```console
$ kubectl get deployments
NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
guestbook   2         2         2            2           10m
```

You can also see the status of the pods and which nodes they are running on as follows.

```console
$ kubectl get pods -o wide
NAME                        READY     STATUS    RESTARTS   AGE       IP           NODE
guestbook-5986549d9-8dnhb   1/1       Running   0          7s        10.244.3.5   kube-node-2
guestbook-5986549d9-qtnmg   1/1       Running   0          7s        10.244.2.5   kube-node-1
```

<a id="scale-exercise"></a>
Once both pods are running, try the following exercise.  Open the application in a browser, enter a few messages, and close the browser.
Repeat again a few times.  You will see the display of previous guest messages changes because the browser is
connecting to one node or the other and the messages are kept in memory.  (You have to close the browser and
re-open it to force a new connection, otherwise it will remain connected to the same node.)

Of course a real guestbook application would keep the messages in a backing store.
The [IBM Cloud Container Service Lab](https://github.com/IBM/kube101/tree/master/workshop)
shows how to add a redis database to guestbook.


# Setting up a remote multi-node cluster using IBM Cloud Kubernetes Service

Let's now look at continuing our application development on a multi-node cluster in the IBM Cloud.

In order to create a multi-node cluster (referred to as a standard cluster) you must have either a Pay-As-You-Go or a Subscription account.
See https://console.bluemix.net/docs/account/index.html#accounts for further information about account types.

Log in to the IBM Cloud CLI and enter your IBM Cloud credentials when prompted.

```console
ibmcloud login
```

Note: If you have a federated ID, use `ibmcloud login --sso` to log in to the IBM Cloud CLI.

## Creating a cluster

It is recommended to use the [IBM Cloud dashboard](https://console.bluemix.net/catalog/) to create a standard cluster because it will help guide you through
the configuration options and it will show you the estimated cost.
* Click the login button in the upper right and follow the login prompts to log into your account.
* Click the `Containers` tab on the left side of the window and then select the "Kubernetes Service" tile.
* Click the `Create` button.
* Fill in the form that appears.  First select a location for your cluster as this will dictate the other options.
Then choose a zone within the location and a machine type.  Set the number of worker nodes to 2.
Give a name to your cluster; for this tutorial we'll use "myStandardCluster".
* Review the cost estimate on the right side of the window.
* Click the `Create Cluster` button.


Cluster creation continues in the background.  You can check the status as follows.
```console
$ ibmcloud ks clusters
OK
Name                ID                                 State     Created          Workers   Location    Version
myStandardCluster   fc5514ef25ac44da9924ff2309020bb3   normal    12 minutes ago   2         Dallas     1.10.7_1520
```

If the cluster state is `pending`, wait for a moment and try the command again.
Once the cluster is provisioned (state is `normal`), the kubernetes client CLI `kubectl` needs to be configured to talk to the provisioned cluster.
Run `ibmcloud ks cluster-config myStandardCluster` which will create a config file on your workstation.

```console
$ ibmcloud ks cluster-config myStandardCluster
OK
The configuration for mycluster was downloaded successfully. Export environment
variables to start using Kubernetes.

export KUBECONFIG=/home/gregd/.bluemix/plugins/container-service/clusters/myStandardCluster/kube-config-hou02-myStandardCluster.yml
```

Copy the `export` statement and run it.  This sets the `KUBECONFIG` environment variable to point to the kubectl config file.
This will make your `kubectl` client work with your new Kubernetes cluster.  You can verify that by entering a `kubectl` command.

```console
$ kubectl get nodes
NAME            STATUS    ROLES     AGE       VERSION
10.177.184.185   Ready     <none>    2m        v1.10.7+IKS
10.177.184.220   Ready     <none>    2m        v1.10.7+IKS
```

## Deploying the application to your remote cluster

Let's continue to use the guestbook application that we [pushed into a private registry](#push-registry).
Your new cluster automatically has access to this registry.
Note that if you created your registry under the trial or Lite plan, it remains under that plan and is subject to certain quotas.
See this [page](https://console.bluemix.net/docs/services/Registry/registry_overview.html) for more information about registry quotas
and how to upgrade the registry service plan.

```console
$ kubectl run guestbook --image=registry.<region>.bluemix.net/<my_namespace>/guestbook:v1.1
deployment.apps/guestbook created
```

We can use kubectl to verify that Kubernetes created a pod containing our container and that it's running.

```console
$ kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
guestbook-7b9f8cf696-fg8v8   1/1       Running   0          1m
```

## Accessing the running application

The guestbook application listens on port `3000` inside the pod.  In order to make the application externally accessible
we need to create a Kubernetes service of type NodePort for it.  Kubernetes will allocate a port in the range 30000-32767
and the node will proxy that port to the pod's target port.

```console
$ kubectl expose deployment guestbook --type=NodePort --port=3000
service/guestbook exposed
```

(Note that with a standard cluster you can also make use of either a LoadBalancer service or an Ingress resource.
These concepts are beyond the scope of this tutorial.)

In order to access the service, we need to know the IP address of one of the worker nodes
and the node port number that Kubernetes assigned to the service.

Get the IP address as follows:

```console
$ ibmcloud ks workers myStandardCluster
OK
ID                                                 Public IP        Private IP       Machine Type        State    Status   Zone    Version
kube-dal10-crfc5514ef25ac44da9924ff2309020bb3-w1   169.47.252.42    10.177.184.220   u2c.2x4.encrypted   normal   Ready    dal10   1.10.7_1520
kube-dal10-crfc5514ef25ac44da9924ff2309020bb3-w2   169.48.165.242   10.177.184.185   u2c.2x4.encrypted   normal   Ready    dal10   1.10.7_1520
```

You can use the public IP address of either node.  Each node proxies the NodePort into the Service.

Get the node port number as follows:

```console
$ kubectl describe services/guestbook
Name:                     guestbook
Namespace:                default
Labels:                   run=guestbook
Annotations:              <none>
Selector:                 run=guestbook
Type:                     NodePort
IP:                       172.21.210.103
Port:                     <unset>  3000/TCP
TargetPort:               3000/TCP
NodePort:                 <unset>  31096/TCP
Endpoints:                172.30.108.133:3000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

In this case the NodePort is 31096.

So in this case the application can be accessed from a browser using the URL `http://169.47.252.42:31096/` or `http://169.48.165.242:31096`.

## Scaling the number of pods

We can tell Kubernetes that we want two replicas just as we did in the local multi-node cluster.

```console
$ kubectl scale --replicas=2 deployment guestbook
deployment.extensions "guestbook" scaled
```

Kubernetes works in the background to start an additional pod to make the total number of pods equal to 2.
You can check the status of this by running the `kubectl get deployments` command.

```console
$ kubectl get deployments
NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
guestbook   2         2         2            2           10m
```

You can also see the status of the pods and which nodes they are running on as follows.

```console
$ kubectl get pods -o wide
NAME                         READY     STATUS    RESTARTS   AGE       IP               NODE
guestbook-7b9f8cf696-dzgcb   1/1       Running   0          14s       172.30.58.204    10.177.184.185
guestbook-7b9f8cf696-hvrc7   1/1       Running   0          14s       172.30.108.136   10.177.184.220
```

You can repeat the previous [exercise](#scale-exercise) to connect to the pod running in each node.

# Cleaning up

After you have completed the tutorial, if you no longer need the resources that you created then you can delete them.

You can clean up the Kubernetes resources that you created in your clusters as follows.

* Set the `kubectl` CLI context for the cluster that you want to clean up.

  For minikube, use `kubectl config use-context minikube`.
  
  For IBM Kubernetes Service, use `ibmcloud ks cluster-config mycluster` or `ibmcloud ks cluster-config myStandardCluster` and copy and paste the
  setting of the KUBECONFIG environment variable.

* Delete the deployment and its service by entering the following commands.

```console
$ kubectl delete deployment guestbook
$ kubectl delete service guestbook
```

To delete your minikube cluster, enter the following commands:

```console
$ minikube stop
$ minikube delete
```

To delete your kubeadm-dind-cluster, enter the following commands:

```console
./dind-cluster-v1.10.sh down
./dind-cluster-v1.10.sh clean
```

To delete your IBM Kubernetes Service clusters, enter the following commands:

```console
ibmcloud ks cluster-rm myCluster
ibmcloud ks cluster-rm myStandardCluster
```
