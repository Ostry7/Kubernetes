# Minikube deploy on Linux Ubuntu VM (based on Proxmox env)

# Kubernetes Tasks for Minikube

## Task 1 — Cluster Startup [v]

- Install Minikube (any driver).
- Start the cluster with your chosen configuration.
- Check all system components and ensure everything is running correctly.

**Expected result:**
- one node in *Ready* state,
- all core pods running.

---
### Solution:
- Install Minikube:
```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

- Run minikube cluster:
```
minikube start --driver=docker
```

- Install kubectl:
```
sudo snap install kubectl --classic
```

- check Kubernetes status:
```
kubectl get nodes
---->
NAME       STATUS   ROLES           AGE    VERSION
minikube   Ready    control-plane   115s   v1.34.0

kubectl get pods -A
---->
NAMESPACE     NAME                               READY   STATUS    RESTARTS        AGE
kube-system   coredns-66bc5c9577-l2v94           1/1     Running   0               3m2s
kube-system   etcd-minikube                      1/1     Running   0               3m8s
kube-system   kube-apiserver-minikube            1/1     Running   0               3m8s
kube-system   kube-controller-manager-minikube   1/1     Running   0               3m8s
kube-system   kube-proxy-tl7kr                   1/1     Running   0               3m3s
kube-system   kube-scheduler-minikube            1/1     Running   0               3m8s
kube-system   storage-provisioner                1/1     Running   1 (2m31s ago)   3m6s
```

## Task 2 — Simple Application

- Create a Deployment with one replica of any lightweight application.
- Expose it outside the cluster using a *NodePort* Service.
- Verify that the application responds in your browser.
---
### Solution:
*Using nginx*

- create deployment.yml from template:
```
kubectl create deployment nginx --image=nginx:latest --dry-run=client -o yaml > deployment.yml
---->
apiVersion: apps/v1

kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:latest
        name: nginx
        resources: {}
status: {}
```
---
### Apply deployment.yml:
```
kubectl apply -f deployment.yml
```
---
### Check deployment status:
```
kubectl get deployments
kubectl get pods
```

---
### Set NodePort to expose the port outside

To expose the port outside the cluster we need to create _service.yml_ file:
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30036
    protocol: TCP
```

Then just apply the changes deployment:
```
kubectl apply -f deployment.yml
```
and service:
```
kubectl apply -f service.yml
```
### Verification:
Check the services:
```
kubectl get svc
---->
NAME            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
kubernetes      ClusterIP   10.96.0.1     <none>        443/TCP        13d
nginx-service   NodePort    10.98.75.55   <none>        80:30036/TCP   86s
```
Check minikube services:
```
minikube service _appName_
---->
minikube service nginx-service
---->
┌───────────┬───────────────┬─────────────┬───────────────────────────┐
│ NAMESPACE │     NAME      │ TARGET PORT │            URL            │
├───────────┼───────────────┼─────────────┼───────────────────────────┤
│ default   │ nginx-service │ 80          │ http://192.168.49.2:30036 │
└───────────┴───────────────┴─────────────┴───────────────────────────┘
```

### Task 3 Scaling and Rollout [X]
---

- Increase the number of replicas in the Deployment.
- Perform a rolling update to a different image version.
- Roll back the update to the previous version.
- Check the rollout history.
---
### Solution
Increasing the number of replicas in the *deployment.yml* we do this changing the *replicas: X* value
example:
```
[...]
spec:
  replicas: 2 #previous: 1
[...]
```
This changing means that Kubernetes will create as much pods as *replicas* value.

### Check ReplicaSet
```
kubectl get rs
---->
NAME               DESIRED   CURRENT   READY   AGE
nginx-7c5d8bf9f7   2         2         2       22h
```

>How it works? Kubernes checks *DESIRED STATE* (at the begining we had 1pod so kubernetes will create additionally 1pod to achive all pods in desired state.) **All pods are up and running simultaneously.**

### Pods traffic

In our case we have *NodePort* set. It works as a LoadBalancer -> **all incoming traffic evenly directed to all available pods.** It means in our case we have 2 pods one request may go to the first pod another to the second and so on.

### Perform a rolling update to a different image version.

First of all lets change the nginx version:
```
[...]
      containers:
      - image: nginx:1.29 #previous: nginx:latest
        name: nginx
[...]
```

After saving the deployment.yml lets apply the changes:
```
kubectl apply -f deployment.yml
```

To check the changes first we need to grab the pod name:
```
kubectl get pods -l app=nginx -o wide
---->
NAME                   READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
nginx-5b6f67b6-7985b   1/1     Running   0          63s   10.244.0.10   minikube   <none>           <none>
nginx-5b6f67b6-thdsv   1/1     Running   0          66s   10.244.0.9    minikube   <none>           <none>
```

Then we can check nginx version per pod:
```
kubectl describe pod nginx-5b6f67b6-7985b
---->
[...]
Image:          nginx:1.29
[...]
```

If the deployment is updated (i.e. new image version) Kubernetes will perform *rolling update*:
- it will create new pods with new image version one by one.
- it gradually removes old pods only when new ones are working properly.
This ensures that there is always a minimum number of pods available in the cluster and the application does not stop working.

### Rollback the update to the previous version

To rollback the changes we need to check the *rollout history*
```
kubectl rollout history deployment **<deployment-name>**
---->
deployment.apps/nginx 
REVISION  CHANGE-CAUSE
1         <none>
2         Switched to new nginx version
```

Right now we can perform the *rollback update:*
```
kubectl rollout undo deployment nginx --to-revision=**NUMBER OF REVISION**
---->
kubectl rollout undo deployment nginx --to-revision=1
---->
deployment.apps/nginx rolled back
```

Check the rollout status:
```
kubectl rollout status deployment nginx
---->
deployment "nginx" successfully rolled out
```
Then we can again check the nginx image version:
```
kubectl get pods -l app=nginx -o wide
---->
NAME                     READY   STATUS    RESTARTS   AGE    IP            NODE       NOMINATED NODE   READINESS GATES
nginx-7c5d8bf9f7-mskjf   1/1     Running   0          113s   10.244.0.11   minikube   <none>           <none>
nginx-7c5d8bf9f7-mtmlb   1/1     Running   0          108s   10.244.0.12   minikube   <none>           <none>
```

```
kubectl describe pod nginx-7c5d8bf9f7-mskjf
---->
[...]
Image:          nginx:latest
[...]
```

**The rollout won't change deployment.yml or service.yml files. Kubernetes takes this files only as a source of input, not as something that it must update or change later**