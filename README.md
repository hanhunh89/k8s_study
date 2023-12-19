# k8s_study

reference : ch1 ~ ch13 : 쉽게 시작하는 쿠버네티스 / 서지영 <br>
            ch13 : ~ google, chatGPT, in my brain...<br><br>
you can install k8s with https://github.com/hanhunh89/k8s_install <br>


# ch1. pod create and delete
## install httpd pod 
master node
```
$kubectl create deployment my-httpd --image=httpd --replicas=1 --port=80
deployment.apps/my-httpd created

$ kubectl get deployment -o wide
NAME       READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES   SELECTOR
my-httpd   1/1     1            1           33s   httpd        httpd    app=my-httpd

$ kubectl get pod -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP           NODE      NOMINATED NODE   READINESS GATES
my-httpd-7547bdb59f-g6lql   1/1     Running   0          48s   10.244.1.2   worker1   <none>           <none>

$ curl 10.244.1.2
<html><body><h1>It works!</h1></body></html>

```

## delete pod
```
sudo kubectl delete deployment my-httpd
```

## change config
```
kubectl edit deployment my-httpd
```

## connect to pod
```
kubectl exec -it [pod_name] -- /bin/bash
```

## check log
```
kubectl logs [pod_name]
```

# ch2. create nginx pod

## nginx-deploy.yaml
".spec.template.metadata.labels" should be same with ".spec.selector.machLabels"
```
#nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:  #deployment info
  name: nginx-deployment  #name of delplyment
  labels:
    app: nginx    #label of deployment
spec:
  replicas: 2      # create 2 pods
  selector:        # deployment manage the selector(pod)
    matchLabels:
      app: nginx
  template:  # pod create option
    metadata:
      labels:
        app: nginx  #lable of the pod
    spec:          #container info
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```
```
kubectl apply -f nginx-deploy.yaml
```

## create service to connect from outside

```
#nginx-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  labels:
    app: nginx
spec:
  type: NodePort
  ports:
  - port: 8080
    nodePort: 31472
    targetPort: 80
  selector:
    app: nginx
```
```
kubectl apply -f nginx-svc.yaml
```

## check nginx running
find host ip. in that case, 10.178.0.18
```
$ kubectl get node -o wide
NAME          STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION          CONTAINER-RUNTIME
master-node   Ready    control-plane   40h   v1.28.2   10.178.0.18   <none>        Debian GNU/Linux 11 (bullseye)   5.10.0-26-cloud-amd64   containerd://1.6.25
worker1       Ready    <none>          39h   v1.28.2   10.178.0.19   <none>        Debian GNU/Linux 11 (bullseye)   5.10.0-26-cloud-amd64   containerd://1.6.25
```

31472 port of node ip is connected to 8080 of nginx-svc
```
$ kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP          40h
nginx-svc    NodePort    10.110.63.152   <none>        8080:31472/TCP   5m18s
```

nginx is running
```
$ curl 10.178.0.18:31472
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```


# ch3. replicaset
## create replicaset
```
#replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: 3-replicaset #name of replica
spec:
  template:
    metadata:
      name: 3-replicaset
      labels:
        app: 3-replicaset
    spec:
      containers:
      - name: 3-replicaset
        image: nginx
        ports:
        - containerPort: 80
  replicas: 3 # create 3 pods
  selector:
    matchLabels:
      app: 3-replicaset
```
```
kubectl apply -f replicaset.yaml
```
replicaset is running
```
$ kubectl get replicaset,pod
NAME                           DESIRED   CURRENT   READY   AGE
replicaset.apps/3-replicaset   3         3         3       41s

NAME                     READY   STATUS    RESTARTS      AGE
pod/3-replicaset-pmnbc   1/1     Running   0             41s
pod/3-replicaset-q4dwp   1/1     Running   1 (30s ago)   41s
pod/3-replicaset-sbq6l   1/1     Running   0             41s
```
## change number of replica
```
kubectl scale replicaset/3-replicaset --replicas=5 #change replica 3 to 5
```
```
$ kubectl get replicaset,pods
NAME                           DESIRED   CURRENT   READY   AGE
replicaset.apps/3-replicaset   5         5         5       3m22s

NAME                     READY   STATUS    RESTARTS        AGE
pod/3-replicaset-gl7vd   1/1     Running   0               10s
pod/3-replicaset-hfv4v   1/1     Running   0               10s
pod/3-replicaset-pmnbc   1/1     Running   2 (24s ago)     3m22s
pod/3-replicaset-q4dwp   1/1     Running   1 (3m11s ago)   3m22s
pod/3-replicaset-sbq6l   1/1     Running   2 (55s ago)     3m22s
```

