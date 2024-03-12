# CKA

## Core concepts

### Cluster Architecture

- worker nodes: host applications as containers
- master nodes: manage, plan, schedule, monitor the nodes of the cluster, using a set of components known as the control plane components
- etcd: highly available key-value store, to maintain information about cluster
- kube-scheduler: identify the right node to place a container on, based on resource requirements
- controller-manager:
    - node-controller: takes care of nodes, onboard new nodes to the cluster, handle nodes that are unavailable or destroyed
    - replication-controller
- kube-apiserver: responsible for orchestrating the controller-managaer, the etcd, the kube-scheduler, monitor the state of the cluster
- kubelet: is an agent that runs on each node of a cluster, it listens for instructions from API server and deploys or destroys containers on the nodes as requires. The API server periodically fetches status report from the kubeletes to monitor the status of nodes and containers.
- kube-proxy: ensures that the necessary rules are in place on the worker nodes to allow the containers to reach each other (for example a web app container comunicating with a database container). Every worker node has a kube-proxy.

### Containers

Everything has to be container compatible: applications, different components that form the entire management system on the master node could be hosted in the form of containers, the DNS servers can all be deployed in the form of containers. Docker is a popular one, so it has to be installed in all the nodes, also the master nodes. Kubernetes also support ContainerD.

### Docker vs ContainerD

docker is the most dominant container tool. Kubernetes was built to orchestrate docker in the beginning. Then Kubernetes grew in popularituy and also started to support containerd and rkt.
