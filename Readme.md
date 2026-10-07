
Taint, Toleratin, NodeSelectors, NodeAffinity

1) kubectl get nodes

NAME                                    STATUS   ROLES           AGE     VERSION
cka-bestseller-cluster2-control-plane   Ready    control-plane   5d10h   v1.35.0
cka-bestseller-cluster2-worker          Ready    <none>          5d10h   v1.35.0
cka-bestseller-cluster2-worker2         Ready    <none>          5d10h   v1.35.0

2) 
kubectl taint nodes cka-bestseller-cluster2-worker prod=true:NoSchedule
kubectl taint nodes cka-bestseller-cluster2-worker2 prod=true:NoSchedule

3) nginx-pod.yml

    apiVersion: v1
    kind: Pod
    metadata:
    name: nginx-pod
    namespace: dev
    labels:
        app: nginx
    spec:
    containers:  
        - name: nginx-container
        image: nginx:1.16
        ports:
            - containerPort: 80
    
4) kubectl get po

    NAME        READY   STATUS    RESTARTS   AGE
    nginx-pod   0/1     Pending   0          5s 

5) kubectl taint nodes cka-bestseller-cluster2-worker2 prod=true:NoSchedule- (Remove taint)

6) kubectl get po

    NAME        READY   STATUS    RESTARTS   AGE
    nginx-pod   0/1     Running   0          5s 

7) k delete -f nginx-pod.yml

8) kubectl taint nodes cka-bestseller-cluster2-worker2 prod=true:NoSchedule

9) edit nginx-pod.yml

    apiVersion: v1
    kind: Pod
    metadata:
    name: nginx-pod
    namespace: dev
    labels:
        app: nginx
    spec:
    containers:  
        - name: nginx-container
        image: nginx:1.16
        ports:
            - containerPort: 80
    tolerations:
        - key: "prod"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule" (ex: NoSchedule,PeferNoSchedule,NoExecute)

    NoSchedule => newer pods
    PreferNoSchedule => no guarantee of pods scheduling
    NoExecute => existing/newer pods

10) kubectl get po

    NAME        READY   STATUS    RESTARTS   AGE
    nginx-pod   0/1     Running   0          5s 

11) ------------------------ Node Selector ---------------------
    untaint the nodes

    kubectl taint nodes cka-bestseller-cluster2-worker prod=true:NoSchedule-
    kubectl taint nodes cka-bestseller-cluster2-worker2 prod=true:NoSchedule-

12) create new pod with "nodeSelector"

    nginx-new.yml
    -------------------------------
    apiVersion: v1
    kind: Pod
    metadata:
    name: nginx-new-pod
    namespace: dev
    spec:
    containers:
        - name: nginx-container
        image: nginx:1.16
        ports:
            - containerPort: 80
    nodeSelector:
        env: dev   (those node which have labels env:dev then this pod will schedule)

    ------------------

13) No nodes has labels of env:dev 
    if we are trying to create the above pod ,it will throw error

    kubectl apply -f nginx-new.yml

    k get po

    NAME            READY   STATUS    RESTARTS   AGE
    nginx-new-pod   0/1     Pending   0          4s
    nginx-pod       1/1     Running   0          11h

    k describe po nginx-new-pod 

    Events:
    Type     Reason            Age   From               Message
    ----     ------            ----  ----               -------
    Warning  FailedScheduling  31s   default-scheduler  0/3 nodes are available: 1 node(s) had untolerated taint(s), 2 node(s) didn't match Pod's node affinity/selector. no new claims to deallocate, preemption: 0/3 nodes are available: 3 Preemption is not helpful for scheduling.


14) Now label the nodes

    k label nodes cka-bestseller-cluster2-worker2 env=dev
    node/cka-bestseller-cluster2-worker2 labeled

15)  kubectl apply -f nginx-new.yml

    azhardanish@Mac taint-tolerance % k get po                                             
    NAME            READY   STATUS    RESTARTS   AGE
    nginx-new-pod   1/1     Running   0          6m20s
    nginx-pod       1/1     Running   0          11h


NodeAffinity

Node1 => labels ==> label: disk=hdd
Node2 => labels ==> label: disk=ssd
Node3 => labels ==> label: disk=

pod1 => affinity => disk=ssd            ==>  requiredDuringSchedulingIgnoredDuringExecution ==>   schedule to Node2
pod2 => affinity => disk=hdd            ==>  requiredDuringSchedulingIgnoredDuringExecution ==>   schedule to Node1
pod3 => affinity => disk=(ssd,hdd)      ==>  preferredDuringSchedulingIgnoredDuringExecution ==>  schedule to Node3






affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: env
            operator: In
            values:
            - demo
        topologyKey: kubernetes.io/hostname


create a new nginx-pod with affinity as above

-------------------
nginx-pod-affinity.yml


apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: dev
spec:
  containers:
    - name: nginx-container
      image: nginx:1.16
      ports:
        - containerPort: 80
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
            - key: diskType
              operator: In
              values:
                - ssd

k apply -f nginx-pod-affinity.yml

---------------------------------
k get po                         
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   0/1     Pending   0          9s

k describe pod nginx-pod

Events:
  Type     Reason            Age               From               Message
  ----     ------            ----              ----               -------
  Warning  FailedScheduling  9s (x4 over 20h)  default-scheduler  0/3 nodes are available: 1 node(s) had untolerated taint(s), 2 node(s) didn't match Pod's node affinity/selector. no new claims to deallocate, preemption: 0/3 nodes are available: 3 Preemption is not helpful for scheduling.


k get nodes 
NAME                                    STATUS   ROLES           AGE     VERSION
cka-bestseller-cluster2-control-plane   Ready    control-plane   6d19h   v1.35.0
cka-bestseller-cluster2-worker          Ready    <none>          6d19h   v1.35.0
cka-bestseller-cluster2-worker2         Ready    <none>          6d19h   v1.35.0


Labeled the nodes

k label nodes cka-bestseller-cluster2-worker diskType=ssd
k label nodes cka-bestseller-cluster2-worker2 diskType=hdd


k get nodes --show-labels=true    

NAME                                    STATUS   ROLES           AGE     VERSION   LABELS
cka-bestseller-cluster2-control-plane   Ready    control-plane   6d19h   v1.35.0   beta.kubernetes.io/arch=arm64,beta.kubernetes.io/os=linux,kubernetes.io/arch=arm64,kubernetes.io/hostname=cka-bestseller-cluster2-control-plane,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
cka-bestseller-cluster2-worker          Ready    <none>          6d19h   v1.35.0   beta.kubernetes.io/arch=arm64,beta.kubernetes.io/os=linux,diskType=ssd,kubernetes.io/arch=arm64,kubernetes.io/hostname=cka-bestseller-cluster2-worker,kubernetes.io/os=linux
cka-bestseller-cluster2-worker2         Ready    <none>          6d19h   v1.35.0   beta.kubernetes.io/arch=arm64,beta.kubernetes.io/os=linux,diskType=hdd,kubernetes.io/arch=arm64,kubernetes.io/hostname=cka-bestseller-cluster2-worker2,kubernetes.io/os=linux

As soon as we have added the desired labels to the node ,pod automatically schedulled to the nodes

k get po
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          20h
# kubernetes-taint-tolerance