## how to delete replicaset and keep the pod
if you do "kubectl delete replicaset ~", replicaset and all pods will be delete.<br>
let's keep pod and only delete replicaset
```
kubectl delete -f replicaset.yaml --cascade=orphan
```
in that situation, if you delete pod, the pod not be re-cretead.
```
$ kubectl get pods,replicaset
NAME                     READY   STATUS    RESTARTS      AGE
pod/3-replicaset-bkmrc   1/1     Running   0             67s
pod/3-replicaset-lm95h   1/1     Running   0             67s
pod/3-replicaset-p96hr   1/1     Running   1 (56s ago)   60s
pod/3-replicaset-xch7q   1/1     Running   0             67s
pod/3-replicaset-xx4rp   1/1     Running   1 (56s ago)   60s

$ kubectl delete pod 3-replicaset-bkmrc
pod "3-replicaset-bkmrc" deleted

$ kubectl get pods,replicaset
NAME                     READY   STATUS    RESTARTS       AGE
pod/3-replicaset-lm95h   1/1     Running   1 (39s ago)    2m6s
pod/3-replicaset-p96hr   1/1     Running   1 (115s ago)   119s
pod/3-replicaset-xch7q   1/1     Running   1 (52s ago)    2m6s
pod/3-replicaset-xx4rp   1/1     Running   2 (32s ago)    119s
```

# ch4. DaemonSet
deamonset create pod to all nodes.
```
#daemonsets.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: prometheus-daemonset # our daemonset name
spec:
  selector:
    matchLabels:
      tier: monitoring # user set label. we this use damonset for monitoring
      name : prometheus-exporter
  template:
    metadata:
      labels:
        tier: monitoring
        name: prometheus-exporter
    spec:
      containers:
      - name: prometheus
        image: prom/node-exporter
        ports:
        - containerPort: 80
```
```
kubectl apply -f daemonsets.yaml
```
check daemonset pod
```
$ kubectl describe daemonset/prometheus-daemonset
Name:           prometheus-daemonset
Selector:       name=prometheus-exporter,tier=monitoring
Node-Selector:  <none>
Labels:         <none>
Annotations:    deprecated.daemonset.template.generation: 1
Desired Number of Nodes Scheduled: 2
Current Number of Nodes Scheduled: 2
Number of Nodes Scheduled with Up-to-date Pods: 2
Number of Nodes Scheduled with Available Pods: 2  # i created 2 workernode. so, we have 2 pod
Number of Nodes Misscheduled: 0
Pods Status:  2 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  name=prometheus-exporter
           tier=monitoring
  Containers:
   prometheus:
    Image:        prom/node-exporter
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age    From                  Message
  ----    ------            ----   ----                  -------
  Normal  SuccessfulCreate  2m15s  daemonset-controller  Created pod: prometheus-daemonset-828m7
  Normal  SuccessfulCreate  2m15s  daemonset-controller  Created pod: prometheus-daemonset-5s5sp
```

we can find 2 pods that running in each workernode
```
$ kubectl get pods -o wide
NAME                         READY   STATUS    RESTARTS      AGE     IP             NODE      NOMINATED NODE   READINESS GATES
prometheus-daemonset-5s5sp   1/1     Running   2 (56s ago)   3m35s   10.244.2.4     worker2   <none>           <none>
prometheus-daemonset-828m7   1/1     Running   3 (30s ago)   3m35s   10.244.1.234   worker1   <none>           <none>
```

clear the project(you have to do this every end of chapter)
```
kubectl delete -f daemonsets.yaml
```

# ch5 cronjob
```
#cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox #very small linux
            imagePullPolicy: IfNotPresent # we will download the busybox image when you don't have it.
            command: # do command as the schedule
            - /bin/sh
            - -c
            - data; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure # you restart this when command is failed
```
```
kubectl create -f cronjob.yaml
```

