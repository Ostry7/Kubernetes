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