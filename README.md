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

The key difference is that, kops is heavily dependent on the internet and it talks directly to AWS for provisioning of infrastructure(e.g. instances and storage). Whereas kubeadm only cares about bootstrapping, making it more vibrant in an internet-restricted environment. And this difference explains why kops is able to automate cluster-building and autoscale cluster but kubeadm cannot.

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

* In AWS EC2 console, find the autoscaling groups for master group and worker group, change both of those desired nodes and min nodes to zero.
* Retain the hostCI node
* The nodes except for the hostci are going to be shut down
* When you wish to restart the cluster, in hostci run kops update --yes. (This step is going to sync the cloud configuration of cluster with local specifiction, since the local specification is still the original one)

### AutoScaling and use of spot instance group

Please refer to https://medium.com/paul-zhao-projects/the-ultimate-guide-to-deploying-kubernetes-cluster-on-aws-ec2-spot-instances-using-kops-and-eks-eecbb5988792 for detailed set up of both autoscaling and spot ig.

Particularly refer to sections autoscaling and spot ig
#### Spot instance
1. Spot instance is much cheaper than on-demand, but it is not always available. So if we coud use two instance groups as our worker groups, one with on-demand instances and one with spot instances. We always request for spot instance first, if unavailable we request for on-demand instance. In this way we could exploit both the availability and discount pricing.
2. In order to change what type of instance we want, we use ``` kops edit ig ``` and set nodeLabels -> ```on-demand: "true"/"false"```.
3. If spot instance group is chosen, we need to configure what is the ```maxPrice: "0.0120"```, refer to the aws ec2 pricing before set the value, set the value slightly higher than current spot instance pricing.
4. We will always request for spot instances first before requesting for on-demand instances, this could be configured by setting taints for on-demand ig
```
  taints:
  - on-demand=true:PreferNoSchedule
```

#### AutoScaling
1. Use kops edit ig to enter specification file
2. As for autoscaling, it is mainly the configuration of instance group ```maxSize: 7``` and ```minSize: 1```
3. Also use cloudLabel
```
  cloudLabels:
    k8s.io/cluster-autoscaler/enabled: ""
    k8s.io/cluster-autoscaler/node-template/label: ""
```
4. It is also required to create a policy and attach it to instance group

### Persistent Volume
There are a few options for using persistent volume
1. Use Amazon S3 by default, this is what used when following the official z2jh documentation set up. Each user's jupyternotebook will create a block volume under ec2-> volume of the size that you specify.
2. Use local storage
3. Use an exported NFS storage (e.g. Amazon EFS or isilon), the general procedure is as follows
* Create AWS EFS and create corresponding access point
* Try mount the EFS via access point
* Umount 
* Kubectl apply pv and pvc file in the reference

pv.yaml (change the server address, path and metadata name)
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-persist
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: fs-00de7740.efs.ap-southeast-1.amazonaws.com  #Notice the id and region
    path: "/"
```
pvc.yaml
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: efs-persist
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```
* Edit config file and run helm upgrade
```
singleuser:
  storage:
    type: "static"
    static:
      pvcName: "efs-persist"
      subPath: 'home/{username}'
  extraEnv:
    CHOWN_HOME: 'yes'
  uid: 0
  fsGid: 0
  cmd: "start-singleuser.sh"
```
* You can further specify mount options like this
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-storage
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  mountOptions:
    - bg
    - hard
    - nfsvers=3
    - tcp
    - intr
    - rsize=8192
    - wsize=8192
    - retry=1024
  nfs:
    server: fs-00de7740.efs.ap-southeast-1.amazonaws.com
    path: "/mydir"
```

### Common Errors and bugs
1. When you encounter 
```
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```
or 
```
no context set in kubecfg
```
Run with --state and --kubeconfig
```
kops export kubecfg --admin --kubeconfig ~/workspace/kubeconfig --state=s3://mykubesbucket
```
or 
```
export KOPS_CLUSTER_NAME=mykubes.k8s.local
kops export kubecfg
```


### Deploy Jupyterhub

### Configure via config.yaml
### MISC

