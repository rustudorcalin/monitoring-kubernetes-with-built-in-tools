## Introduction

Kubernetes is an open-source orchestration platform for working with containers. As its core it gives us means to do deployments, easy way to scale, and monitoring. In this article we will be talking about the built-in monitoring capabilities of Kubernetes, also showing some demos for better understanding.

## Brief overview of Kubernetes architecture

At the **infrastructure** level, a Kubernetes cluster is a set of physical or virtual machines acting in a specific role. The machines acting in the role of Master act as the brain of all operations and are charged with orchestrating functions managing all containers that run on all of the Nodes.

- Master components manage the lifecycle of a pod
    - apiserver - main component exposing APIs for all the other master components
    - scheduler - uses information in the Pod spec to decide on which Node to run a Pod
    - controller-manager - responsible for Node management (detecting if a Node fails), pod replication, and endpoint creation
    - etcd - key/value store used for storing all internal cluster data

- Node components are worker machines in Kubernetes, managed by the Master. Each node contains the necessary components to run pods:
    - kubelet - handles all communication between the Master and the node on which it is running. It interfaces with the container runtime to deploy and monitor containers
    - kube-proxy: is in charge with maintaining network rules for the node. Also it handles communication between pods, node, and the outside world (internet)
    - container runtime: its purpose is to run containers on the node.

From a **logical** perspective, a Kubernetes deployment is comprised of various components, each serving a specific purpose within the cluster.
- Pods are the basic unit of deployment within Kubernetes. A pod consists of one or more containers that share the same network namespace and IP address.
- Services act like a load balancer. They provide an IP address in front of a pool (set of pods) and also a policy that controls access to them.
- ReplicaSets are controlled by deployments and ensure that the desired number of pods for that deployment are running.
- Namespaces define a logical segregation for different kind of resources like pods and/or services.
- Metadata marks containers based on their deployment characteristics.

## Monitoring Kubernetes

Monitoring an application is absolutely required if we want to anticipate problems and have visibility of potential bottlenecks in a dev or production deployment. 
To help monitoring the cluster and so many moving parts which form a deployment, Kubernetes is shipped with some built-in monitoring capabilities:
- The kubernetes dashboard which gives an overview of the resources running on your cluster. It also gives very basic means of deploying and interacting with those resources.
- Probes (Liveness and Readiness) actively monitor the health of a container.
- Horizontal Pod Autoscaler increases the number of pods if needed (decision taken analyzing different metrics)
- cAdvisor is an open source agent that monitors resource usage and analyzes the performance of containers.

There are a bunch of Kubernetes metrics to monitor. As we've described the architecture splitting it into two (infrastructure and logical) we can do the same with monitoring and separate this into two main components: monitoring the cluster itself and monitoring the pods.

### Cluster monitoring
All clusters should have monitoring of the underlying server components because problems at the server level will show up in the workloads. Some metrics to look for while monitoring node resources - CPU, disk, network bandwidth. Having an overview of these metrics will let you know if it's needed to scale up or down the cluster (useful when using cloud providers where running cost is important)

### Pods monitoring
Metrics to be taken into consideration here can be related to deployments and their pods. Checking the number of pods a deployment has at a moment and its desired state can be relevant. Also we can look for health checks, container metrics and finally application metrics.

