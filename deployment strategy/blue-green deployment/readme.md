## Blue-Green Deployment in Kubernetes

- Prepare a web application (e.g., simple Nginx or your own container with a Node.js/Go/Python app that displays a visible version, such as /version returning "v1" or "v2").
- Create two separate Deployments: `app-blue` (version v1) and `app-green` (version v2) with different labels (e.g., `version: v1` and `version: v2`).
- Create a single Service (type ClusterIP or LoadBalancer) that initially routes all traffic to `app-blue` (selector: version: v1).
- Apply everything using a Helm chart or plain manifests.
- Perform a full Blue-Green switch:
  - Verify that the new version (green) is ready and healthy (all pods Ready).
  - Update the Service selector to route traffic to `app-green` (version: v2).
  - Confirm that all traffic goes to the new version and the old version (blue) receives no traffic.
  - (Optional) After successful verification, delete or scale down the old Deployment to 0 replicas.
- Bonus: Automate the switch using a kubectl script or Argo Rollouts (if you want to go further).




First of all we need to create a *blue* deployment of nginx service.

```yaml
#blue deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-blue
  namespace: blue-green-deployment
  labels:
    version: v1
spec:
  selector:
    matchLabels:
      app: nginx
      version: v1
  template:
    metadata:
      labels:
        app: nginx
        version: v1
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        resources: {}
        ports:
        - containerPort: 80
```

We used `version: v1` to distinguish blue and green version of the app. Also we created a service which distribute all incoming traffic into v1:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: blue-green-deployment
spec:
  selector:
    app: nginx
    version: v1
  ports:
  - port: 8080
    targetPort: 80

```

![alt text](image.png)