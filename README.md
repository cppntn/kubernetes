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

## Certification Tips - Imperative Commands with Kubectl

While you would be working mostly the declarative way - using definition files, imperative commands can help in getting one time tasks done quickly, as well as generate a definition template easily. This would help save considerable amount of time during your exams.

Before we begin, familiarize with the two options that can come in handy while working with the below commands:

--dry-run: By default as soon as the command is run, the resource will be created. If you simply want to test your command , use the --dry-run=client option. This will not create the resource, instead, tell you whether the resource can be created and if your command is right.

-o yaml: This will output the resource definition in YAML format on screen.

Use the above two in combination to generate a resource definition file quickly, that you can then modify and create resources as required, instead of creating the files from scratch.

POD
Create an NGINX Pod

kubectl run nginx --image=nginx

Generate POD Manifest YAML file (-o yaml). Don't create it(--dry-run)

kubectl run nginx --image=nginx --dry-run=client -o yaml


Deployment
Create a deployment

kubectl create deployment --image=nginx nginx

Generate Deployment YAML file (-o yaml). Don't create it(--dry-run)

kubectl create deployment --image=nginx nginx --dry-run=client -o yaml


Generate Deployment with 4 Replicas

kubectl create deployment nginx --image=nginx --replicas=4


You can also scale a deployment using the kubectl scale command.

kubectl scale deployment nginx --replicas=4

Another way to do this is to save the YAML definition to a file and modify

kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > nginx-deployment.yaml

You can then update the YAML file with the replicas or any other field before creating the deployment.

Service
Create a Service named redis-service of type ClusterIP to expose pod redis on port 6379

kubectl expose pod redis --port=6379 --name redis-service --dry-run=client -o yaml

