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

Docker is the most dominant container tool. Kubernetes was built to orchestrate docker in the beginning. Then Kubernetes grew in popularituy and also started to support containerd and rkt.

### ETCD

Key-value store. Differently from relational databases, k-v stores information in the form of k-v pages. k-v translates easily into JSON or YAML. 

```bash
# first install etcd

# it starts a server that listens on port 2379 by default
./etcd 

# the client, to store or get a key value pair

./etcdctl set key1 value1
./etcdctl get key1
```

With kubeadm etcd runs as a container in the master node; in case of a cluster of master nodes, you need to specify the cluster IPs of all the etcd servers in order to get them known by each other.

### kube-api server

Primary management component in kubernetes. When you run a `kubectl` command, this actually reaches the kube-api server, which first authenticates the request and validates it. Instead of using the `kubectl` you could as well use API requests.

What happens when creating a pod: the request is authenticated, validated, then the API server creates the pod object without assigning it to a node, updates the info into the ETCD cluster, updates the user that the pod has been created; then the scheduler continuosly monitors the API server and realizes that there is a new pod without a node assigned. The scheduler identifies the right node to place the new pod on and communicates that back to the API server. The API updates the info into the ETCD cluster. The API server then passes that information to the kubelet in the appropriate worker node. The kubelet then creates the pod on the node and instructs the container runtime engine to deploy the application image. Once done, the kubelet updates the status back to the API server, and the API server updates the data back in the ETCD cluster.

A similar pattern is followed every time a change is requested: the API server is at the center of all the different tasks that need to be performed to make a change in the cluser.
The kube-api server is responsbile for:
- authenticate and validate request
- retrieve and update data in the ETCD store, infact kube-api is the only component that interacts directly with the ETCD
- scheduler uses API server to perform updates in the cluster
- communicate with the kubelet
