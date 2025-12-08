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
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

- Run minikube cluster:
```bash
minikube start --driver=docker
```

- Install kubectl:
```bash
sudo snap install kubectl --classic
```

- check Kubernetes status:
```bash
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
```yml
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
```bash
kubectl apply -f deployment.yml
```
---
### Check deployment status:
```bash
kubectl get deployments
kubectl get pods
```

---
### Set NodePort to expose the port outside

To expose the port outside the cluster we need to create _service.yml_ file:
```yml
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
```bash
kubectl apply -f deployment.yml
```
and service:
```bash
kubectl apply -f service.yml
```
### Verification:
Check the services:
```bash
kubectl get svc
---->
NAME            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
kubernetes      ClusterIP   10.96.0.1     <none>        443/TCP        13d
nginx-service   NodePort    10.98.75.55   <none>        80:30036/TCP   86s
```
Check minikube services:
```bash
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

## Task 3 Scaling and Rollout [v]
---

- Increase the number of replicas in the Deployment.
- Perform a rolling update to a different image version.
- Roll back the update to the previous version.
- Check the rollout history.
---
### Solution
Increasing the number of replicas in the *deployment.yml* we do this changing the *replicas: X* value
example:
```yml
[...]
spec:
  replicas: 2 #previous: 1
[...]
```
This changing means that Kubernetes will create as much pods as *replicas* value.

### Check ReplicaSet
```bash
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
```yml
[...]
      containers:
      - image: nginx:1.29 #previous: nginx:latest
        name: nginx
[...]
```

After saving the deployment.yml lets apply the changes:
```bash
kubectl apply -f deployment.yml
```

To check the changes first we need to grab the pod name:
```bash
kubectl get pods -l app=nginx -o wide
---->
NAME                   READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
nginx-5b6f67b6-7985b   1/1     Running   0          63s   10.244.0.10   minikube   <none>           <none>
nginx-5b6f67b6-thdsv   1/1     Running   0          66s   10.244.0.9    minikube   <none>           <none>
```

Then we can check nginx version per pod:
```bash
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
```bash
kubectl rollout history deployment **<deployment-name>**
---->
deployment.apps/nginx 
REVISION  CHANGE-CAUSE
1         <none>
2         Switched to new nginx version
```

Right now we can perform the *rollback update:*
```bash
kubectl rollout undo deployment nginx --to-revision=**NUMBER OF REVISION**
---->
kubectl rollout undo deployment nginx --to-revision=1
---->
deployment.apps/nginx rolled back
```

Check the rollout status:
```bash
kubectl rollout status deployment nginx
---->
deployment "nginx" successfully rolled out
```
Then we can again check the nginx image version:
```bash
kubectl get pods -l app=nginx -o wide
---->
NAME                     READY   STATUS    RESTARTS   AGE    IP            NODE       NOMINATED NODE   READINESS GATES
nginx-7c5d8bf9f7-mskjf   1/1     Running   0          113s   10.244.0.11   minikube   <none>           <none>
nginx-7c5d8bf9f7-mtmlb   1/1     Running   0          108s   10.244.0.12   minikube   <none>           <none>
```

```bash
kubectl describe pod nginx-7c5d8bf9f7-mskjf
---->
[...]
Image:          nginx:latest
[...]
```

**The rollout won't change deployment.yml or service.yml files. Kubernetes takes this files only as a source of input, not as something that it must update or change later**

## Task 4 - Application Configuration (Secret and ConfigMap) [v]
- Create a ConfigMap containing some application configuration.
- Create a Secret with sensitive data.
- Run an application that reads values from both the ConfigMap and the Secret.
- Verify that the configuration is applied correctly (e.g., via environment variables or mounted files).

### Solution

First of all lets create a deployment without environmental variables:
```yml
###Task 4

apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodatabase
spec:
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:latest
        resources: {0}   
      - name: mongo_express
        image: mongo-express:latest
        resources: {0}      
```

Then we need to create a `configmap.yml` with some env variables:
```yml
config map:
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb
data:
  ME_CONFIG_BASICAUTH_ENABLED: "true"
```
`ConfigMap` is related with *POD* and not related with container. So that container could use `ConfigMap` we need to specify it at `env:` section in `deployment`.

---
### Secrets
Kubernetes Secrets are objects used to store and manage sensitive information such as passwords, OAuth tokens, SSH keys, and API keys. The primary purpose of Secrets is to reduce the risk of exposing sensitive data while deploying applications on Kubernetes.

We need to encode all sensitive data to store in secret objects:
```bash
echo -n "root" | base64
---->
cm9vdA==
```

And then we can add secrets we can create `secret.yml:`

```yml
secret:
apiVersion: v1
kind: Secret
metadata:
  name: mongo-secrets
type: Opaque
data:
  ME_CONFIG_MONGODB_AUTH_DATABASE: ZGI= 
  ME_CONFIG_MONGODB_AUTH_USERNAME: cm9vdA==
  ME_CONFIG_MONGODB_AUTH_PASSWORD: bW9qZWhhc2xvMTIz

```

To use base64 encoded secrets we can add following to deployment.yml:
```yml
       env:
          - name: ME_CONFIG_MONGODB_AUTH_DATABASE
            valueFrom:
              secretKeyRef:
                name: mongo-secrets
                key: ME_CONFIG_MONGODB_AUTH_DATABASE
          - name: ME_CONFIG_MONGODB_AUTH_USERNAME
            valueFrom:
              secretKeyRef:
                name: mongo-secrets
                key: ME_CONFIG_MONGODB_AUTH_USERNAME
          - name: ME_CONFIG_MONGODB_AUTH_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mongo-secrets
                key: ME_CONFIG_MONGODB_AUTH_PASSWORD
```

After that we need to apply all changes. Main sequence of applies:
```bash
kubectl apply -f mongo-secret.yml
kubectl apply -f mongo-configmap.yml
kubectl apply -f deployment.yml
kubectl apply -f service.yml
```

---

## Task 5 - Working with Volumes []

- Create a PersistentVolume using *hostPath*.
- Create a matching PersistentVolumeClaim.
- Attach the volume to a new Pod.
- Write a file inside the Pod and ensure it persists after restarting the Pod.

### Solution

We can use *hostPath* to create a local PersistentVolume. A *hostPath* volume mounts a file or directory from the host node's file system into pod.