(This will automatically use the pod's labels as selectors)

Or

kubectl create service clusterip redis --tcp=6379:6379 --dry-run=client -o yaml (This will not use the pods labels as selectors, instead it will assume selectors as app=redis. You cannot pass in selectors as an option. So it does not work very well if your pod has a different label set. So generate the file and modify the selectors before creating the service)


Create a Service named nginx of type NodePort to expose pod nginx's port 80 on port 30080 on the nodes:

kubectl expose pod nginx --type=NodePort --port=80 --name=nginx-service --dry-run=client -o yaml

(This will automatically use the pod's labels as selectors, but you cannot specify the node port. You have to generate a definition file and then add the node port in manually before creating the service with the pod.)

Or

kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml

(This will not use the pods labels as selectors)

Both the above commands have their own challenges. While one of it cannot accept a selector the other cannot accept a node port. I would recommend going with the kubectl expose command. If you need to specify a node port, generate a definition file using the same command and manually input the nodeport before creating the service.

## kubectl apply

apply can be used to manage objects in a declarative way. The apply command takes into consideration the local yml file, the live object definition on kubernetes and the last applied configuration, before making a decision on what changes need to be made.

When an object is created, kubernetes creates a live object configuration internally. The last applied configuration instead is a JSON format of the YML file.

local file YML -> kubernetes live object YML -> last applied JSON

last applied JSON is stored as an annotation in the live object in kubernetes memory.

## Scheduling

### Manual Scheduling

If you want to schedule the pod yourself: every pod has a field called nodeName, which you don't usually set. Kubernetes sets the nodeName automatically. The scheduler goes through all the pods and looks for those that do not have this propoerty set. Those are the candidates for scheduling. It then identifies the right node for the pod by running the scheduling algorithm. Once identified, it schedules the pod on the node by setting the nodeName property to the name of the node, by creating a binding object.

If there's no scheduler to monitor and schedule pods, the pod remains in a pending state. So you have to do it yourself manually by setting the nodeName property (it's a sibling of containers field). The pod then gets assigned to specified node.
You can only specify the node for a pod at creation time; to assign a different node to a pod, Kubernetes does not allow you to modify the nodeName property of a pod.
So in order to assign a node to an existing pod is to create a Binding object:

```pod-bind.yaml
apiVersion: v1
kind: Binding
metadata:
    name: nginx
target:
    apiVersion: v1
    kind: Node
    name: node02
```

### Labels and selectors

Labels and selectors are a standard method to group things together, with one or multiple criteria. Labels are properties attached to each object, and selectors are filter you can use.
They are just tags. 
We have a lot of different types of objects in Kubernetes: pods, services, replicasets, deployments, etc... Over time you may end up having hundreds of these objects, and you need a way to view these categorised by type or application of functionality or whatever.
For each object attach labels as per your needs.

How do you specify labels? In metadata field, add labels, in key value format. Add as many labels as you like.
You can get the object with --selector option

```bash
kubectl get pods --selector app=App1
```

Kubernetes also uses labels and selector internally to connect different objects together, for example to create a ReplicaSet with 3 pods you need to specify the selector in the ReplicaSet to match with the labels of the pod. It also works with services, you need the selector to attach a service to pods.

Annotations are instead used to record other details. `annotations` is a sibling of `labels` in `metadata`.

### Taints and tolerations

What pods are placed on what nodes. Taints and tolerations have nothing to do with security, they are used to set restrictions on what pods are scheduled on a node.

Suppose a node, node01, we place a Taint=blue; we have to specify which pods are tolerant to Taint=blue; we add a toleration to pod D, and so pod D can go to node 1.

```
kubectl taint nodes node-name key=value:taint-effect
```

taint-effect defines what happens to the pod if it can't tolerate the node:
- NoSchedule: the pod will not be scheduled on the node
- PrefereNoSchedule: avoid but not guaranteed
- NoExecute: new pods will not be scheduled on the node and existing pods on the node will be evicted if they do not tolerate the taint

```
kubectl taint nodes node01 app=blue:NoSchedule
```

then, as a sibling of containers in the spec of a pod, add a section called tolerations:

```yaml
tolerations:
- key: "app"
  operator: "Equal"
  value: "blue"
  effect: "NoSchedule"

```

master node already has a taint set to NoSchedule to prevent pods to be created on the master node.

to remove a taint, just put the minus symbol at the end of the taint command:

```
kubectl taint nodes node01 app=blue:NoSchedule-
```


### Node Selectors

Suppose we have 2 small nodes and 1 big node, with more resources. In the current default setup, any pod can go to any node. We can set a limitation on the pod so that they only run on particular nodes.

In the pod definition file, add as a sibling of the containers called nodeSelector

```
nodeSelector:
    size: Large
```

size: Large is a k-v pair assigned previously to the node we want. 

How we can label the node?
```
kubectl label nodes node01 size=Large
```

When the pod is created, is created on the node01 as desired. What if we want to place the pod on any node that is not small, but only to nodes labeled as Large or Medium. We need to leverage Node Affinity for this.

### Node Affinity

we nede to specify a sibling of the containers in the pod definition:

```yaml
affinity:
    nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: size
                operator: NotIn
                values:
                - Small
```

What if we have no match for the nodes?

we also have another type of node affinity:

`preferredDuringSchedulingIgnoredDuringExecution`

During scheduling is the state when the pod is first created; what if the nodes with matching labels are not available? If you select the required, the pod will not be scheduled.
If you select preferred, the pod will be scheduled on another node.

IgnoredDuringExecution: pods will continue to run despite any change in node affinity, once they are scheduled.

We also have `requiredDuringSchedulingRequiredDuringExecution`, where the pod will be evicted if the node affinity changes.

### Resource Requirements and Limits

Each node as a CPU and Memory available. Then we have pods, and every pods consume a number of CPU and Memory. 
To do this, add a `resources` field into the `containers` field

```bash
containers:
    resources:
        requests:
            memory: "4Gi"
            cpu: 2
        limits:
            memory: "8 Gi"
            cpu: 4
```

1 count of CPU is 1 vCPU, and you can specify as low as 1m (0.001). So 100m is 0.1, for example. With memory you can either specify G or Gi depending on GB or GiB.
If the container goes beyond the limits, it's ok for CPU because is throttled, but for Memory it will throw an Out of Memory error and the pod terminates.

By default CPU and Memory do not have requests or limit, so any pod can use as much as it can.


If you don't specify requests, but specify limits, requests are set to the same limits. Setting requests but not limits, any pod can consume as many as it wants. And this is the most ideal scenario, to guarantee a minimum with requests, but without limits.

For memory is similar: one pod can consume all memory if limits are not set. If we have limits only, requests are set to be similar to limits. But unlike CPU the Memory cannot throttle, so we have to kill the pod to free memory.

We can use LimitRange to set default values for containers, and it is applied at a namespace level. This only affects new pods created or updated after the LimitRange creation.

To restrict the total amount of resources in a cluster, we can create ResourceQuota at a namespace level: hard limits and hard requests. This limits the total requests and limits consumed by all the pods together in the namespace.

### Daemon Sets

Daemon Sets are like Replica Sets, but it runs one copy of your pod in each node of the cluster. Whenever a new node is added to the cluster, a new pod replica is added automatically to the node. The DaemonSet ensures that one replica of the pod is present on every node.

Use cases: monitoring, log collector, to be deployed on each node. One of the worker node components that is required in every node is the kube-proxy, and can be deployed as a daemon set. 

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
    name: monitorin-daemon
spec:
    selector:
        matchLabels:
            app: monitoring-agent
    template:
        metadata:
            labels:
                app: monitoring-agent
        spec:
            containers:
            - name: monitoring-agent
              image: monitoring-agent
```

The YAML is exactly like the ReplicaSet, except the kind is DaemonSet

```
kubectl create -f daemon.yml
kubectl get daemonsets
```

How does it schedule a pod on each node? It uses the default scheduler and the node affinity rules to schedule pods on all nodes.


### Static Pods

What if you only have a node, no part of any cluster, no master, no kube-api server, notihing, just a node: is there anything the kubelet can do? Who can provide instructions to create pods?

The kubelet can manage a node independently, without a k8s cluster, and without a kubeapi-server. How to create a pod? You can configure the kubelet to read the pod configuration files YAML. The kubelet periodically checks the files, and the kubelet takes changes accordingly. If you remove the file, the pod is deleted automatically. These are static pods. You cannot create replicasets, deployments, daemonsets. Only pods.

The kubelet works at a pod level, and can only understand pods.

The directory for the kubelet to read can be any directory, and is passed when running the kubelet service with the `--config=kubeconfig.yaml` option

Into kubeconfig.yaml you'll have `staticPodPath: /etc/kubernetes/manifest` which is the directory where pods yaml are stored.
We don't have the rest of the kubernetes cluster, so we don't have kubectl. We can only use docker. If you attach later a kubernetes cluster, kubectl will see the pods created by the kubelet.

But WHY you want to use static pods? The use case is to deploy the controlplane node itself, where every pod is a controlplane component, so you don't have to worry about installing kubernetes binaries but just dockerize the master node components.

The kube-scheduler has no effects on both static pods and daemonsets.


### Multiple Schedulers

By default scheduler distributes pods across nodes et cetera... but what if this does not satisfy your needs? What if you have to place pods on nodes after additional steps and custom conditions? Kubernetes is extensible, you can write your own kubernetes scheduler, package it and deploy it as the default scheduler (or additional scheduler). 

You kubernetes cluster can have multiple scheduler at a time. You can instruct to have a pod scheduled by a specific scheduler.

The default schedulder is the `default-scheduler`. To create another scheduler you have to create a separate configuration file.


```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
    - schedulerName: my-custom-scheduler
```

then:

```
apiVersion: v1
kind: Pod
metadata:
    name: my-custom-scheduler
    namespace: kube-system
spec:
    containers:
    - command:
      - kube-scheduler
      - --address=127.0.0.1
      - --kubeconfig=/etc/kubernetes/scheduler.conf
      - --config=/etc/kubernetes/my-scheduler-config.yaml

      image: k8s.gcr.io/kube-scheduler-amd64:v1.11.3
      name: kube-scheduler 
```

How can we specify a pod to use this scheduler? Set the `schedulerName` value (a sibling of `containers`) into a pod that you want to be scheduled with a custom scheduled.

How to know which scheduler picked up the a certain pod?

```bash
kubectl get events -o wide
```

Get events and see which scheduler picked which pod.
Or you can see the logs:

```bash
kubectl logs my-custom-scheduler --namespace=kube-system
```

P.S. Get serviceaccount and clusterrolebinding (we'll see these later)

```bash
kubectl get serviceaccount -n kube-system
kubectl get clusterrolebinding
```

### Scheduler Profiles

Pods end up in a scheduling queue, and they are put on nodes by scheduler based on a priority value:
You can set priority in a `priorityClassName` with value `high-priority`, and is a sibling of `containers`.

Pods go through schedulingQueue->filtering->scoring->binding

- schedulingQueue: prioritySort
- filtering: NodeResourcesFit (filters out nodes that have no sufficient resources) or NodeName or NodeUnschedulable (like the control-plane node)
- scoring: NodeResourcesFit (same plugin as before that gives a score to each node, giving a high score to nodes that would have most resources left after placing the pod) or ImageLocality (give a higher score to nodes where the pod is already present)
- binding: DefaultBinder (provides the binding mechanism)

It's possible to customize what plugins go where: This is achieved with extension points, to which plugin can be plugged to. At each stage there is an extension point. There are also extension preFilter and postFilter, or preScore, preBind, postBind. Basically you can get a custom code on your own to plug a plugin where you want.

How we can change the default behaviour of how these plugins are called?
Suppose you have 2 custome scheduler and the default scheduler. This is one way to deploy multiple schedulers. But since they are separate processes we can run into race conditions when having all of them trying to schedule pods.

In a single scheduler instead you can use multiple profiles, so that you can have different scheduling strategies that run into the same binary.

```
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: my-scheduler-1
  plugins:
    score:
      disabled:
      - name: TaintToleration
      enabled:
      - name: MyCustomPluginA
      - name: MyCustomPluginB
- schedulerName: my-scheduler-2
  plugins:
    preScore:
      disabled:
      - name: ...
    score:
      disabled:
      - name: ...
- schedulerName: my-scheduler-3
```

## Logging and Monitoring

What would you like to monitor? Number of nodes in the cluster, how many of them are healthy, CPU, memory and Disk Utilization, etc... We need a solution that will monitor these metrics, store them and provide analytics for these data. There are a number of open source solutions (Prometheus, ELK) to show analytics.

Kubernetes runs a kubelet on each node. The kubelet also contains a subcomponent, called `cAdvisor`, responsible for retrieving performance metrics from pods and expose them throught the kubelet api to make them available. 

To deploy the metrics-server you clone the manifest from github and deploy it with `kubectl create -f ...`

This command deploys a set of pods, services and roles to enable metrics server to poll metrics. After some time, the data will be collected and processed, and you can view metrics with:

For nodes:
```
kubectl top node
```

For pods:
```
kubectl top pod
```

### Managing application logs

Docker streams log events to stdout. If I run a docker detached I could not see the logs, I have to run `docker logs -f` to see the logs. 

In kubernetes we can view the logs with:

```
kubectl logs -f name-of-the-pod
```

These logs are specific to the container running inside the pod. If you have more than one container in a pod, you must specify the name of the container.

```
kubectl logs -f name-of-the-pod name-of-the-container
```

## Application Lifecycle Management

We'll see rolling updates and rollbacks in deployments, the different ways to configure applications, scale applications, and primitives of self-healing applications.

### Rolling updates and rollbacks

In a deployment, when you first create it, it triggers a rollout, and a new rollout creates a new deployment revision. Let's call it revision 1. In the future, when the container version is updated, a new rollout is triggered and a revision 2 is created. This helps keep track of our deployments and roll back to previous revisions if necessary.

```
kubectl rollout status deployment/deployment-name
```

To see the revisions and history

```
kubectl rollout history deployment/deployment-name
```

There are 2 types of deployments strategies. Say you have 5 replicas of your pod. One way to upgrade is to detroy all of them and create newer versions. First destroy, then create. The problem with this is that you experience a downtime and users cannot access the webapp. But this is not the default deployment strategy. 
A second strategy is to take down and bring up a new version one by one, and in this way the application does not go down. This is the rolling update strategy, which is also the default strategy.

How exactly do you update the deployment? For example you update the version of the docker image used, or you update the number of replicas. You modify the manifest file and then run the `kubectl apply -f deploy.yaml` to apply the changes. A new rollout is triggered and a new revision of the deployment is created.

You can also set the image manually, but be aware that you will have to change the manifest file accordingly:

```
kubectl set image deployment/deployment-name <container-name>=<image-name>:<image-tag>
```

If you run `kubectl describe deployment` command you can see which strategy has been used.

How a deployment perform an upgrade under the hoods?

It first creates the new ReplicaSet and starts deploying the containers there, and in the meantime it brings down the pods in the old ReplicaSet, one by one. 

If you notice errors and would like to rollback, you can undo a change with:

```bash
kubectl rollout undo deployment/deployment-name
```

To notice the different before and after the rollout, run `kubectl get replicasets`


Create: `kubectl create -f deployment.yml`
Get: `kubectl get deployments`
Update: `kubectl apply -f deployment.yml`
Status: `kubectl rollout status deployment/deployment-name`
History: `kubectl rollout history deployment/deployment-name`
Rollback: `kubectl rollout undo deployment/deployment-name`

To change the deployment strategy:

```
kubectl edit deploy frontend
```

Then change the strategy type to `Recreate`, and save the file.

Then verify with `kubectl describe deploy frontend`

### Configure Applications

Application commands and arguments and entrypoints in docker.

When you run a docker, you can append a command that overrides the `CMD` in the Dockerfile: 

```
docker run ubuntu sleep 5
```

How do you make this permanent? You can create another Dockerfile and specify a new `CMD`.

To set arguments:

```docker
FROM ubuntu
ENTRYPOINT ["sleep"]
```

With entrypoint all the arguments are appended. so `5` will be appended.

To configure a deafult value to append, you can use both `ENTRYPOINT` and `CMD`.

```docker
FROM ubuntu
ENTRYPOINT ["sleep"]
CMD ["5"]
```

To override during run:

```
docker run --entrypoint nosleep ubuntu-sleeper 10
```

Let's see commands and arguments in a kubernetes pod.

Given this as `ubuntu-sleeper`:

```docker
FROM ubuntu
ENTRYPOINT ["sleep"]
CMD ["5"]
```

You use this pod with args to be appended

```
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    args: ["10"]             ## this overrides the CMD
    command: ["sleep2.0"]    ## this overrides the entrypoint
```

### Environment Variables

You can hard code env variables, or retrieve them from ConfigMaps and Secrets.

```
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    args: ["10"]             ## this overrides the CMD
    command: ["sleep2.0"]    ## this overrides the entrypoint
    env:
    - name: APP_COLOR
      value: pink
    - name: APP_COLOR_FROM_CONFIGMAP
      valueFrom:
        configMapKeyRef:
    - name: APP_COLOR_FROM_SECRETS
      valueFrom:
        secretKeyRef: 
```


### Configuring ConfigMaps in applications

1) Create the ConfigMap
2) Inject into the pod


There is an imperative way, you can directly specify the configmap.
```bash
kubectl create configmap \
    app-config --from-literal=APP_COLOR=blue
```

or imperative from a file
```bash
kubectl create configmap \
    app-config --from-file=app_config.properties
```

But a declarative approach is better, with a file `config-map.yml`:

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: prod
```

And then create the config map:
```
kubectl create -f config-map.yml
```

To view configmaps:

```
kubectl get configmaps
```
or 
```
kubectl describe configmaps
```

Now, to use them in a pod:

```
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    envFrom:
    - configMapRef:
      name: app-config
```

But you can also give them one by one

```
env:
- name: APP_COLOR
  valueFrom:
    configMapKeyRef: 
      name: app-config
      key: APP_COLOR
```

or into volumes:

```
volumes:
- name: app-config-volume
  configMap:
    name: app-config
```

### Configure Secrets in applications

ConfigMaps store config data in plain text format, but definitely not the right place to store a password. Secrets are used to store sensitive information.
They are stored in an encoded format.
There are 2 ways to create a secret:

1) Imperative

