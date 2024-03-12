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

### controller-manager

Manages the containers, takes care of damaged containers. Continuously monitors the state of various components within the system and works towards bringing the system to the desired state. The node-controller is responsible for monitoring the status of each node, and it does that through the kube-api server, and does this every 5s(node monitor period). If it stops receiving heartbeats from the nodes, it marks it as unreachable after 40s (node monitor grace period).
After a node is marked as unreachable, it gives it 5 minutes to come back up (pod eviction timeout); if it does not it removes the pods assigned to that node and provision them on the healthy ones, if the pods are part of a replicaSet.
The next controller is the replication-controller: it is responsible for monitoring the status of replicaSet and ensures that the desired number of pods are available at all times within the set. If the pod dies, it creates another one.

```bash
kubectl get nodes
```

There are many more such controllers available: deployment, namespace, endpoint, cronjob, job, stateful-set, replicaSet, pv-binder, pv-protection, service-account, (besides the already seed node, replication).
These controllers are packaged into the kube-controller-manager.

Download the kube-controller-manager and run it as a service.
If instead you set up with kubeadm, the kube-controller-manager is run as a pod in the kube-system namespace in the master node.

```bash
kubectl get pods -n kube-system
```

### scheduler

Responsible for scheduling pods on nodes. It's only responsible for deciding on which pod goes on which node. It does not actually place the pod on the node. That's the job of the kubelet. The kubelet creates the pod on the node, the scheduler only decides which pod goes where.

Why do you need a scheduler? You want to make sure the node has sufficient resources to accomodate a pod. The scheduler looks at each pod and tries to find the best node for it.
In the first phase, the scheduler filters out the nodes that do not have sufficient CPU and memory for the pod. Now how does the scheduler pick one from the remaining nodes? It uses a priority function to assign a score to each node. It calculates the amount of the resources that would be free on the node after placing the pod on it. 
This can be customized and you can write your own scheduler as well.

You can either install the kube-scheduler or have it with the kubeadm tool. In this case it comes up as a pod on the kube-system namespace in the master node.

### kubelet

The kubelet in the kubernetes worker node registers the node in the cluster. When it receives instructions to load a pod on the node, it request the container runtime engine (docker) to pull the required image and run an instance. The kubelet then continues to monitor the state of the pod and containers in it and reports to the kube-api server on a timely basis. 

If you use the kubeadm tool to deploy the cluster, it does not automatically deploy te kubelet. That's the difference with other components, you must always manually install the kubelet on the worker nodes.

### kube-proxy

Every pod can reach every other pod. This is accomplished by deploying a pod network to the cluster. It's an internal virtual network that spans across all the nodes in the cluster to which all the pods connect to. Through this network they are able to communicate with each other.
Example: web app deployed on first node, and database on the second. The web app can reach the database simply by using the IP of the pod, but is not guaranteed that the IP of the pod will remain the same. A better way is to use a service. The service is used to expose the DB app across the cluster. The service also gets an IP address assigned to it, and it forwards the traffic to the database.
What is this service and how does it get an IP? Does the service join the same pod network?
The service cannot join the pod network, because it's not actually a thing, is not a container like pods. It does not havy any interface. It is a virtual component that only lives in the kubernetes memory. 
That's where kube-proxy comes in. 
kube-proxy is a process that runs on each node in the cluster, and its job is to look for new services and every time a new service is created it creates the appropriate rules on each node to forward traffic to those services. One way is to use IPtable rules on each node.

To install kube-proxy download it and run it as a service. The kubeadm deploys kube-proxy as a pod on each node, infact it is deployed as daemonset. 

```bash
kubectl get dameonset -n kube-system
```

### pods

Are usually one-to-one with containers: when scaling, you don't add containers to a pod, but rather add new pods. Maybe helper containers can be ship together in the same pod and can live alongside the main container.

```
## run a pod pulling the image from the docker hub
kubectl run nginx --image nginx 
```

```bash
kubectl get pods
```

This just deploys a pod, but later we'll see how to make this accessible from the outside world.

You can create pods using YAML based configuration file. Kubeernetes uses YAML files as inputs for the creation of objects such as pods, replicas, deployments, services, etc...
Similar structure with 4 top-level fields:

-- `pod-definition.yml`
```yaml
apiVersion: v1
kind: Pod
metadata:
    name: myapp-prod
    labels:
        app: myapp
        type: front-en
spec:
    containers:
        - name: nginx-controller
          image: nginx
        
```

`apiVersion`: we must use the right API version, v1 for pod and service, `apps/v1` for ReplicaSet and Deployment.
`kind`: Pod, Service, ReplicaSet, Deployment
`metadata`: is a dictionary, where name and labels are siblings. The name is a string value, and is the name of the pod in this example. Labels can have as many k-v as you wish. This will help you identify this object later in time when there are a lot of pods. You can label them as frontend, backend, database, etc... to filter them later in time.
`spec`: this is different for different objects, refer to docs to get the right format. In a pod with a single container, you add a containers key, which is a list of containers in the pod.

```bash
kubectl create -f pod-definition.yml
```

Once created you see it with:
```bash
kubectl get pods
kubectl describe pod myapp-prod
```

If you are creating a new object you can either use apply or create.
