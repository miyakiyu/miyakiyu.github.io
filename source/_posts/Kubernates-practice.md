---
title: Kubernates practice
date: 2024-07-29 14:10:49
tags:
- Kubernetes
- Docker Desktop
top: 98
---
In this article I will practice deploy docker container to Kubernetes! [reference](https://docs.docker.com/guides/deployment-orchestration/kube-deploy/)
For more code please visit my article repo:[Github](https://github.com/miyakiyu/DevOps-Exercise/tree/main/Kubernetes%20practice/go)

<!--more-->

## Docker Desktop
Install Docker Desktop following by official page.
In my case I'm using Linux mint, and this is cause the problem, so if you're using Ubuntu-like linux and you using apt to install it will missing some package for docker desktop,here is my solution:
```bash
vim /etc/apt/sources.list.d/docker.list
```
You need to fix the contact like the below one:
```bash
deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu   jammy stable
```
I guess this is cause apt can't get correctly version name so it will cause the error :/

---
## Describe application with Kubernetes 
### Write a Dockerfile
Here is the Dockerfile for go:
```bash
FROM golang:latest

WORKDIR /app

COPY . .

RUN go mod init main
RUN go mod tidy
RUN go build -o main

EXPOSE 3000

CMD ["./main"]
```
And here is main.go, I'm using gin official templete:
```go
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.Run(":3000")
}
```
And we build it:
```bash
docker build -t go .
```
Then we run is as container:
```bash
docker run -d --name go -p 127.0.0.1:3000:3000 go
```
After that we finish the first part!

### Deploy by Kubernetes
We write .yaml file to deploy this service: 
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-demo
  namespace: default
spec:
   replicas: 1
   selector:
      matchLabels:
        web: web
   template:
      metadata:
        labels:
          web: web
      spec:
        containers:
          - name: go
            image: go
            imagePullPolicy: Never
---
apiVersion: v1
kind: Service
metadata:
   name: web-entrypoint
   namespace: default
spec:
   type: NodePort
   selector:
      web: web
   ports:
      - port: 3000
        targetPort: 3000
        nodePort: 30001
```
*The port here need to be the same with main.go, or it won't show the result*
Now we can deploy application with this .yaml file:
```bash
kubectl apply -f web.yaml
```
Now you should see this:
```bash
deployment.apps/web-demo created
service/web-entrypoint created
```
And then we checking the deploy situation:
```bash
kubectl get deployments
```
you should see the pod:
```bash
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
web-demo   0/1     1            0           45s
```
*You should wait READY become 1/1 that your service is totally ready*
For checking all pods:
```bash
kubectl get services
```
You will see the service we just created and k8s service:
```bash
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
kubernetes       ClusterIP   10.96.0.1      <none>        443/TCP          4h58m
web-entrypoint   NodePort    10.104.231.8   <none>        3000:30001/TCP   3m3s
```
Now open "localhost:30001/ping"
you will see:
![pong](/pic/pong.png)

After that we can delete the application:
```bash
kubectl delete -f bb.yaml
```










