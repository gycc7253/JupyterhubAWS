# JupyterhubAWS
Setting up Jupyterhub on AWS with kops/kubeadm

## Understanding Kubernetes
For your basic understanding, refer to the image below, kubernetes is a container orchastration service that has a single control-plain/master to admin all activities of worker nodes and pods are containers that are running on nodes.

### A simple metaphor
Just imagine that the cluster is a fleet, and the control-plane/master is the flag ship which does all the admin stuff but doesn't carry any cargo. Each of the worker nodes is a cargo ship of the fleet, carrying a number of containers.

As kubernetes admin, what you need to do is simply talking to the flag ship's captain(API server), then you have command over the whole fleet.

Then on each of the cargo ship(worker node), there is a network proxy that directs traffic from the node's port listener to pod, and this network proxy is what the users are talking to. Each of the pod is like mini self-sufficient operating system, has all the dependencies installed for its image to run.
![full-kubernetes-model-architecture](https://user-images.githubusercontent.com/58676681/127451109-9bb4bdd2-c6c1-44c2-8955-33d9e0566627.png)


### Understanding Jupyterhub
![Overview-of-JupyterHub-components-56](https://user-images.githubusercontent.com/58676681/127451116-41b4cf78-9295-49d1-b0b9-60fc1a16ca96.png)
