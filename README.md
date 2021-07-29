# JupyterhubAWS
Setting up Jupyterhub on AWS with kops/kubeadm

## Understanding Kubernetes and Jupyterhub (z2jh)

### Understanding Kubernetes
If you are well familiar with kubernetes, skip this section.

Refer to the image below, kubernetes is a container orchastration service that has a single control-plain/master to admin all activities of worker nodes and pods are containers that are running on nodes.

#### A simple metaphor
Just imagine that the cluster is a fleet, and the control-plane/master is the flag ship which does all the admin stuff but doesn't carry any cargo. Each of the worker nodes is a cargo ship of the fleet, carrying a number of containers.

As kubernetes admin, what you need to do is simply talking to the flag ship's captain(API server), then you have command over the whole fleet.

Then on each of the cargo ship(worker node), there is a network proxy that directs traffic from the node's port listener to pod, and this network proxy is what the users are talking to. Each of the pod is like mini self-sufficient operating system, has all the dependencies installed for its image to run.

![full-kubernetes-model-architecture](https://user-images.githubusercontent.com/58676681/127451109-9bb4bdd2-c6c1-44c2-8955-33d9e0566627.png)


### Understanding Jupyterhub
If you are familiar with z2jh, skip this section.

Jupyterhub is a multi-user JupyterNotebook server. It is different from many individual JupyterNotebooks because it is able to take command over a shared resource and assign those resources to each of the notebooks. That is the task that could not be completed by individual JupyterNotebooks. Furthermore, JupyterHub supports Notebook sharing and many more collaboratory activities.

Refer to the image below, each of the 'Hub', 'HTTP Proxy', 'Authenticator' are pods that are running on our nodes. Pay attention to 'Hub', it is different from kuberentes master node, these two concepts are on different layers, kubernetes master manage the whole cluster on an instance level, and jupyterhub manages the services on a pod level.

![Overview-of-JupyterHub-components-56](https://user-images.githubusercontent.com/58676681/127451116-41b4cf78-9295-49d1-b0b9-60fc1a16ca96.png)


## Kubernetes Management Software 
There are a few kubernetes management softwares out there, we will choose 2, kops and kubeadm respectively.

The reason for these two choices is that, kops is well integrated with AWS, with a lot of automation supported, where as kubeadm is able to work in internet restricted environment.

A comparison table is made below
![Comparison](/resources/comparison.png)

The key difference is that, kops is heavily dependent on the internet and it talks directly to AWS for provisioning of infrastructure(instances). Whereas kubeadm only cares about bootstrapping, making it more vibrant in an internet-restricted environment. And this difference explains why kops is able to automate cluster-building and autoscale cluster but kubeadm cannot.

## Jupyterhub in an open Internet
Just simply refer to z2jh website

https://zero-to-jupyterhub.readthedocs.io/en/latest/

### Set up kubernetes
https://zero-to-jupyterhub.readthedocs.io/en/latest/kubernetes/index.html
### Set up Jupyterhub
https://zero-to-jupyterhub.readthedocs.io/en/latest/jupyterhub/index.html
### Administration
https://zero-to-jupyterhub.readthedocs.io/en/latest/jupyterhub/index.html
### System Maintenance
1. You could export environment variables in ~/.bashrc such that you don't have to export everytime you ssh into the instance.
For example, just add those to the end of ~/.bashrc
```
    export NAME=mykubes.k8s.local
    export KOPS_STATE_STORE=s3://mykubesbucket
    export ZONES=ap-southeast-1a
    export REGION=ap-southeast-1a
```
2. Automatic Volume snapshot backup
To save the snapshot of instance in case it went down accidentally, you could recover it.
Use Amazon Data LifeCycle Manager (Amazon DLM) or AWS backup
3. Temporarily stop the cluster and relaunch
When you want to stop the cluster for maintenance or save cost, you could simply stop the cluster and relaunch. You could do this by following these steps.
```
1. In AWS EC2 console, find the autoscaling groups for master group and worker group, change both of those desired nodes and min nodes to zero.
2. Retain the hostCI node
3. The nodes except for the hostci are going to be shut down
3. When you wish to restart the cluster, in hostci run kops update --yes. (This step is going to sync the cloud configuration of cluster with local specifiction, since the local specification is still the original one)
```


### Deploy Jupyterhub

### Configure via config.yaml
### MISC

