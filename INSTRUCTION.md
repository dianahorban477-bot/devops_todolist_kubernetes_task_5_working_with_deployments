# INSTRUCTION.md

## 1. How to Deploy the Application
To deploy the application and its autoscaling configuration to the `mateapp` namespace, run the following commands from the project root directory:
```bash
```bash
kubectl apply -f deployment.yml
kubectl apply -f hpa.yml
```
## 2. Verify that the deployment, pods, and HPA are properly running in the mateapp namespace:

```bash
kubectl get all,hpa -n mateapp
```
## 3. How to Access the Application After Deployment
Testing via port-forward
```bash
kubectl port-forward svc/todoapp-clusterip-service 8080:80 -n mateapp
```
Testing via internal DNS from busybox
Connect to the interactive terminal of the busybox pod:
```bash
kubectl exec -it busybox -n mateapp -- sh
```
Run HTTP requests through the internal cluster DNS name:
```bash
wget -qO- http://todoapp-clusterip-service:80/api/healthz/ready
```
Type exit to disconnect.

Accessing via NodePort
Retrieve the IP address of your Kubernetes node:
```bash
kubectl get nodes -o wide
```

## 4. Rationale for Configuration Choices
Rationale for Resource Requests and Limits

    Resource Requests (cpu: 100m, memory: 128Mi): This is the minimum amount of resources guaranteed to each pod. Django is a lightweight framework in an idle state, so 128Mi of RAM is more than enough for the WSGI/ASGI server to boot, and 100m (0.1 CPU core) ensures smooth container initialization without wasting cluster capacity.

    Resource Limits (cpu: 300m, memory: 256Mi): Limits prevent a single pod from consuming all node resources in case of traffic spikes or unexpected memory leaks. Setting the CPU limit to 300m allows Django to handle short bursts of concurrent heavy requests or database queries, while the 256Mi memory ceiling keeps the application stable under normal load.

Rationale for HPA (Horizontal Pod Autoscaler) Configuration

    Min Pods: 2 / Max Pods: 5: * Having a minimum of 2 pods ensures High Availability (HA). If one pod crashes or restarts, the second one continues to serve traffic, preventing downtime.

        The maximum of 5 pods sets a safe boundary so that even under an intense DDoS attack or extreme load, the cluster won't spin up infinitely and exhaust the underlying cloud infrastructure resources.

    Triggers (CPU & Memory at 70%): Scaling is triggered by both metrics because different types of load affect different resources. Intense API request processing or cryptographic operations spike the CPU, while an increase in active user sessions or data caching increases memory usage. A 70% threshold leaves a 30% safety buffer for existing pods to handle traffic while new pods are being provisioned and passing their readiness probes.

Rationale for Deployment Strategy (Why Such Numbers)

    Strategy Type: RollingUpdate

    maxSurge: 1: This setting tells Kubernetes that during an update, it can create exactly 1 additional pod above the desired number (2). This ensures that we don't overload the cluster node with too many simultaneous container builds during deployment.

    maxUnavailable: 0: This is a crucial setting for zero-downtime updates. It guarantees that 0 pods can be unavailable during the update process. Kubernetes will not destroy any old pods until the newly created pods successfully pass their readinessProbe and are fully ready to take over the traffic. Thus, at least 2 healthy pods are always running.