## Prerequisites for demo
In the following sections we will take one by one each of the listed built-in monitoring features to see how they can help us. Prerequisites needed for this exercise: 
- a Google Cloud Platform account, the free tier provided is more than enough (any other cloud should work the same)
- a host where Rancher will be running (can be a personal PC/Mac or a VM in a public cloud)
- on same host [Google Cloud SDK](https://cloud.google.com/sdk/install) should be installed along [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/). Make sure that `gcloud` has access to your Google Cloud account by authenticating with your credentials (`gcloud init` and `gcloud auth login`).
- Kubernetes cluster running on Google Kubernetes Engine (running EKS or AKS should be the same)
 
### Starting a Rancher Instance

To begin, start a Rancher instance. There is a very intuitive [getting started guide for Rancher](https://rancher.com/quick-start/) that you can follow for this purpose.

### Using Rancher to deploy a GKE cluster

Use Rancher to set up and configure your Kubernetes cluster, follow the how-to [guide](https://rancher.com/docs/rancher/v2.x/en/cluster-provisioning/hosted-kubernetes-clusters/gke/). 

OBS: please make sure `Kubernetes Dashboard` is enabled and Kubernetes version is `v.1.10`

![03](images/03-rancher-monitoring-create-cluster.png)

## 1. Kubernetes Dashboard

The Kubernetes dashboard is a web-based Kubernetes user interface which we can use to troubleshoot the applications, and manage the cluster resources.
Rancher, as seen above helps us with installing the dashboard only by checking a radio button. Let's take a look now on how the dashboard can help us, by listing some of its uses:
- overview of the cluster resources (overall and individual per node), shows us all the namespaces used, all the storage classes defined
- shows all applications running on the cluster
- provides information on the state of Kubernetes resources in your cluster, and on any errors that may have occurred.

For accessing the dashboard we will need to proxy the server between our machine and Kubernetes API server.
```
$ kubectl proxy &
[1] 3190
$ Starting to serve on 127.0.0.1:8001
```
Now, to view the dashboard in the browser, navigate to the following address in the browser:

```
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

```

![04](images/04-rancher-monitoring-dashboard-login.png)

We will then be prompted with the login page to enter the credentials. Let's take a look on how to create a user with admin permission using Service Account mechanism. We will use two yaml files, one will create the Service Account, the other one will be used to create ClusterRoleBinding for our user.

```
$ cat ServiceAccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system

$ cat ClusterRoleBinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

```
$ kubectl apply -f ServiceAccount.yaml 
serviceaccount "admin-user" created

$ kubectl apply -f ClusterRoleBinding.yaml 
clusterrolebinding.rbac.authorization.k8s.io "admin-user" created
```

As soon as our user is created and the correct permission have been set, we will need to find out the token in order to login:

```
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
Name:         admin-user-token-lnnsn
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=admin-user
              kubernetes.io/service-account.uid=e34a9438-4e12-11e9-a57b-42010aa4009e

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1119 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLWxubnNuIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJlMzRhOTQzOC00ZTEyLTExZTktYTU3Yi00MjAxMGFhNDAwOWUiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.sIsu30ClACmbHl_i9G-PewR2PZ4gPrCzs1ArcNe8oSEtqECZWSeeopV1cYZ-VK02VZWaZSfAnJeydyKNzRwEAHNWakVscAD84eQePNrtiAaXf2C7B6vc0CXMWVkbYdHR0ftR8i3DBsqVpJ32qH7RHqhtLNV316v1cDdI0VJg7RfO_5OQ6prXFYXbNDX9bavPKCkO6-xMMz6FmrzW_JmOnMVjG0Rfamb0sCRkqdNlJmptWGrOroBwItauaCLi510O3h0FxymaHMwTbtxF9V2dHMRvwct3vO_5xkn14e9Bzw__sZHsyKpS2Gt5aHomJ9NTR02T-8uty_w6ko6cBSZ08g
```

Kubernetes Dashboard consists of few main views:

- admin view: lists Nodes, Namespaces, and Persistent Volumes among others. We can get from here aggregated details for our nodes (CPU and memory usage metrics) and individual details view for each node showing the metrics for a Node, its specification, status, allocated resources, events, and pods running on the node.

- workload view: shows all applications running in a selected namespace. It summarizes the actionable information about the workloads, for example, the number of ready pods for a StatefulSet/Deployment or current memory usage for a Pod.

- Discover and Load Balancing view: shows Kubernetes resources that allow for exposing services to the external world and discovering them within a cluster.

- Config and storage view: shows Persistent Volume Claim resources which are used by applications while config view is used to shows all the Kubernetes resources that are used for live configuration of applications running in clusters.

If you want to explore all the views which the dashboard has to offer the best way is to deploy apps with different workload configurations (stateful set, deployments, replica sets etc). You can check this [article](https://rancher.com/blog/2019/deploying-redis-cluster/) to see how to deploy a redis cluster (having a stateful set configuration with volume claims and configMap) plus a testing app (having a deployment configuration) so the dashboard tabs will have some relevant info (otherwise they will be empty as there is nothing deployed on top of kubernetes). Now if we take down one node we can check the different tabs to see some updates:

```
[crus@192-168-0-100 ~]$ kubectl delete pod redis-cluster-2
pod "redis-cluster-2" deleted
[crus@192-168-0-100 ~]$ kubectl get pods
NAME                              READY     STATUS        RESTARTS   AGE
hit-counter-app-9c5d54b99-xv5hj   1/1       Running       0          1h
redis-cluster-0                   1/1       Running       0          1h
redis-cluster-1                   1/1       Running       0          1h
redis-cluster-2                   0/1       Terminating   0          1h
redis-cluster-3                   1/1       Running       0          44s
redis-cluster-4                   1/1       Running       0          1h
redis-cluster-5                   1/1       Running       0          1h
```

![05](images/05-rancher-monitoring-dashboard-statefulSet.png)
![06](images/06-rancher-monitoring-dashboard-pods.png)

## 2. Probes
There are two kind of health checks, liveness and readiness probes.
- readiness probes: their purpose is to let Kubernetes know when the app is ready to serve traffic. So only when the probe passes, Kubernetes will allow the Service to send traffic to the Pod. If the probe fails, Kubernetes will stop sending traffic to that Pod until it passes again. 
These kind of probes are useful when you have an application which takes some good seconds to start. The Service won't work until is up and running even the process has already started. Be default Kubernetes will start sending traffic as soon as the process inside the container is started, but with the readiness probe Kuebernetes will wait until the app is fully started before allows Services to send traffic to it.
- liveness probe: let's Kubenertes know if the app is alive or not. If alive then there is no action taken, but if the app is dead Kubernetes will remove the Pod and start a new one to replace it. These probes are useful when you have an app which can hang in definitely and stop serving requests. Because the process is still running, Kubernetes by default will continue sending requests to this Pod. With these probes Kubernetes will detect that the app is no longer serving requests and will restart the Pod.

Type of probes for both liveness and readiness checks:
- http - most command type of custom probe. Kubernetes pings a path and if it gets a http response of 200-300 range will mark it healthy
- command - for this, Kubernetes will run a command inside a container. If the command returns an exit code 0 the container will be marked healthy
- tcp - Kubernetes will try to establish a tcp connection on a specified port. If it's able to establish it then the container is marked healthy

Configuring probes (parameters to take in consideration):
- initialDelaySeconds - time to wait before sending readiness/liveness probe for a started container. Make sure for liveness probe that the probe will start only after the app is ready, otherwise your app will keep restarting
- periodSeconds - how often the probe is performed (default is 10)
- timeoutSeconds - number of seconds for a probe to timeout (default is 1)
- successThreshold - minimum consecutive success for a probe to be considered successful
- failureThreshold - time before giving up for a started container for which the probe failed. Giving up for liveness probe means restarting the Pod, for readiness will be just marked as Unready.

### 2.1 Readiness probe
In this section we will be playing with a readiness probe, configured using a `command` check. We will have a deployment of 2 replicas (default nginx container) and until there will be no file called `/tmp/healthy` inside the container, no traffic will be sent to either of the two.

```
$ cat readiness.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-demo
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
          - containerPort: 80
        readinessProbe:
          exec:
            command:
            - ls
            - /tmp/healthy
          initialDelaySeconds: 5
          periodSeconds: 5          
---
apiVersion: v1
kind: Service
metadata:
  name: lb
spec:
  type: LoadBalancer
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
      app: nginx
```

Let's apply our yaml file, we will see a deployment and a service being created.

```
$ kubectl apply -f readiness.yml 
deployment.apps "readiness-demo" created
service "lb" created
```

The Pods won't get the `READY` state unless the readiness probe will pass. In this case as there is no file called `/tmp/healthy` it will be marked as failed, so no traffic will be sent by the Service.

```
$ kubectl get deployments
NAME             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
readiness-demo   2         2         2            0           20s

$ kubectl get pods
NAME                              READY     STATUS    RESTARTS   AGE
readiness-demo-6c48bbb79f-xvgsk   0/1       Running   0          23s
readiness-demo-6c48bbb79f-xvr4x   0/1       Running   0          23s
```

For better understanding we will modify the default nxing page for the two Pods, if hit, first one will show `1` as response, the second one will show `2` as a response.

```
$ kubectl exec  -it readiness-demo-6c48bbb79f-xvgsk -- bash -c "echo 1 > /usr/share/nginx/html/index.html"

$ kubectl exec  -it readiness-demo-6c48bbb79f-xvr4x -- bash -c "echo 2 > /usr/share/nginx/html/index.html"
```

For our first Pod, let's create the file, so the state should go into `READY` and traffic will be sent here. The probe runs every 5 seconds, so we will need to wait a bit.

```
$ kubectl exec  -it readiness-demo-6c48bbb79f-xvgsk -- touch /tmp/healthy

$ kubectl get pods
NAME                              READY     STATUS    RESTARTS   AGE
readiness-demo-6c48bbb79f-xvgsk   0/1       Running   0          23m
readiness-demo-6c48bbb79f-xvr4x   0/1       Running   0          23m

$ kubectl get pods
NAME                              READY     STATUS    RESTARTS   AGE
readiness-demo-6c48bbb79f-xvgsk   1/1       Running   0          23m
readiness-demo-6c48bbb79f-xvr4x   0/1       Running   0          23m
```

As soon as the state changes we can start hitting the external IP of our load balancer and we will see our modified nginx page.

```
$ curl 35.204.145.38
1
```
Doing the same for the second Pod (creating the file), the `READY` state will be changed for this as well, and traffic will be redirected here too.

```
$ kubectl exec -it readiness-demo-6c48bbb79f-xvr4x -- touch /tmp/healthy

$ kubectl get pods
NAME                              READY     STATUS    RESTARTS   AGE
readiness-demo-6c48bbb79f-xvgsk   1/1       Running   0          25m
readiness-demo-6c48bbb79f-xvr4x   1/1       Running   0          25m
```
As second Pod is marked now as `READY` the Service will send traffic to both:
```
$ curl 35.204.145.38
2
$ curl 35.204.145.38
1
```

### 2.2 Liveness probe
In this section we will have a demo for a liveness probe, configured with a `tcp` check. Same as above, we will use a deployment of 2 replicas (default nginx container) and if port 80 inside the container will not be listening, traffic will not be sent to the container and this will be restarted.

```
$ cat readiness.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: liveness-demo
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
          - containerPort: 80
        livenessProbe:
          tcpSocket:
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: lb
spec:
  type: LoadBalancer
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
      app: nginx
```

After applying the yaml with a simple `kubectl apply -f readiness.yaml` we can check the Pods and modify like above the default nginx page to respond with a simple `1` or `2`.

```
$ kubectl get pods
NAME                              READY     STATUS    RESTARTS   AGE
liveness-demo-7bdcdd47d9-l8wj8   1/1       Running   0          2m
liveness-demo-7bdcdd47d9-m825b   1/1       Running   0          2m

$ kubectl exec -ti liveness-demo-7bdcdd47d9-l8wj8 -- bash -c "echo 1 > /usr/share/nginx/html/index.html"

$ kubectl exec -ti liveness-demo-7bdcdd47d9-m825b -- bash -c "echo 2 > /usr/share/nginx/html/index.html"
```
Traffic is already redirected from the Service, so we get responses from both Pods.

```
$ curl 35.204.202.158
2
$ curl 35.204.202.158
1
```

Let's stop the process for the first Pod to see the liveness probe in action. As soon as port `80` is not listening anymore, Pod will have the `STATUS` changed and it will get restarted (we can observe some of the statuses for it, until it's back Running correctly)

```
$ kubectl exec -ti liveness-demo-7bdcdd47d9-l8wj8 -- service nginx stop
command terminated with exit code 137

$ kubectl get pods
NAME                              READY     STATUS      RESTARTS   AGE
liveness-demo-7bdcdd47d9-l8wj8   0/1       Completed   2          7m
liveness-demo-7bdcdd47d9-m825b   1/1       Running     0          7m

$ kubectl get pods
NAME                              READY     STATUS             RESTARTS   AGE
liveness-demo-7bdcdd47d9-l8wj8   0/1       CrashLoopBackOff   2          7m
liveness-demo-7bdcdd47d9-m825b   1/1       Running            0          7m

$ kubectl get pods
NAME                              READY     STATUS    RESTARTS   AGE
liveness-demo-7bdcdd47d9-l8wj8   1/1       Running   3          8m
liveness-demo-7bdcdd47d9-m825b   1/1       Running   0          8m
```

Now, if we hit our Service, we will see the correct response from the second Pod (modified previously to `2`) while for the other Pod which has just been created we will get the default nxing page, which comes from the nginx container image.


```
$ curl 35.204.202.158 
2

$ curl 35.204.202.158 
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

## 3. HPA (Horizontal Pod Autoscaler)
Horizontal Pod Autoscaler is a feature of Kubernetes which enables us to have automatically scaled a number of needed Pods for a deployment, replication controller or replica set, based on some observed metrics (mainly CPU, but custom metrics are possible too). Everything is automated, based on the metrics, so there is no human intervention needed. The metrics are fetched from APIs like `metrics.k8s.io`, `custom.metrics.k8s.io` or `external.metrics.k8s.io`.

In this example we will be running a demo based on CPU metrics. A useful command which we can use in this scenario is the `kubectl top pods` command, which shows CPU and memory usage for pods.

First, let's apply the yaml which will create a deployment having only 1 replica.

```
$ cat hpa.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo
spec:
  selector:
    matchLabels:
      app: stress
  replicas: 1
  template:
    metadata:
      labels:
        app: stress
    spec:
      containers:
      - image: nginx
        name: stress
```

```
$ kubectl apply -f hpa.yaml 
horizontalpodautoscaler.autoscaling "hpa-demo" created
```

It will be simple deployment, same nginx image, 1 replica this time:

```
$ kubectl get deployment
NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
hpa-demo   1         1         1            1           38s
```

Let's see now how can we create an autoscaling mechanism. This can be achieved by using commands like `kubectl create/delete`. Also we can list the autoscalers with `kubectl get/describe hpa`.
But the easiest way to create an autoscaler is shown below. This will create an autoscaler for our deployment called hpa-demo (which we earlier created), with target CPU utilization set to 50%. The replica number is set here to be between 1 and 10, so maximum number of Pods we will get in case of higher load, will be limited to 10.

```
$ kubectl autoscale deployment hpa-demo --cpu-percent=50 --min=1 --max=10
deployment.apps "hpa-demo" autoscaled

$ kubectl get hpa
NAME       REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
hpa-demo   Deployment/hpa-demo   0%/50%    1         10        1          23s
```

We can have this in a yaml format too, not only from a command line.

```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-demo
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-demo
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

In order to see HPA in action we need to run some command which creates load on the CPU. There are numerous ways to achieve this, one very simple would be:

```
while true; do sleep 1 && date & done
```

Let's check first the load on our only Pod. As this currently sits idle, there is not much going:
```
$ kubectl top pods
NAME                        CPU(cores)   MEMORY(bytes)   
hpa-demo-7c68555d8b-6hjvj   0m           1Mi
```

Now, let's generate some load on the only Pod currently available (let the command run for few seconds then stop it). As soon as load increases we should see some Pods automatically created by HPA to handle this traffic increase.

```
kubectl exec -it hpa-demo-7c68555d8b-6hjvj  -- bash -c "while true; do sleep 1 && date & done"
```

The current load on the single Pod:

```
$ kubectl top pods
NAME                        CPU(cores)   MEMORY(bytes)   
hpa-demo-7c68555d8b-6hjvj   104m         3Mi
```

HPA kicks in and starts creating extra Pods, we see now the deployment has been automatically scaled and now has 3 replicas.

```
$ kubectl get deployments
NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
hpa-demo   3         3         3            2           4m
```

```
$ kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
hpa-demo-7c68555d8b-6hjvj   1/1       Running   0          5m
hpa-demo-7c68555d8b-9b7dn   1/1       Running   0          58s
hpa-demo-7c68555d8b-lt7t2   1/1       Running   0          58s
```

We can see the details of our HPA and the reason why this has been scaled to 3 replicas:

```
$ kubectl describe hpa hpa-demo
Name:                                                  hpa-demo
Namespace:                                             default
Labels:                                                <none>
Annotations:                                           kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"autoscaling/v1","kind":"HorizontalPodAutoscaler","metadata":{"annotations":{},"name":"hpa-demo","namespace":"default"},"spec":{"maxRepli...
CreationTimestamp:                                     Sat, 30 Mar 2019 17:43:50 +0200
Reference:                                             Deployment/hpa-demo
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  104% (104m) / 50%
Min replicas:                                          1
Max replicas:                                          10
Conditions:
  Type            Status  Reason              Message
  ----            ------  ------              -------
  AbleToScale     True    ReadyForNewScale    recommended size matches current size
  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  False   DesiredWithinRange  the desired count is within the acceptable range
Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  SuccessfulRescale  15s   horizontal-pod-autoscaler  New size: 3; reason: cpu resource utilization (percentage of request) above target
```  

Let's wait 5 minutes to see how HPA will scale down the number of replicas as there is no more load, so no need for the extra 2 Pods created. Five minutes is the default time in Kubernetes the autoscaler has to wait before doing a downscale operation (`--horizontal-pod-autoscaler-downscale-delay`, more details about this can be found [here](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/))

```
$ kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
hpa-demo-7c68555d8b-6hjvj   1/1       Running   0          9m
hpa-demo-7c68555d8b-9b7dn   1/1       Running   0          5m
hpa-demo-7c68555d8b-lt7t2   1/1       Running   0          5m

[crus@192-168-0-100 ~]$ kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
hpa-demo-7c68555d8b-6hjvj   1/1       Running   0          9m
```

Checking now the description for the HPA we see the downscale reason:

```
$ kubectl describe hpa hpa-demo
Name:                                                  hpa-demo
Namespace:                                             default
Labels:                                                <none>
Annotations:                                           kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"autoscaling/v1","kind":"HorizontalPodAutoscaler","metadata":{"annotations":{},"name":"hpa-demo","namespace":"default"},"spec":{"maxRepli...
CreationTimestamp:                                     Sat, 30 Mar 2019 17:43:50 +0200
Reference:                                             Deployment/hpa-demo
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  0% (0) / 50%
Min replicas:                                          1
Max replicas:                                          10
Conditions:
  Type            Status  Reason            Message
  ----            ------  ------            -------
  AbleToScale     True    SucceededRescale  the HPA controller was able to update the target scale to 1
  ScalingActive   True    ValidMetricFound  the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  True    TooFewReplicas    the desired replica count is increasing faster than the maximum scale rate
Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  SuccessfulRescale  5m    horizontal-pod-autoscaler  New size: 3; reason: cpu resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  13s   horizontal-pod-autoscaler  New size: 1; reason: All metrics below target
```

## 4. cAdvisor
cAdvisor is an open source agent that monitors resource usage and analyzes the performance of containers being integrated into the Kubelet binary. It collects statistics about the CPU, memory, file and network usage ("core" metrics) for all containers running on a given node (it does not operate at Pod level). Also in addition to the core metrics it monitors events as well. Metrics can be accessed directly, using commands like `kubectl top` or used by the scheduler to perform orchestration (for example scheduling or autoscaling - HPA).
As an observation, cAdvisor doesn't store metrics for long-term use, so if you want this you'll need to look for a dedicated monitoring tool to do this.


If you are running a Kubernetes v1.10 you will be able to see the cAdvisor UI. This feature has been marked deprecated in this version and it is completely removed in v1.12. Rancher gives you the possibility to choose the version you want for your Kubernetes cluster (remember as in our prerequisites we configured a cluster running v1.10).

For accessing the cAdvisor UI we will need to proxy the server between our machine and Kubernetes API server.
```
$ kubectl proxy &
[1] 3190
$ Starting to serve on 127.0.0.1:8001
```
Now, to view the UI in the browser, navigate to the following address in the browser (the node name is obtained with `kubectl get nodes`:

```
http://localhost:8001/api/v1/nodes/gke-c-plnf4-default-pool-5eb56043-23p5:4194/proxy/containers/
```

![07](images/07-rancher-monitoring-cAdvisor-UI.png)
![08](images/08-rancher-monitoring-cAdvisor-UI2.png)

If you are curious about the process running on port 4194, here are the steps to follow:

```
$ gcloud compute ssh admin@gke-c-plnf4-default-pool-5eb56043-23p5 --zone europe-west4-c
 
Welcome to Kubernetes v1.10.12-gke.7!

You can find documentation for Kubernetes at:
  http://docs.kubernetes.io/

The source for this release can be found at:
  /home/kubernetes/kubernetes-src.tar.gz
Or you can download it at:
  https://storage.googleapis.com/kubernetes-release-gke/release/v1.10.12-gke.7/kubernetes-src.tar.gz

It is based on the Kubernetes source at:
  https://github.com/kubernetes/kubernetes/tree/v1.10.12-gke.7

For Kubernetes copyright and licensing information, see:
  /home/kubernetes/LICENSES

admin@gke-c-plnf4-default-pool-5eb56043-23p5 ~ $ sudo su -
gke-c-plnf4-default-pool-5eb56043-23p5 ~ # netstat -anp | grep LISTEN | grep 4194
tcp6       0      0 :::4194                 :::*                    LISTEN      1060/kubelet 

```

If you are running a Kubernetes v1.12, the UI has been deprecated, kubelet is not listening on port 4194 anymore (can be checked with above commands), but the metrics are still there, cAdvisor being part of the kubelet binary.

The kubelet exposes all of its runtime metrics, and all of the cAdvisor metrics, on a `/metrics` endpoint in the Prometheus exposition [format](https://github.com/prometheus/docs/blob/master/content/docs/instrumenting/exposition_formats.md).

```
http://localhost:8001/api/v1/nodes/gke-c-plnf4-default-pool-5eb56043-23p5/proxy/metrics/cadvisor
```

![09](images/09-rancher-monitoring-metrics-cAdvisor.png)

Metrics to look for:
- CPU:
    - container_cpu_user_seconds_total  - Cumulative "user" cpu time consumed in seconds
    - container_cpu_system_seconds_total  - Cumulative "system" cpu time consumed in seconds
    - container_cpu_usage_seconds_total  - Cumulative cpu time consumed in seconds (sum of the above)

- Memory:
    - container_memory_cache - Number of bytes of page cache memory
    - container_memory_swap - container swap usage in bytes
    - container_memory_usage_bytes - Current memory usage in bytes, including all memory regardless of when it was accessed
    - container_memory_max_usage_bytes - maximum memory usage in byte
                                   
- Disk: 
    - container_fs_io_time_seconds_total - count of seconds spent doing I/Os
    - container_fs_io_time_weighted_seconds_total - cumulative weighted I/O time in seconds 
    - container_fs_writes_bytes_total - cumulative count of bytes written
    - container_fs_reads_bytes_total - cumulative count of bytes read

- Network:
    - container_network_receive_bytes_total - Cumulative count of bytes received
    - container_network_receive_errors_total - Cumulative count of errors encountered while receiving
    - container_network_transmit_bytes_total - Cumulative count of bytes transmitted
    - container_network_transmit_errors_total - Cumulative count of errors encountered while transmitting

Also usefull metrics can be found here:
- `/healthz` endpoint for determining whether cAdvisor is healthy
- `/healthz/ping` to check connectivity to etcd
- `/spec` endpoint returns the cAdvisor MachineInfo()

`http://localhost:8001/api/v1/nodes/gke-c-plnf4-default-pool-5eb56043-23p5:10255/proxy/spec/`

![10](images/10-rancher-monitoring-metrics-spec.png)

- `pods` endpoint provides asme output as `kubectl get pods -o json` for the pods running on this node

`http://localhost:8001/api/v1/nodes/gke-c-plnf4-default-pool-5eb56043-23p5:10255/proxy/pods/`

![11](images/11-rancher-monitoring-metrics-pods.png)

- logs can also be retrieved `http://localhost:8001/logs/kube-apiserver.log`

## Conclusion

We've seen how Kubernetes helps us using the built-in tools to set up monitoring for our cluster. We've seen how it works nonstop behind the scenes to keep our apps running, but this doesn't mean that we shouldn’t be aware of what's happening. Gathering all these data from the Dashboard, from the probes and having all these container resources exposed by cAdvisor, can help us investigate resource limitations and/or capacity planning.
Monitoring Kubernetes is vital as it helps us understand the health and performance of a cluster and the applications running on top of it.
