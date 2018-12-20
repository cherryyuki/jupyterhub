# Setting up JupyterHub

## Prerequisite:

### 1. Setup k8s cluster and Enable dynamic storage on your Kubernetes cluster.

$ vi storageclass.yml 

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  annotations:
     storageclass.beta.kubernetes.io/is-default-class: "true"
  name: gp2
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
```

$ kubectl apply -f storageclass.yml

$ kubectl get StorageClass

This enables dynamic provisioning of disks, allowing us to automatically assign a disk per user when they log in to JupyterHub.


### 2. Install helm (Option 1)

$ curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash
$ helm init

#### Initialization
After installing helm on your machine, initialize helm on your Kubernetes cluster. At a terminal for your local machine (or within an interactive cloud shell from your provider), enter:

1.  Set up a ServiceAccount for use by Tiller, the server side component of helm.

$ kubectl --namespace kube-system create serviceaccount tiller

2.  Give the ServiceAccount RBAC full permissions to manage the cluster.
$ kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller

3.  Set up Helm on the cluster.
$ helm init --service-account tiller

# if helm is already initiated then run $ helm init --upgrade --service-account tiller


4. You can verify that you have the correct version and that it installed properly by running:
$ helm version

It should provide output like:
```
Client: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
```

# check current releases
$ helm ls



### 2. Install helm (Option 2)

wget https://storage.googleapis.com/kubernetes-helm/helm-v2.9.1-linux-amd64.tar.gz

tar -xzvf helm-v2.9.1-linux-amd64.tar.gz

sudo cp /linux-amd64/helm /usr/local/bin/

# check
$ helm

# install tilller
$ helm init


$ vi default-service-account-rbac2.yml 

```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: fabric8-rbac2
subjects:
  - kind: ServiceAccount
    # Reference to upper's `metadata.name`
    name: default
    # Reference to upper's `metadata.namespace`
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

$ kubectl create -f default-service-account-rbac2.yml


## Setting up JupyterHub:

### 1. Prepare configuration file

$ openssl rand -hex 32

$ vi config.yml

```
proxy:
  secretToken: "<OUTPUT-OF-`openssl rand -hex 32`>"
e.g.:
proxy:
  secretToken: "130b1219eeeea7c845880b8fca0695e6bbca37550b42f7ff20e6bf52e6e7e80a"
```

### 2. Set image in the config.yml

```
proxy:
  secretToken: "95914ae74757592342c8df510526b5cd40ffd3b639b2bf2fc95718083c30e7f1"
singleuser:
  image:
    name: yukizzz/pyspark-notebook
    tag: spark-2.4.0
```

### 3. Install Jupyterhub
a) Add the JupyterHub helm repository to your helm, so you can install JupyterHub from it. 
This makes it easy to refer to the JupyterHub chart without having to use a long URL each time.

$ helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
$ helm repo update

This should show output like:

```
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "jupyterhub" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈ 
```

b) Now you can install the chart! Run this command from the directory that contains the config.yaml file to spin up JupyterHub:
$ helm install jupyterhub/jupyterhub \
    --version=0.7.0  \
    --name=k8s-jupyterhub \
    --namespace=jupyterhub \
    -f config.yml

While the command is running, you can see the pods being created by entering in a different terminal:
$ kubectl --namespace=jupyterhub get pod

You can find the IP to use for accessing the JupyterHub with:
$ kubectl --namespace=default get svc proxy-public

The external IP for the proxy-public service should be accessible in a minute or two.

If the IP for proxy-public is too long to fit into the window, you can find the longer version by calling:
kubectl --namespace=jupyterhub describe svc proxy-public


![](https://github.com/cherryyuki/jupyterhub/blob/develop/kubernetes/img/ip_for_jupyterhub.png?raw=true)


To use JupyterHub, enter the external IP for the proxy-public service in to a browser. JupyterHub is running with a default dummy authenticator so entering any username and password combination will let you enter the hub.



### 4. Shutdown JupyterHub
helm delete <YOUR-HELM-RELEASE-NAME> --purge
e.g.:
helm delete k8s-jupyterhub --purge




