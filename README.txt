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

pod-eviction-timeout is 5 minutes - if a node is down, the pod will recreate on another node
To be sure prefer to drain the node before it goes offline:
kubectl drain node-01
Than
kubectl uncordon node-01
kubectl cordon node-01 - new pods wont be scheduled on the node

K8S Releases
============
Cluster upgrade
===============
On controlplane:
----------------
kubeadm upgrade plan
apt update
apt install -y kubeadm
kubeadm upgrade apply v1.32.7 (according to the plan)
systemctl restart kubelet

Backup and restore
==================
Backup:
etcd:
[with all the following etcd commands include --endpoints=https://127.0.0.1:2379 --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/etcd-server.crt --key=/etc/etcd/etcd-server.key]
In the etcd.service you'll find the data-dir path. back it up
Snapshot: etcdctl snapshot save snap.db
etcdctl snapshot status snap.db
To get etcd version and other details (certs, IP, data-dir):
kubectl describe po etcd-controlplane -n kube-system

ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-snapshot.db

for offline file-level backup of the data dir:
This copies the etcd backend database and WAL files to the target location.
etcdutl backup \
  --data-dir /var/lib/etcd \
  --backup-dir /backup/etcd-backup

To restore:
service kube-apiserver stop
etcdctl snapshot restore snap.db --data-dir <new path!>
configure the new path in the etcd.service
systemctl daemon-reload
restart etcd
restart kube-apiserver

etcdutl snapshot restore /backup/etcd-snapshot.db \
  --data-dir /var/lib/etcd-restored
To use a backup made with etcdutl backup, simply copy the backup contents back into /var/lib/etcd and restart etcd.

SECURITY
Manifests
=========
/etc/kubernetes/manifests/

Certificates API
================
➜  cat akashay.csr.yaml 
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: akshay
spec:
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - digital signature
  - client auth
  request: |
    LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZqQ0NBVDRDQVFBd0VURVBN
    QTBHQTFVRUF3d0dZV3R6YUdGNU1JSUJJakFOQmdrcWhraUc5dzBCQVFFRgpBQU9DQVE4QU1JSUJD
    Z0tDQVFFQXJFTktZQzVoWmxVTzFhMUhROVhURWxrejV3eHdJWjRycnI4V3lLYkdQWUlxCmgrRmk2
    eUhNTmdUZXVLMHJMQk85NlB2UVdQRUtQeXQ0NnJhSm9lSmw2VGV6UlJ0bDkvT3h5VWl2dS9KOGJx
    UzQKRUNiRUl3akpQZGp1N29LeGw0dDA4d2lXRjAyb291WUg0NzVORFJwMzVxZWtCUkZ0bTNwNzNZ
    QzQ1K0RlVnZGNgpLQXJrSGFPZXBWbThLYjk2N3FKS3VhVzcrbkdWUEFvWFhlbG9pcG9jQ0FsTWV5
    c1hMTjloYlZ0VnpRdjBhWXVZCkdjL2FQa05GV21GNVQwNVlWVzYyNEluUHl1d0Fzb1M0RnY2M05q
    RWI0alo0N2ZhZmoybjBHK0xlZUtPbVE5S2kKZk9pYUQzYzhLaVBSQk51NXZqRGdseGgzcHd3UWJF
    cWZlbXNRSThaSVJRSURBUUFCb0FBd0RRWUpLb1pJaHZjTgpBUUVMQlFBRGdnRUJBSjlvaTFNVXFY
    RmVYYXZyalQ0eWZzMEJ6QStPQ01ROEp0cnBheXlMN3N4dnZROHhkV3pNCklJbDhHZ3J0QmZGUnV3
    ejNVNVYrV2dPdENXTkl1LzVxQ0NJL2c3NVVhS1AvSUYzNmg3OWRKamljMWY3Sm5kdXUKYU5GV1kv
    NGhDVHJmblNVeWtjUElvNVArNzVOaDZiSWl0NVRueWZnWDJPeDZzbnU3RXNocDVHdlhtcDRUazhM
    YgpUemhOdnBhbXZmU1MzNWFZMy9aNlNXNnVlTU9yQ2VsVDZUOGZPTWNpRDgreDljQzJNak84QnNT
    TUgwRWxXL09XCmhQcHk4SDgxOTd1ZUdGYmtKSEFLQytaSy9PVkIzQXlIRExJNnYwTTk0VXBHeFpk
    TTYzejdvRUpTMFU4VEdIUWsKOEdScUs5WitCWWt6VVJOVWpnNjNmVVJqdUdXUFZ4SlY0NzQ9Ci0t
    LS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=

To get status
kubectl get csr akshay 
kubectl get csr akshay -o yaml

kubectl certificate approve <name>
kubectl certificate deny <name>
kbectl delete csr <name>

kubeConfig
==========

~/.kube/config
apiVersion: v1
kind: Config

current-context: <default context name>

clusters:
  # multiple clusters
  - name: my-kube-playground
    cluster:
    certificate-authority: ca.crt
    #certificate-authority-data: <base64-encoded-ca.crt-string>
    # certificate-authority-data can be used instead of certificate-authority
    server: https://my-kube-playground:6443

contexts:
  i# which user is used to access each cluster
  - name: my-kube-admin@my-kube-playground
    context:
      cluster: my-kube-playground
      user: my-kube-admin
      namespace: <set-the-default-namespace>

users:
  # user accounts that access clusters
  - name: my-kube-admin
    user:
      client-certificate: admin.crt
      client-key: admin.key


kubectl config view
To change context:
kubectl config use-conext <context-name> --kubeconfig <path-to-non-default-config>

API Groups
==========
kubectl proxy - create proxy to access the api server

Authorization
=============
RBAC
Roles and Role Binings are namespaced

kubectl auth can-i create deployments - a user can check if it has authorization to do something
kubectl auth can-i create deployments -n <namespace> --as dev-user - impersonate a user to check authorization

kubectl get roles -A
kubectl describe role kube-proxy -n kube-system
kubectl describe rolebinding kube-proxy -n kube-system

cat developer-role.yaml developer-rolebind.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: developer
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["delete", "create", "list"]

apiVersion: rbac.authorization.k8s.io/v1
# This role binding allows "jane" to read pods in the "default" namespace.
# You need to already have a Role named "pod-reader" in that namespace.
kind: RoleBinding
metadata:
  name: dev-user-binding
  namespace: default
subjects:
# You can specify more than one "subject"
- kind: User
  name: dev-user # "name" is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  # "roleRef" specifies the binding to a Role / ClusterRole
  kind: Role #this must be Role or ClusterRole
  name: developer # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io

To edit Role:
kubectl edit role developer -n blue

 kubectl get role developer -n blue -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  creationTimestamp: "2025-08-12T14:36:32Z"
  name: developer
  namespace: blue
  resourceVersion: "2375"
  uid: f059e360-d979-43c9-b120-38297c8677c1
rules:
- apiGroups:
  - apps
  resources:
  - deployments
  verbs:
  - create

- apiGroups:
  - ""
  resourceNames:
  - blue-app
  - dark-blue-app
  resources:
  - pods
  - deployments
  verbs:
  - get
  - watch
  - create
  - delete

Cluster Roles
=============
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false

ClusterRole
ClusterRolesBinding

clusterrole can be created to namespaced resources which will be applied across namespaces. For example pods across all namespaces in the cluster.

A new user michelle joined the team. She will be focusing on the nodes in the cluster. Create the required ClusterRoles and ClusterRoleBindings so she gets access to the nodes.

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
  name: nodes-admin
rules:
- apiGroups:
  - '*'
  resources:
  - 'nodes'
  verbs:
  - '*'

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: nodes-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nodes-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: michelle


michelle's responsibilities are growing and now she will be responsible for storage as well. Create the required ClusterRoles and ClusterRoleBindings to allow her access to Storage.

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
  name: storage-admin
rules:
- apiGroups:
  - '*'
  resources:
  - 'persistentvolumes'
  - 'storageclasses'
  verbs:
  - '*'

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: michelle-storage-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: storage-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: michelle

Service Accounts
================
kubectl create serviceaccount <name>
kubectl create token <serviceaccount name>
A timed token is created and stored in secret automatically 

Assign access sing RBAC

kubectrl describe serviceaccount <name>

get the token:
kubectl describe secret <token-name>

Use:
curl http://<IP>:6443/api -insecure --header "Authorization: Bearer <token>"

If the pod is in the cluster you can mount the secret as a volume for easy access.

Each namespace has a default serviceaccount, when a pod is created the default serviceaccount and the token secret are automaically mounted.

A pod can have more service accounts by adding serviceAccountName in spec

You can prevent the default serviceaccount mount using automountServiceAccountToken: false in spec

DANGER!
To create a non expiring token create service-account-token secret:
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: secret-name

  annotations:
    kubernetes.io/service-account.name: <serviceaccount name> # associate the serviceaccount


