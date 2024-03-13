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

You can create pods using YAML based configuration file. Kubernetes uses YAML files as inputs for the creation of objects such as pods, replicas, deployments, services, etc...
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

### Replication Controller

This is one controller in particular. What is a replica? If our app crashes and fails, to prevent users from losing access, we'd like to have more than 1 pod running. Replication controller helps us run multiple instances of a single pod in the kubernetes cluster, providing high availability. Even if you have a single pod, replication controller can help by automatically bringing up the new pod when the existing one fails.

Another application of replication controller is to create multiple pods to share the load across them. When the number of users increase, we deploy additional pods to balance the load across the 2 pods. If the demand further increase and we run out of resources on the first node, we could deploy additional pods across other nodes in the cluster. Replication controller spans across multiple nodes in the cluster.

Replication controller is the older technology, now replaced by ReplicaSet. ReplicaSet is the new recommended technology. 

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-replicaset
  labels:
    app: myapp
    type: front-end
spec:
  template:
    metadata:
      name: myapp
      labels: 
        app: myapp
        type: front-end
    spec:
      containers:
      - name: nginx-container
        image: nginx
  replicas: 3
  selector: 
    matchLabels:
      type: front-end
```

in the template we simply add what we already had in the pod yaml (withou apiVersion and kind), then also add the replicas number. The selector is necessary in the ReplicaSet because ReplicaSet can also manage pods that were not created with it, i.e. are not in the template section. Selector is the major difference between ReplicaSet and Replication Controller. RC assumes that selector is the pod in the template, instead RS needs this selector.

The role of the ReplicaSet is to monitor the pods and provision the exact number of pods if any of them fails. We use the matchLabels filter to provide the label for the pods we want to replicate. Template definitino is also required in the ReplicaSet, even if the pods were run created in another yaml; this is because ReplicaSet needs to know the spec when spawning new pods.

How we scale the ReplicaSet? What if we decide to scale to 6? There are two ways:
1) Update the number of replicas in the yaml to 6 and run `kubectl replace -f replicaset.yml`
2) Run `kubectl scale --replicas=6 -f replicaset.yml` or `kubectl scale --replicas=6 replicaset myapp-replicaset`

Second option will not update the number of replicas in the yaml.

Some useful commands:
```bash
kubectl create -f replicaset.yml
kubectl get replicaset
kubectl delete replicaset myapp-replicaset
kubectl replace -f replicaset.yml
kubectl scale --replicas=6 -f replicaset.yml
kubectl scale --replicas=6 replicaset myapp-replicaset
kubectl edit replicaset myapp-replicaset
```

### Deployments

How to deploy your app in a production environment? Rolling updates (update pod in production one by one); if errors, rollback to previous container.

So far: container in pods, multiple pods are deployed using replicasets, and then comes deployment, which provides use with the capability to update the underlyings (replicaset) using rolling updates, undo changes, etc...


The content of the Deployment is identical to the ReplicaSet, except the kind


```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-replicaset
  labels:
    app: myapp
    type: front-end
spec:
  template:
    metadata:
      name: myapp
      labels: 
        app: myapp
        type: front-end
    spec:
      containers:
      - name: nginx-container
        image: nginx
  replicas: 3
  selector: 
    matchLabels:
      type: front-end
```

Some useful commands:
```bash
kubectl apply -f deployment.yml
kubectl get deployments
kubectl get replicaset
kubectl get pods
```

The deployment automatically creates the replicaset. And the replicaset ultimately creates pods.

```bash
kubectl get all
```

TIP: use these commands to generate yml files (dry run them!!)

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment --image=nginx nginx --dry-run=client -o yaml > deployment.yaml
```

### Services

Services enable communication between various componentes within and outside of the application. Services help us connect applications together.
For example we have pods exposed to users, pods for the backend, and pods that communicate with external data sources.

Services enable connectivity for the frontend pods to the end users, communication between frontend and backend, and connectivity between other pods and external data sources.

Example of a use case:

External communication: we deployed our pod. How do we access as external user? Let's look ad the existing setup. The kubernetes node as an IP address (192.168.1.2) and my pc is on the same network with IP 192.168.1.10. Moreover, the internal pod network is in the range 10.244.0.0 and the pod as an IP 10.244.0.2.
I cannot access the pod ad 10.244.0.2 as it is in a separate network. If we ssh into the kubernetes node, from the node we would be able to curl 10.244.0.2; But this is from inside the kubernetes node. I want to be able to access the webserver from the external. We need something in the middle to help us map requests. 
This is where kubernetes service comes in. It is an object, just like pod, replicaset, etc...
The use case of the service is to listen to a port on the node and forward requests on that port to a port on the pod running the application. This kind of application is called NodePort service, because the service listens to a port on the node and forwards requests to the pods. NodePort service has also its own ClusterIP address.

