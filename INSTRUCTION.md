# INSTRUCTION.md

## 1. How to Deploy the Application
To deploy the application and its autoscaling configuration to the `mateapp` namespace, run the following commands from the project root directory:
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

    Resource Requests (cpu: 100m, memory: 128Mi): This defines the minimum resources guaranteed to each pod upon creation. Django in an idle or startup state is lightweight; 128Mi of RAM is fully sufficient for the WSGI/ASGI server process to initialize, and 100m (0.1 CPU core) ensures predictable pod scheduling across the cluster nodes without over-provisioning.

    Resource Limits (cpu: 300m, memory: 256Mi): Limits prevent a single container from consuming unlimited node resources in the event of unexpected traffic spikes or runtime memory growth. A CPU limit of 300m allows Django to handle temporary bursts of API requests or database operations smoothly, while the 256Mi memory ceiling keeps the pod contained and stable.

Rationale for HPA (Horizontal Pod Autoscaler) Configuration

    Min Pods: 2 / Max Pods: 5: * Maintaining a minimum of 2 pods guarantees High Availability (HA) even when the application is idle. If one pod fails or undergoes a restart, the remaining pod continues to handle the incoming traffic.

        Setting a maximum of 5 pods establishes a safe structural limit, ensuring the cluster can scale up to absorb significant traffic loads without infinitely consuming underlying cloud infrastructure resources.

    Triggers (CPU & Memory at 70%): Dual-metric scaling provides comprehensive protection. CPU-intensive operations (such as serialization or requests processing) will trigger scaling via the CPU threshold, whereas sudden growth in concurrent active sessions or data caching will trigger scaling via memory usage. The 70% threshold leaves a safe 30% operational buffer while new pods are being provisioned and passing their readiness probes.

Rationale for Deployment Strategy Configuration

    Strategy Type: RollingUpdate

    maxSurge: 1: Instructs Kubernetes to create at most 1 additional pod above the desired state (2) during a version update. This single-pod buffer ensures the node is never overloaded with too many simultaneous container creations during deployment.

    maxUnavailable: 0: This parameter is critical for enforcing a strict zero-downtime deployment. It ensures that 0 pods can be taken offline during an update. Kubernetes will only terminate an old pod after a newly created pod successfully passes its readinessProbe and is verified as ready to receive traffic.