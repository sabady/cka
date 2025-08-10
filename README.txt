# vim:set ts=2 sw=2 sts=2 et :

CORE CONCEPTS

kubectl create deployment --image=nginx nginx --dry-run=client -o yaml > deployment.yaml
kubectl create deployment redis-deploy --image=redis --namespace=dev-ns -r=2
kubectl edit deployment nginx

kubectl create deployment httpd-frontend --image=httpd:2.4-alpine -r=3 --dry-run=client -o yaml > httpd-deployment.yaml

kubectl scale deployment httpd-frontend --replicas=4

Create pod:
kubectl run redis --image=redis -n finance --dry-run=client -o yaml > redis.yaml
kubectl run redis --image=redis:alpine --labels='tier=db'
kubectl run custom-nginx --image=nginx --port=8080
kubectl get all --all-namespaces
kubectl replace --force -f nginx.yaml

To move a pod to another node you must delete and recreate it!

The service endpoints count equals that of the pods count.
1 endpoint per pod

Create Service:
kubectl expose pod redis --port=6379 --name redis-service --dry-run=client -o yaml
This will automatically use the pod's labels as selectors

Or

kubectl create service clusterip redis --tcp=6379:6379 --dry-run=client -o yaml
This will assume selectors as app=redis

Or

kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml

Run pod and expose it:
kubectl run httpd --image=httpd:alpine
kubectl expose pod httpd --port=80 --name=httpd

DNS
===
To access pod on another namespace:
<svc_name>.ns.svc.cluster.local
db-service.dev.svc.cluster.local

The metadata can have a namespace value

To set dev as the default namespace:
kubectl config set-context $(kubectl config current-context) --namespace=dev

Context is used to manage multiple clusters and environments from the same management system.

To limit resources in a namespace create Resource Quota.

====
Imperative - step by step instructions - using kubectl commands
Declerative - What (not how) to do - Desired state - kubectl apply 

SCHEDULING

Labels and Selectors and Annotations
====================================
Labels in the template section are those of the pods, labels in the metadata section belong to the ReplicaSet (or other resource).
Other objects that need to discover the ReplicaSet will use those labels.

kubectl get pods --selector app=App1
kubectl get all --selector env=prod
kubectl get pods --selector bu=finance,env=prod,tier=frontend

Taints and Tolerations
======================
Taint - Reject all pods except for ones that answer specific labels
NoExecute - kill running pods if they have no tolerance to the taint
Tolerance - Pods cannot be scheduled on tainted nodes
Master nodes have taint that prevents pods from running.

This mechanism does not garentee node affinity
A pod can be tolerant to certain taint and be scheduled on a node
Annotations are used for other details.

kubectl taint node node01 spray=mortein:NoSchedule

cat bee.yaml 
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: bee
  name: bee
spec:
  containers:
  - image: nginx
    name: bee
    resources: {}
  tolerations:
  - key: "spray"
    operator: "Equal"
    value: "mortein"
    effect: "NoSchedule"
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

Untaint:
kubectl taint nodes controlplane node-role.kubernetes.io/control-plane-

Node Selectors
==============
kubectl label nodes <node-name> <key>=<value>
In a Pod yaml add nodeSelector to the spec section and speify the node labels

Node Affinity
=============
Add affinity and nodeAffinity to the spec section.

➜  cat deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: blue
  name: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: blue
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: blue
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: color
                operator: In
                values:
                - blue
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}

➜  cat red.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: red
  name: red
spec:
  replicas: 2
  selector:
    matchLabels:
      app: red
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: red
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/control-plane
                operator: Exists
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}

Use Taints, Tolerations, and Affinity to make sure pods run on specific nodes.

Resources Limits
================

Add resources section for each container
A request specifies the minimum guaranteed amount of a resource (CPU or memory) that a container needs to function effectively.
A limit defines the maximum amount of a resource that a container is allowed to consume.

 A pod that constantly uses more memory than limited will be terminated and exit with OOM error.

There is LimitRange resource where yo can set default, min, and max limits per container.

ResourceQuota - namespace limits

DaemonSets
==========
1 pod per node always present
yaml is same as ReplicaSet


Static Pods
===========
Pod name postfix is the node name

➜  cat bb.yaml 
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: static-busybox
  name: static-busybox
  namespace: default
spec:
  containers:
  - image: busybox
    name: static-busybox
    command: ["sleep"]
    args: ["3600"]
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

cp bb.yaml /etc/kubernetes/manifests/
# kubelet will take it from there
# The path may change, so check kubelet config:
cat /var/lib/kubelet/config.yaml | grep -i staticPodPath

Priorities
==========

Priority classes

kubectl get priorityclass (pc)

kubectl create pc pc1 --dry-run=client -o yaml                     16:35:51
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  creationTimestamp: null
  name: pc1
preemptionPolicy: PreemptLowerPriority
value: 0 # Edit this
description: # optional
globalDefault: true # optional if you want it to be the default for new pods - there can e only one!
preemptionPolicy: PreemptLowerPriority # defaults to terminating pods with lower priority, set it to never in order to wait for resources

In the pod yaml add priorityClassName under spec

Multiple Schedulers
===================
After deplying a scheduler, in order to use it, add to the pod spec a schedulerName

To view scheduler logs:
kubectl get events -o wide
kubectl logs custom-scheduler --name-space=kube-system

Admission Controllers
=====================

Create TLS secret with crt and key files:
kubectl create secret tls webhook-server-tls --cert=/root/keys/webhook-server-tls.crt --key=/root/keys/webhook-server-tls.key --namespace webhook-demo


LOGGING AND MONITORING


