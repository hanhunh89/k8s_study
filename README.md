# k8s_study

reference : 쉽게 시작하는 쿠버네티스 / 서지영 <br><br>

you can install k8s with https://github.com/hanhunh89/k8s_install
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