now, check the commnad
```
kubectl get cronjob -w
NAME    SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
hello   */1 * * * *   False     0        13s             51s
hello   */1 * * * *   False     1        0s              98s
hello   */1 * * * *   False     0        7s              105s
hello   */1 * * * *   False     0        7s              105s
hello   */1 * * * *   False     1        0s              2m38s
hello   */1 * * * *   False     0        3s              2m41s
hello   */1 * * * *   False     0        3s              2m41s
```
command is running good.
```
$ kubectl get pods -w
NAME                   READY   STATUS      RESTARTS   AGE
hello-28358309-k2t5c   0/1     Completed   0          3m
hello-28358310-7rldj   0/1     Completed   0          2m
hello-28358311-zrbk6   0/1     Completed   0          60s
hello-28358312-qs8vn   0/1     Pending     0          0s
hello-28358312-qs8vn   0/1     Pending     0          0s
hello-28358312-qs8vn   0/1     ContainerCreating   0          0s
hello-28358312-qs8vn   0/1     Completed           0          1s
hello-28358312-qs8vn   0/1     Completed           0          2s
hello-28358312-qs8vn   0/1     Completed           0          3s
hello-28358312-qs8vn   0/1     Completed           0          3s
hello-28358309-k2t5c   0/1     Terminating         0          3m3s
hello-28358309-k2t5c   0/1     Terminating         0          3m3s
hello-28358313-4gxnb   0/1     Pending             0          0s
hello-28358313-4gxnb   0/1     Pending             0          0s
hello-28358313-4gxnb   0/1     ContainerCreating   0          0s
hello-28358313-4gxnb   0/1     Completed           0          1s
hello-28358313-4gxnb   0/1     Completed           0          2s
hello-28358313-4gxnb   0/1     Completed           0          3s
hello-28358313-4gxnb   0/1     Completed           0          3s
hello-28358310-7rldj   0/1     Terminating         0          3m3s
hello-28358310-7rldj   0/1     Terminating         0          3m3s
```
kubenetes creates pod to execute command, and terminating.<br><br>

clear the project
```
kubectl delete cronjob hello
```

# ch6 ConfigMap and secret
configmap create command
```
kubectl create configmap <map-name> <data-source> <arguments>
```
map-name : name of configmap<br>
data-source : configmap file/dir path<br>
arguments : how to create configmap. file or literal<br>

## literal configmap
```
kubectl create configmap my-config --from-literal=my_key1=my_value1 --from-literal=my_key2=my_value2
```
check configmap that we made
```
kubectl describe configmap
```

## file/dir configmap
we use file that has config info.
```
echo Hello, world! >> configmap_test.html
```
```
kubectl create configmap configmap-file --from-file configmap_test.html
```
```
kubectl describe configmap
```

## secret
configmap value is saved in container.<br>
if you secret info(ex id/password), you can use secret.<br>
secret is not saved in container.<br>
when pod running, read the secret, and give to container.

```
kubectl create secret generic dbuser --from-literal=username=myuser --from-literal=password=1234
```

# ch7. volume
we can make three type of volume<br>
1. pod volume : storage is inside pod. if pod restarted, we loses the volume.<br>
   ex) emptyDir <br>
2. worker node volume :  storage is inside node. file is saved in node local disk.<br>
   ex) hostPath <br>
3. 2. external volume :  storage is outside of kubernetes. even node/pod restarted, we can keep the volume.<br>
   ex) NFS, cephFS, glusterFS, iSCSI, AWS EBS, azureDisk <br>

## emptyDir
emptydir creaed and removed with pod. 

```
#emptydir.yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydata # creates pod. pod name is emptydata
spec:
  containers:
  - name: nginx # container name is nginx. we mount /data/shared to nginx
    image: nginx
    volumeMounts:
    - name: shared-storage
      mountPath: /data/shared
  volumes: #create volume. volume name is shared-storage
  - name: shared-storage
    emptyDir: {}
```
we create pod. <br>
pod mounts "shared-storage" at /data/shared. <br>
shared-storage is emptyDir.

```
kubectl apply -f emptydir.yaml
```

check volume
```
$ kubectl get pod
NAME        READY   STATUS    RESTARTS   AGE
emptydata   1/1     Running   0          16s

$ kubectl exec -it emptydata -- /bin/bash
$ cd /data/shared
$ echo "hello" > test.txt
$ cat hello
```

## hostPath
hostPath created in Node local disk.
So, even when the pod is removed, the hostPath persists.

```
#hostpath.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: localpath
      mountPath: /data/shared   # worker node directory(/tmp) is mounted /data/shared
  volumes:
  - name: localpath
    hostPath:
      path: /tmp   # we use worker node directory /tmp as hostPath
      type: Directory
```
```
kubectl apply -f hostpath.yaml
kubectl exec -t hostpath -- /bin/bash
cd /data/shared
echo "hello" > test.txt
```
we can find test.txt file in worker node. 
```
worker2:/tmp$ cat test.txt
hello
```