We have other types of services:
- NodePort: service makes an internal pod accessible on a port on the node
- ClusterIP: the service creates a virtual IP inside the cluster to enable communication between different services such as a set of frontend servers to a set of backend servers
- LoadBalancer: to distribute load across different webservers


#### NodePort

3 ports involved: one port on the pod (target port, where the service forwards the requests to), one port on the service itself (just the port), and the port on the node (the node port).

NodePorts can only be in a valid range, which by default is from 30000-32767.


service-definition.yml
```bash
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: NodePort
  ports:
  - targetPort: 80
    port: 80
    nodePort: 30008
  selector:
    app: myapp
    type: front-end

```

targetPort is the port on the pod, port is the service object's port, and nodePort is the port on the node. If you don't provide a targetPort, it'll assume is the same as port. If you don't provide nodePort, it will pick one free port in the range 30000-32767. 
We then use label and selector to connect the pod.

```bash
kubectl create -f service-definition.yml
kubectl get services
```

If you have more than one pod with same labels, the nodeport service automatically selects all the pods as endpoints to forward the requests. The nodeport uses a random algorithm to balance the load across the pods. The nodeport service acts then as a load balancer.

What happens when pods are distributed across multiple nodes? Kubernetes will span the service accross all the nodes and maps the target port to the same node port on all the nodes in the cluster. This way you can access your application using the IP of any of the nodes in the cluster and the port.
To summarize: whether it will be a single pod on a single node, multiple pods on a single node, or multiple pods on multiple nodes, the service NodePort is created exactly in the same way. Without us having to do any additional steps.


#### ClusterIP

A full-stack application typically has different kinds of pods hosting different parts of an application: pods for fronted, pods for backend, pods for mysql, etc...

The frontend servers needs to communicate to backend servers and backend needs to communicate with databases. How to estabilish connectivity between these services or tiers of my app? The pods all have an IP address assign to them, but these IPs are not static; if a pod goes down, another one is recreated but with a new IP address. So we cannot rely on these IP addresses for internal communication. 
A kubernetes server can help us group together an provide a single interface to access the pods in a group. For example, a service created for the backend part, will help group the backend part together and provide a single interface for other pods to access the backend.

This enables us to deploy a microservices based application on kubernetes.

Each layer can now scale or move without impacting communication between tiers.
Each service gets an IP, a name assigned to it, inside the cluster, and that is the name that should be used by other pods to access the service. 
This type of service is known as ClusterIP.

service-definition.yml
```
apiVersion: v1
kind: Service
metadata:
    name: back-end
spec:
    type: ClusterIP
    ports:
    - targetPort: 80
      port: 80
    selector:
      app: myapp
      type: back-end
```

ClusterIP is the default type for Service. targetPort is the port where the backend is exposed; and the port is where the service exposes. To link the service to a set of pods we use the selector to match labels.

```bash
kubectl create -f service-definition.yml
kubectl get services
```

#### Load Balancer

Kubernetes can integrate with cloud native LB. Set the service type to LoadBalancer instead of NodePort. 

service-definition.yml
```
apiVersion: v1
kind: Service
metadata:
    name: back-end
    
spec:
    type: LoadBalancer
    ports:
    - targetPort: 80
      port: 80
      nodePort: 30008
    selector:
      app: myapp
      type: back-end
```

### Namespaces

We were doing stuff inside the `default` namespace. Kubernetes creates a set of pods and services for its internal purposes to isolate this from the user under another namespace, named `kube-system`. A third namespace is created as `kube-public`, where all resources available to all the users are created.

You can create your own namespace as well: for example if you want to create staging and production in the same cluster, you can use namespaces to not affect accidentally the wrong environment. You can also assign resources to each namespace.

`name-of-the-service.namespace.svc.cluster.local`

```
kubectl get pods --namespace=kube-system
```

To create the pod in another namesace use the --namespace arg in the command, or add it in the metadata section for the pod yml.

How to create a namespace?

```bash
apiVersion: v1
kind: Namespace
metadata:
    name: dev
```

By default, you are in the default namespace.
To stick to a namespace without specifying the --namespace option every time, just do:

```
kubectl config set-context $(kubectl config current-context) --namespace=dev
```

To view pods in all namespaces:
```
kubectl get pods --all-namespaces
```

Create a resource quota to set limits on namespaces:


```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
    name: compute-quota
    namespace: dev
spec:
    hard: 
        pods: "10"
        requests.cpu: "4"
        requests.memory: 5Gi
        limits.cpu: "10"
        limits.memory: 10Gi
```


## Imperative vs Declarative approaches

In the IaaC world there are different approaches in managing the infrastructure.
Declarative: final state (the system already know what is done at any time and only applies changes needed to reach the final state)
Imperative: step-by-step instructions

With declarative approach just use `kubectl apply -f deployment.yml` every time you want, both during creation and during update.

Managing object with manifest files (yml) can help us have the final state of the object as we want it.
