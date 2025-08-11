# vim:set ts=2 sw=2 sts=2 et :

CORE CONCEPTS

kubectl create deployment redis-deploy --image=redis --namespace=dev-ns -r=2 -o yaml > deployment.yaml
kubectl edit deployment nginx
kubectl scale deployment httpd-frontend --replicas=4

Create pod:
kubectl run redis --image=redis -n finance --dry-run=client -o yaml > redis.yaml
kubectl run redis --image=redis:alpine --labels='tier=db'
kubectl run custom-nginx --image=nginx --port=8080
kubectl get all --all-namespaces
kubectl replace --force -f nginx.yaml

To move a pod to another node you must delete and recreate it!

The service endpoints count equals that of the pods count - 1 endpoint per pod

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
<svc_name>.<ns>.svc.cluster.local => db-service.dev.svc.cluster.local

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

cat bee.yaml  -- set Tolerance
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
kubelet runs the pod, not the api server - put the yaml file in /etc/kubernetes/manifests/ or the path value for staticPodPath in /var/lib/kubelet/config.yaml
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
Monitoring
==========
Kubelet includes CAdvisor for use with Prometheus
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl top node - Nodes CPU RAM usage
kubectl top pod - Pods CPU RAM usage

Logs
====
kubectl logs -f pod-name container-name (if there is more than 1 container in the pod)

APPLICATION LIFECYCLE
Rolling updates and Rollbacks, config apps, scale, self healing.

Rolling updates and Rollbacks
=============================
kubectl rollout status deployment/app-deployment

kubectl rollout history deployment/app-deployment

Recreate strategy - destroy and create

Rolling update - the default - replace pods one at a time

You can use the kubectl apply command

Changing image can be done with:
kubectl set image deployment/app-deployment nginx-container=nginx:1.9.1
kubectl set image deployment/<deployment-name> <container-name>=<new image>

Rollback
========
kubectl rollout undo deployment/app-deployment

Commands and Arguments
======================

Use the args, under the container name:
The argument is passed to the entry point command as parameter...

apiVersion: v1
kind: Pod
metadata:
  name: bee
spec:
  containers:
  - image: nginx
    name: bee
    args: ["10"]

To override the entry point use the command field:

apiVersion: v1
kind: Pod
metadata:
  name: bee
spec:
  containers:
  - image: nginx
    name: bee
    command: ["this"]
    args: ["10"]

This also works:

spec:
  containers:
  - image: nginx
    name: bee
    command: 
      - "this"
      - "10"

Environment Variables
=====================
env: under the image in the containers section:

spec:
  containers:
  - image: nginx
    name: bee
    env:
      - name:
        value:

Get value from configMap or Secret:

    env:
      - name:
        valueFrom:
          configMapKeyRef:

    env:
      - name:
        valueFrom:
          sectKeyRef:

Config Map
==========
kubectl create configmap <config-name> \
  --from-literal=<key>=>value> \
  --from-literal=<key>=>value> 

kubectl create configmap <config-name> \
  --from-file=<path-to-key-value-pairs-file>

configmap.yaml
==============
apiVersion: v1
kind: ConfigMap
metadata:
  name: test
  namespace: default
data:
  APP_COLOR: blue
  APP_MODE: prod

kubectl create -f configmap.yaml


kubectl create configmap test -o yaml
kubectl describe cm test

To use variables in a pod config:
envFrom:
  - configMapRef:
      name: test
      key: APP_COLOR

Or as a volume
volumes:
- name: app-config-volume
  configMap:
    name: test

secrets
=======
Imperative:
kubectl create secret generic \
  <secret-name> --from-literal=<key>=<value>

kubectl create secret generic \
  <secret-name> --from-file=<path>

Declerative:
secret.yaml
apiVersion: v1
kind: secret
metadata:
  name: app-secret
data:
  HOST: bXlkYgo=
  USER: cm9vdAo=
  PASSWORD: cGFzc3dkCg==