## persistent volume.
we will create persistent volume and make mysql server.<br>

first, create the volume.
```
#pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-volume
  labels:
    type: local
spec:
  storageClassName: my-storage
  capacity:
    storage: 20Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: "/mnt/data"  # we don't have external storage.... so, just use local dist.
```

second, create volumeClaim<br>
bind PersistentVolumeClaim (PVC) and PersistentVolume (PV) with the name 'my-storage.'
```
#pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  storageClassName: my-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```
```
kubectl apply -f pvc.yaml
```

third, create pod
```
#pvc-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:8.0.29
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
          name : mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: var/mysql
      volumes:
        - name: mysql-persistent-storage
          persistentVolumeClaim:
            claimName: mysql-pv-claim
```
```
kubectl apply -f pvc-deployment.yaml
```

fourth, create service
```
#mysql_service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  ports:
  - port: 3306
  selector:
    app: mysql
```
```
kubectl create -f mysql_service.yaml
```


get pvc info
```
$ kubectl get pvc
NAME             STATUS   VOLUME            CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-pv-claim   Bound    mysql-pv-volume   20Gi       RWO            my-storage     3m40s
```
what is access modes?<br>
1. RWO(read wirte once) : only one node access to storage.<br>
2. ROX(read only many) : many node can mount, but read only.<br>
3. RWX(read write many) : many node mount this. all node can write, read.<br>
4. RWOP(read write once pod) : only one pod can mount.<br><br>

## use mysql
let's create table
```
$kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
mysql-74d9b99946-gvzqv   1/1     Running   0          2s

$ kubectl exec -it mysql-74d9b99946-gvzqv -- /bin/bash
$ mysql -u root -p
Enter password:   #password is password. we made it pvc-deployment.yaml

Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 8.0.29 MySQL Community Server - GPL

Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> create DATABASE mydb
    -> ;
Query OK, 1 row affected (0.01 sec)

mysql> use mydb
Database changed

mysql> create table test(id int, name varchar(30));
Query OK, 0 rows affected (0.03 sec)

mysql> select * from test;
Empty set (0.01 sec)

mysql> insert into test value(1,'aa');
Query OK, 1 row affected (0.00 sec)

mysql> select * from test;
+------+------+
| id   | name |
+------+------+
|    1 | aa   |
+------+------+
1 row in set (0.00 sec)
```

# ch8. cluster IP
Service has clusterIP.<br>
when you bind service to pod, you can connect to your pod with clusterIP<br>
let's create pod.
```
#clusterip.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clusterip-nginx
spec:
  selector:
    machLabels:
      run: clusterip-nginx
  replicas: 3
  template:
    metadata:
      labels:
        run: clusterip-nginx
    spec:
      containers:
      - name: custerip-nginx
        image: nginx
        ports:
        - containerPort: 80
```
```
kubectl apply -f clusterip.yaml
```

let's create service
```
kubectl expose deployment/clusterip-nginx
```

for test clusterip-nginx, we create busybox
```
$ kubectl run busybox --rm -it --image=busybox /bin/sh

$ kubectl get service
NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
clusterip-nginx   ClusterIP   10.104.133.209   <none>        80/TCP    9s
kubernetes        ClusterIP   10.96.0.1        <none>        443/TCP   2d17h

$ wget 10.104.133.209

$ cat index.html 
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

# ch9. NodePort
create nginx pod and service
```
#nginx-deploy.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:  #deployment info
  name: nginx-deploy  #name of delplyment
  labels:
    app: nginx    #label of deployment
spec:
  replicas: 3      # create 3 pods
  selector:        # deployment manage the selector
    matchLabels:
      app: nginx
  template:  # pod create option
    metadata:
      labels:
        app: nginx  #lable of the pod
    spec:          #container info
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```
```
kubectl apply -f nginx-deploy.yaml 
```

```
#nginx-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  labels:
    app: nginx
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 80
  selector:
    app: nginx
```
```
kubectl apply -f nginx-svc.yaml
```
```
$kubectl get pods,nodes,services -o wide
NAME                                READY   STATUS             RESTARTS        AGE   IP             NODE      NOMINATED NODE   READINESS GATES
pod/nginx-deploy-86dcfdf4c6-8bkfl   0/1     Running            5 (2m2s ago)    14m   10.244.2.238   worker2   <none>           <none>
pod/nginx-deploy-86dcfdf4c6-ksgkv   0/1     Running            5 (103s ago)    14m   10.244.2.239   worker2   <none>           <none>
pod/nginx-deploy-86dcfdf4c6-x4vjj   0/1     Running            5 (2m35s ago)   14m   10.244.1.34    worker1   <none>           <none>