```
kubectl create secret generic <secret-name> --from-literal=<key>=<value>
kubectl create secret generic app-secret --from-literal=DB_HOST=mysql
```

If you have multiple secrets, and want to create them in an imperative way, use a file:
```
kubectl create secret generic <secret-name> --from-file=app_secret.properties
```

2) Declarative using .yaml file

```
kubectl create -f secrets.yaml
```

Create a definition file:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_HOST: mysql
  DB_USER: root
  DB_PASSWORD: password
```

But it's safer to encode data in base64 encoded format:

```
echo -n 'mysql' | base64
echo -n 'root' | base64
echo -n 'password' | base64
```


So now we have:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_HOST: bXlzcWw=
  DB_USER: cm9vdA==
  DB_PASSWORD: cGFzc3dvcmQ=
```

List newly created secrets:

```
kubectl get secrets
kubectl describe secrets
```

But this hides the values. To view the value of the secrets do:

```
kubectl get secret app-secret -o yaml
```

To decode encoded values do:

```bash
echo -n 'bXlzcWw=' | base64 --decode
```

Now, to use them in a pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: app-secret
```


But you can also give them one by one

```yaml
env:
- name: APP_COLOR
  valueFrom:
    secretKeyRef: 
      name: app-secret
      key: DB_HOST