The values must be base64 encoded

kubectl get secrets
kubectl describe secret <name> -o yaml

In a pod use:

spec:
  containers:
  - name:
    image:
    ports:
      - containerPort:
    envFrom:
      - secretRef:
          name: app-secret
# This is the easiest way ^^^

OR Single env
    env:
      - name: DB_PASS
        valueFrom:
          secretKeyRef:
            name: app-secret
            key: PASSWORD

OR Volume
  volumes:
    - name: app-secret-volume
      secret:
        secretName: app-secret  

# In the container add volumeMount:
    volumeMounts:
      - name: secret-volume
        readOnly: true
        mountPath: <path>

# This will create a file for each secret using the keys

# ENABLE ENCRYPTION AT REST since the etcd is not encrypted!
# Configure least-privilege access to secrets - RBAC
# Consider third party secrets store provider


Encrypting secret data at rest
==============================
Only new secrets will be encrypted!

Backup secrets:
kubectl get secret -A -o yaml > all-secrets-backup-$(date +%F).yaml

Resend secrets:
kubectl get secret -A -o json \
  | jq -c '.items[]' \
  | while read -r secret; do
      ns=$(echo "$secret" | jq -r '.metadata.namespace')
      name=$(echo "$secret" | jq -r '.metadata.name')
      typ=$(echo "$secret" | jq -r '.type')

      # Skip service account tokens (these are managed by controller)
      if [ "$typ" = "kubernetes.io/service-account-token" ]; then
        echo "skipping SA token: $ns/$name"
        continue
      fi

      echo "re-applying: $ns/$name"
      echo "$secret" \
        | jq 'del(.metadata.uid, .metadata.resourceVersion, .metadata.selfLink, .metadata.creationTimestamp, .metadata.managedFields, .status)' \ # del reoves fields
        | kubectl apply -f -
    done


Multi container pods
====================
You might have initContainers at the same level as containers, but they will start first. Init containers will start according to their order in the yaml file each init must finish before the next one starts.

Init container may have restartPolicy: Always which will make it a sidecar.

apiVersion: v1
kind: Pod
metadata:
  name: yellow
spec:
  containers:
  - image: busybox
    name: lemon
    command: ["sleep","1000"]
  - image: redis
    name: gold

cat elastic-search/app.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: elastic-stack
  labels:
    name: app
spec:
  initContainers:
  - name: sidecar
    image: kodekloud/filebeat-configured
    restartPolicy: Always
    volumeMounts:
    - mountPath: /var/log/event-simulator
      name: log-volume
  containers:
  - name: app
    image: kodekloud/event-simulator
    volumeMounts:
    - mountPath: /log
      name: log-volume

  volumes:
  - name: log-volume
    hostPath:
      path: /var/log/webapp
      type: DirectoryOrCreate

Run command in pod:
kubctl -n <namespace> exec -it app -- cat /log/app.log

kubctl replace --force -f app.yaml - to force replace instead of recreate

Scaling
=======
Vertical - Add resources. In k8s, add pods
Horizantal - Add Instances (Hosts). In k8s, increase resources allocation to a pod

Cluster Autoscaler - for infra horizantal scaling
Horizontal Pod Autoscaler (HPA) - for stateless workloads
Vertical Pod Autoscaler (VPA) - for stateful workloads

Imperative HPA
==============
kubectl autoscale depolyment app \
  --cpu-percent=50 --min=1 --max=10 # will increase or decrease the number of pods
kubectl delete hpa app


kubectl scale --replicas=3 deployment flask-web-app

kubectl get hpa --watch

VPA - for stateful workloads
===
Use kubectl edit deployment to do it manually

For automation, vertical-pod-autoscaler needs to be installed (kube-system namespace)
Recommender - monitors pods and sends recommendations to the Updater
Updater - removes pods with issues
Admission controller - also gets recommendations and applies new pods with updated resorces

kubectl describe vpa flask-app

CLUSTER MAINTENANCE