NAME               STATUS   ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION          CONTAINER-RUNTIME
node/master-node   Ready    control-plane   2d18h   v1.28.2   10.178.0.18   <none>        Debian GNU/Linux 11 (bullseye)   5.10.0-26-cloud-amd64   containerd://1.6.25
node/worker1       Ready    <none>          2d17h   v1.28.2   10.178.0.19   <none>        Debian GNU/Linux 11 (bullseye)   5.10.0-26-cloud-amd64   containerd://1.6.25
node/worker2       Ready    <none>          25h     v1.28.2   10.178.0.20   <none>        Debian GNU/Linux 11 (bullseye)   5.10.0-26-cloud-amd64   containerd://1.6.25

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE     SELECTOR
service/kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP          2d18h   <none>
service/nginx-svc    NodePort    10.104.105.212   <none>        8080:32274/TCP   26m     app=nginx
```

nodePort open port to connect pod.<br>
node(32274) - service(8080) - pod(80).<br>
below command get same result!
```
curl 10.244.2.238:80
curl 10.104.105.212:8080
curl 10.178.0.18:32274
```
if you have external ip on worker node, you can "curl [external-ip]:32274"

# ch10. loadbalancer
```
kubectl expose deployment [deploy-name] --type=LoadBalancer --name=[service-name]
```

# ch11. ingress
install ingress controller
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
```
create pod and service
```
#cafe.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coffee
spec:
  replicas: 2
  selector:
    matchLabels:
      app: coffee
  template:
    metadata:
      labels:
        app: coffee
    spec:
      containers:
      - name: coffe
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: coffee-svc
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: coffee
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tea
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tea
  template:
    metadata:
      labels:
        app: tea
    spec:
      containers:
      - name: tea
        image: nginxdemos/nginx-hello:plain-text
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: tea-svc
  labels:
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: tea
```
```
 kubectl apply -f cafe.yaml
```

create ingress
```
#cafe-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cafe-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - http:
    paths:
    - path: /tea
      pathType: Prefix
      backend:
        service:
          name: tea-svc
          port:
            number: 80
    - path: /coffee
      pathType: Prefix
      backend:
        service:
          name: coffee-svc
          port:
            number: 80
```
check ingress created
```
$ kubectl get ingress
NAME           CLASS    HOSTS   ADDRESS   PORTS   AGE
cafe-ingress   <none>   *                 80      12m
```
we need ADDRESS
```
#ingress-nginx-svc-patch.yaml
spec:
  externalIPs:
  - 10.178.0.19 #external IP
```
```
kubectl patch service ingress-nginx-controller --namespace=ingress-nginx --patch "$(cat ingress-nginx-svc-patch.yaml)"
```
external-ip created.
```
$kubectl get service -n ingress-nginx
 kubectl get service -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.107.90.109    10.178.0.19   80:31772/TCP,443:32706/TCP   45m
ingress-nginx-controller-admission   ClusterIP      10.101.115.137   <none>        443/TCP                      45m
```

check result
```
$ curl http://10.178.0.19:31772/tea
$ curl http://10.178.0.19:31772/coffee
```
when you try this command, the server is changed. load balancing work !

# ch12. limitRange

```
#set-limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: set-limit-range
spec:
  limits:
  - max:
      cou: "800m"  #0.8 cpu core
    min:
      cpu: "200m"  #0.2 cpu core
    type: Conatainer
```
```
$ kubectl apply -f set-limit-range.yaml
$ kubectl describe limitrange set-limit-range
Name:       set-limit-range
Namespace:  default
Type        Resource  Min   Max   Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---   ---   ---------------  -------------  -----------------------
Container   cpu       200m  800m  800m             800m           -
```
```
#pod-with-cpu-range.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-cpu-range
spec:
  containers:
  - name: pod-with-cpu-range
    image: nginx
    resources:
      limits:
        cpu: "800m"
      requests:
        cpu: "500m"  
```

# ch13. autoscaling
install metrics server
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl edit deployments.apps -n kube-system metrics-server
```
change metrics-server config
```
kubectl edit deployments.apps -n kube-system metrics-server
```
```
spec:
  containers:
  - args:
    - --cert-dir=/tmp
    - --secure-port=4443
    - --kubelet-insecure-tls=true # change
    - --kubelet-preferred-address-types=InternalIP # change
