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