```

or into volumes:

```yaml
volumes:
- name: app-secret-volume
  secret:
    secretName: app-secret
```

If you were to mount the secret as a volume in the pod, each attribute in the secret is created as a file with the value of the secret as its content.

Secrets are not encrypted, they are only encoded. So anyone can decode them.
Secrets are not encrypted in ETCD, so consider enabling encrypting secret data at Rest: https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data

Anyone able to create pods/deployments in the same namespace can access the secrets.

Consider third-party secrets store providers: AWS, Azure, GCP, Vault Provider.

### Encrypting secret data at rest

This is a demo:

```bash
kubectl create secret generic my-secret --from-literal=key1=supersecret
```

But how is this data stored in the ETCD server?

First step is to install `etcdctl` command:

```bash
apt-get install etcd-client
```

If you do not want to install it, you can also run `etcdctl` command into the pod where the ETCD is running.

```bash
kubectl get pods -n kube-system
```

You will have the `etcd-controlplane` pod.

### Multi-container Pods

Sometimes you have to add a logger to a pod, so you use two containers. They are created and destroyed together, and they share the same network space (localhost) and storage, so you don't have to establish volume sharing or services between them to enable communication. 
`containers` is an array an we can add more than one container per pod.


#### Init Containers

In a multi-container pod, each container is expected to run a process that stays alive as long as the POD's lifecycle. For example in the multi-container pod that we talked about earlier that has a web application and logging agent, both the containers are expected to stay alive at all times. The process running in the log agent container is expected to stay alive as long as the web application is running. If any of them fails, the POD restarts.

But at times you may want to run a process that runs to completion in a container. For example a process that pulls a code or binary from a repository that will be used by the main web application. That is a task that will be run only  one time when the pod is first created. Or a process that waits  for an external service or database to be up before the actual application starts. That's where initContainers comes in.

An initContainer is configured in a pod like all other containers, except that it is specified inside a initContainers section,  like this:

```yml

apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'git clone <some-repository-that-will-be-used-by-application> ; done;']
```


When a POD is first created the initContainer is run, and the process in the initContainer must run to a completion before the real container hosting the application starts. 

You can configure multiple such initContainers as well, like how we did for multi-containers pod. In that case each init container is run one at a time in sequential order.

If any of the initContainers fail to complete, Kubernetes restarts the Pod repeatedly until the Init Container succeeds.

## Cluster Maintenance

Topics: operating systems upgrades, losing a node from the cluster, apply patches, kubernetes releases and versions, best practices around upgrading.
Perform an end-to-end upgrade on a cluster, and also apply a disaster recovery scenario.

### Node OS Upgrades

`pod-eviction-timeout` is 5 minutes by default and is the time the kubelet waits before deploying again a pod on another node when the previous node is offline, but only if the pod is part of a replicaSet.
If you don't have replicaSet for a pod, you can drain the node before putting it offline with `kubectl drain node-1` and the pods gracefully terminated and recreated on another node. You can then put the node offline and do the upgrades. When the node comes back online it is still unschedulable, so you need to uncordon it with `kubectl uncordon node-1`.
There is also `kubectl cordon node-2`: this simply makes sure that new pods are not scheduled on this pod, but this command does not move existing pods to other nodes.

### Kubernetes releases and versions

If you prompt `kubectl get nodes` you'll see the Kubernetes version number:

Example:
```
v1.13.4
```

kube-apiserver, controller-manager, kube-scheduler, kubelet, kube-proxy and kubectl all have the same kubernetes version (core control-plane components). etcd-cluster and core-dns have their own versions as they are separate projects.

### Cluster Upgrade Process

Let's focus on the core control-plane components. Components cannot be at versions higher than the kube-apiserver. They can be at X-1 (controller-manager and kube-scheduler) or at most X-2 versions lower (kubelet and kube-proxy).

When should you upgrade?

Only the last three minor versions are supported. So if kubernetes releases v1.13, v1.10 becomes unsupported. Say you have version v1.11, and v1.13 comes out. Should you upgrade to v1.13? No, because v1.11 is still supported. When v1.14 comes out, update one at a time, so upgrade to v1.12.

`kubeadm upgrade plan` and `kubeadm upgrade apply` to upgrade master and worker nodes.

First you upgrade your master node, and then the worker nodes. When upgrading the master, your app is still running because the workers are still running. Once the upgrade on master is completed, upgrade the worker nodes. Now the pods are down and users are not able to access the application. But how to avoid downtime? Upgrade one node at a time.

But first you have to update kubeadm:

```md
apt-get upgrade -y kubeadm=1.12.0-00
kubeadm upgrade apply v1.12.0
# then upgrade on worker nodes as well

# upgrade kubelet on the master node
apt-get upgrade -y kubelet=1.12.0-00
systemctl restart kubelet

# then worker nodes, one at a time
# first move the workload on other nodes
kubectl drain node-1

# the on node-1
apt-get upgrade -y kubeadm=1.12.0-00
apt-get upgrade -y kubelet=1.12.0-00
kubeadm upgrade node config --kubelet-version v1.12.0
systemctl restart kubelet

# and uncordon it on master
kubectl uncordon node-1
```