```
change kube-apiserver.yaml
```
vi /etc/kubernetes/manifests/kube-apiserver.yaml
```
```
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=172.30.1.26
    - --enable-aggregator-routing=true # change 
```
```
#php-apache.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  replicas: 1
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests: 
            cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache
```
```
kubectl apply -f php-apache.yaml
```

make autoscale
```
 kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```
get hpa(horizontal pod autoscaling
```
$ kubectl get hpa
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        1          33s
```

check pod resource 
```
$ kubectl top pods
NAME                          CPU(cores)   MEMORY(bytes)   
php-apache-5d54745f55-nbh2b   1m           19Mi       
```

```
kubectl run -i --tty load-generator --rm --image=busybox --restart-Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```
with this code, pod cpu usage increase, and num of additional pod added.


# ch14. create docker image
install docker. (if you did not install docker)
```
sudo apt update
sudo apt install -y docker.io
```

create Dockerfile
```
vi Dockerfile
```
```
#Dockerfile

FROM ubuntu:20.04

# update apt
RUN apt-get update && apt-get -y upgrade

# install jre
RUN apt install -y default-jre

# install wget
RUN apt-get -y install wget

# install tomcat
WORKDIR /usr/local
RUN wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.84/bin/apache-tomcat-9.0.84.tar.gz
RUN tar -xvzf apache-tomcat-9.0.84.tar.gz
RUN rm apache-tomcat-9.0.84.tar.gz
RUN mv apache-tomcat-9.0.84 tomcat9

# download war
WORKDIR /usr/local/tomcat9/webapps
RUN git clone https://github.com/hanhunh89/spring-miniBoard ./my
RUN mv ./my/miniboard.war ./miniboard.war
RUN rm -rf my


# expose port
EXPOSE 8080


# clean apt package
RUN rm -rf /var/lib/apt/lists/*

CMD ["/usr/local/tomcat9/bin/catalina.sh", "run"]
```

build docker image
```
sudo docker build -t miniboardimage .
```

run docker container for test
```
docker run -p 8080:8080 miniboardimage
```
```
curl localhost:8080/miniboard 
curl host_ip:8080/miniboard 

```

8080 port of docker container is binded in host(local or server or e2c..) 8080 port.<br>
if you connect http://host_ip:8080/miniboard, you can connect to container.<br>

next, we have to upload our image to docker repository.<br>
kubernetes read image from docker repository.<br>
(you can read image from local repository also, but it is out of this post scope)<br>

before next command, you have to sign up at https://hub.docker.com/ and create repository. <br>
```
$ sudo docker login
```
```
$ sudo docker images
REPOSITORY       TAG       IMAGE ID       CREATED        SIZE
miniboardimage   latest    8c10bc690bd0   24 hours ago   770MB
```
```
$ sudo docker tag miniboardimage:latest embdaramzi/myrepository:miniboard
```
```
$ sudo docker images
REPOSITORY                TAG         IMAGE ID       CREATED        SIZE
embdaramzi/myrepository   miniboard   8c10bc690bd0   24 hours ago   770MB
miniboardimage            latest      8c10bc690bd0   24 hours ago   770MB
```
```
$ sudo docker push embdaramzi/myrepository:miniboard
```
# ch15. create kubernetes pod with customized docker image.

```
#miniboard-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: miniboard-deploy
  labels:
    app: miniboard
spec:
  replicas: 5
  selector:
    matchLabels:
      app: miniboard
  template:
    metadata:
      labels:
        app: miniboard
    spec:
      containers:
      - name: miniboard
        image: embdaramzi/myrepository:miniboard  # we created this docker image at ch.14 create docker image
        ports:
        - containerPort: 8080
```
```
kubectl apply -f miniboard-deploy.yaml
```
```
$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
miniboard-deploy-746f4f6c76-4j8rc   0/1     Pending   0          2m33s
miniboard-deploy-746f4f6c76-khgzl   1/1     Running   0          8m44s
miniboard-deploy-746f4f6c76-n7srq   1/1     Running   0          8m44s
miniboard-deploy-746f4f6c76-r6vjj   1/1     Running   0          8m44s
miniboard-deploy-746f4f6c76-wpp8r   1/1     Running   0          8m44s
```

<br>
할일
1. 도커 레포지토리에 이미지 등록한다. 
2. 쿠버네티스로 이미지 불러온다. 
3. db 이미지를 만든다. 
4. db를 연결할 때 ip가 아니라 호스트명을 사용한다. 
