---
title: Minikube with Kubernetes
date: 2024-07-22 11:51:21
tags:
- Kubernetes
- Minikube
top: 98
---
In this ariticle I will recording some minikube practices and some notes!
The practice is from [official website](https://minikube.sigs.k8s.io/docs/tutorials/kubernetes_101/module1/) :D

<!--more-->
## Kubernates 101
### Create cluster
First we start the minikube:
``` bash
minikube start
```
And you should see this:
``` bash
😄  minikube v1.33.1 on Linuxmint 21.3
✨  Automatically selected the docker driver
📌  Using Docker driver with root privileges
👍  Starting "minikube" primary control-plane node in "minikube" cluster
🚜  Pulling base image v0.0.44 ...
🔥  Creating docker container (CPUs=2, Memory=2900MB) ...
🐳  Preparing Kubernetes v1.30.0 on Docker 26.1.1 ...
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔗  Configuring bridge CNI (Container Networking Interface) ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```
After that, we can use this command to check the cluster we just created:
```bash
kubectl cluster-info
```
```bash
Kubernetes control plane is running at https://192.168.49.2:8443
CoreDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```
And use this command to see the nodes:
```bash
kubectl get nodes
```
```bash
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   28m   v1.30.0
```
### Deploy 
The general kubectl command looks like:
```bash
kubectl [action][resources]
```
Deploy useing this command:
```bash
kubectl create deployment test --image=gcr.io/k8s-minikube/kubernetes-bootcamp:v1
```
Using this command for checking your deplotment:
```bash
kubectl get deployments
```
You might see this, if ready is 0/1 just wait a minute:
```bash
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
test   1/1     1            1           2m2s
```
In general pod run in private network created by Kubernates, you can't access from outsiude, but we can access by using kubectl proxy:
```bash
echo -e "Starting Proxy. After starting it will not output a response. Please return to your original terminal window\n"; kubectl proxy
```
You should see the output by using other terminal:
```bash
curl http://localhost:8001/version
```
```bash
{
  "major": "1",
  "minor": "30",
  "gitVersion": "v1.30.0",
  "gitCommit": "7c48c2bd72b9bf5c44d21d7338cc7bea77d0ad2a",
  "gitTreeState": "clean",
  "buildDate": "2024-04-17T17:27:03Z",
  "goVersion": "go1.22.2",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```
### Explore 
Checking the application running in your pod:
```bash
kubectl describe pods
```
Using this command to review the log:
```bash
kubectl logs $POD_NAME
```
And you can using this commad to get inside the pod's container, just like docker container:
```bash
kubectl exec -ti $POD_NAME -- bash
```
And we see our test pod is already start runnung and we can run this command to see the server content:
```bash
cat server.js
```
And we use this command to checking the connection is working, but remember you can't access this service outside the k8s network:
```bash
root@test-588cd7b956-f2668:/# curl localhost:8080
Hello Kubernetes bootcamp! | Running on: test-588cd7b956-f2668 | v=1
```
### Expose
If we want to expose our service to other users we could do this:
```bash
kubectl expose deployment/test --type="NodePort" --port 8080
```
And we see the service is increase:
```bash
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP          21h
test         NodePort    10.101.129.172   <none>        8080:30627/TCP   5m20s
```
If you want to found the outside port:
```bash
kubectl describe services/test
```
You will see:
```bash
Name:                     test
Namespace:                default
Labels:                   app=test
Annotations:              <none>
Selector:                 app=test
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.101.129.172
IPs:                      10.101.129.172
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  30627/TCP
Endpoints:                10.244.0.5:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```
### Scale up
We can use this command to expand our service:
```bash
kubectl scale deployments/test --replicas=3
```
And checking if this is working:
```bash
miyakiyu@miyakiyu: kubectl get deployments
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
test   3/3     3            3           4h32m
```
You can see we have 3 clusters, and let's checking pod:
```bash
kubectl get pods -o wide
```
```bash
NAME                    READY   STATUS    RESTARTS   AGE     IP           NODE       NOMINATED NODE   READINESS GATES
test-588cd7b956-f2668   1/1     Running   0          4h34m   10.244.0.5   minikube   <none>           <none>
test-588cd7b956-nktpq   1/1     Running   0          3m8s    10.244.0.7   minikube   <none>           <none>
test-588cd7b956-zt24j   1/1     Running   0          3m8s    10.244.0.6   minikube   <none>           <none>
```
We see that our pod expaning and have different IP.
Let's see if this pod have load balancing:
```bash
miyakiyu@miyakiyu: curl $(minikube ip):$NODE_PORT
-> Hello Kubernetes bootcamp! | Running on: test-588cd7b956-nktpq | v=1
miyakiyu@miyakiyu: curl $(minikube ip):$NODE_PORT
-> Hello Kubernetes bootcamp! | Running on: test-588cd7b956-f2668 | v=1
miyakiyu@miyakiyu: curl $(minikube ip):$NODE_PORT
-> Hello Kubernetes bootcamp! | Running on: test-588cd7b956-zt24j | v=1
```
Can see that our request will give to diffferent pod.
If you think 3 services are too much then you could reduce the services:
```bash
kubectl scale deployments/test --replicas=2
```
```bash
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
test   2/2     2            2           4h40m
```
```bash
NAME                    READY   STATUS    RESTARTS   AGE     IP           NODE       NOMINATED NODE   READINESS GATES
test-588cd7b956-f2668   1/1     Running   0          4h40m   10.244.0.5   minikube   <none>           <none>
test-588cd7b956-nktpq   1/1     Running   0          9m15s   10.244.0.7   minikube   <none>           <none>
```
We have close a service.
### Update
Let's update our image to v2, first checking our version:
```bash
kubectl describe pods
```
```bash
...
Containers:
  kubernetes-bootcamp:
    Container ID:   docker://172a20ba85ba09c0b4bbc6b481ac3be1ca1230eb80068b94d024cdfc9c929dc1
    Image:          gcr.io/k8s-minikube/kubernetes-bootcamp:v1
...
```
We see our image version is v1, let's update it to v2!
```bash
kubectl set image deployments/test kubernetes-bootcamp=gcr.io/k8s-minikube/kubernetes-bootcamp:v2
-> deployment.apps/test image updated
```
This is our new version:
```bash
NAME                    READY   STATUS    RESTARTS   AGE
test-6cd499dfd9-69hvr   1/1     Running   0          90s
test-6cd499dfd9-pp2xp   1/1     Running   0          69s
```
Let's verify our version is v2:
```bash
export NODE_PORT=$(kubectl get services/test -o go-template='{{(index .spec.ports 0).nodePort}}')
```
```bash
curl $(minikube ip):$NODE_PORT
```
```bash
miyakiyu@miyakiyu: curl $(minikube ip):$NODE_PORT
-> Hello Kubernetes bootcamp! | Hostname: test-6cd499dfd9-pp2xp | v=2
miyakiyu@miyakiyu: curl $(minikube ip):$NODE_PORT
-> Hello Kubernetes bootcamp! | Hostname: test-6cd499dfd9-69hvr | v=2
```
We see our version are all updating to v2.
Now let's rollback the update, assume we update it to v10:
```bash
kubectl set image deployments/test kubernetes-bootcamp=gcr.io/k8s-minikube/kubernetes-bootcamp:v10
```
let's see the deployment:
```bash
kubectl get deployments
```
```bash
NAME                    READY   STATUS             RESTARTS   AGE
test-6cd499dfd9-69hvr   1/1     Running            0          7m18s
test-6cd499dfd9-pp2xp   1/1     Running            0          6m57s
test-864784584d-9slx6   0/1     ImagePullBackOff   0          2m9s
```
We see "ImagePullBackOff",let's checking logs:
```bash
kubectl describe pods
```
```bash
...
Warning  Failed     82s (x4 over 2m59s)  kubelet            Failed to pull image "gcr.io/k8s-minikube/kubernetes-bootcamp:v10": Error response from daemon: manifest for gcr.io/k8s-minikube/kubernetes-bootcamp:v10 not found: manifest unknown: Failed to fetch "v10" from request "/v2/k8s-minikube/kubernetes-bootcamp/manifests/v10".
  Warning  Failed     82s (x4 over 2m59s)  kubelet            Error: ErrImagePull
  Warning  Failed     69s (x6 over 2m58s)  kubelet            Error: ImagePullBackOff
...
```
So we see that, we can found v10 in repo, now we need to rollback to v2, using undo command:
```bash
kubectl rollout undo deployments/test               
-> deployment.apps/test rolled back
```
And describe pods again to see all pod working fine, we rollback sucesful :D