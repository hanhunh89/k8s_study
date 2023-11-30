# k8s_study

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
