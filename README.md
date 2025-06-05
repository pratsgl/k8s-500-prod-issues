# k8s-500-prod-issues

This repository documents common 500-level (server-side) errors encountered in Kubernetes production environments and provides solutions or troubleshooting steps for each.

---

## Common 500-Level Issues in K8s Production

### 1. `ImagePullBackOff` / `ErrImagePull`

**Description:**
The Kubernetes scheduler tries to pull an image for a Pod, but it fails. This often results in a 5xx error for the application if it cannot start.

**Possible Causes:**
* **Incorrect Image Name/Tag:** Typo in the image name or tag.
* **Image Not Found:** The image does not exist in the specified registry.
* **Private Registry Authentication Failure:** Missing or incorrect image pull secrets for a private registry.
* **Network Issues:** Kubernetes nodes cannot reach the image registry.
* **Registry Rate Limits:** Exceeding Docker Hub or other registry pull limits.

**Troubleshooting & Solution:**
* Check Pod status: `kubectl get pod <pod-name> -o yaml`
* Describe Pod events: `kubectl describe pod <pod-name>` (Look for `Events` section)
* Verify image name and tag in deployment YAML.
* Ensure `imagePullSecrets` are correctly configured and applied for private registries.
* Check network connectivity from node to registry.
* Consider using a local image cache or a paid tier for registries with rate limits.

---

### 2. `CrashLoopBackOff`

**Description:**
A Pod starts, crashes, Kubernetes restarts it, it crashes again, and this cycle repeats. This is a common cause of application unavailability and 5xx errors.

**Possible Causes:**
* **Application Bug:** The application code has a bug that causes it to crash on startup or during operation.
* **Misconfiguration:** Incorrect environment variables, missing configuration files, or invalid command-line arguments.
* **Resource Limits:** The container might be running out of memory or CPU, leading to crashes.
* **Dependency Issues:** External services (databases, message queues) are unavailable or misconfigured.
* **Missing Files/Permissions:** Application expects files that aren't present or has permission issues accessing them.

**Troubleshooting & Solution:**
* Check Pod status: `kubectl get pod <pod-name>`
* Describe Pod events: `kubectl describe pod <pod-name>`
* View container logs: `kubectl logs <pod-name> -c <container-name>` (Crucial for application-specific errors)
* Check resource usage: `kubectl top pod <pod-name>`
* Review application configuration (ConfigMaps, Secrets, environment variables).
* Test application locally or in a non-K8s environment if possible to isolate the issue.
* Increase resource limits/requests in Pod definition if resource exhaustion is suspected.

---

### 3. `OOMKilled` (Out Of Memory Killed)

**Description:**
The operating system killed a container due to it exceeding its memory limit. This is a specific type of `CrashLoopBackOff` often reported as exit code 137.

**Possible Causes:**
* **Insufficient Memory Limit:** The `resources.limits.memory` defined for the container is too low for the application's actual memory consumption.
* **Memory Leak:** The application has a memory leak, causing its memory usage to continuously grow until it hits the limit.
* **Spike in Memory Usage:** A sudden surge in traffic or processing demands causes a temporary spike in memory that exceeds the limit.

**Troubleshooting & Solution:**
* Describe Pod events: `kubectl describe pod <pod-name>` (Look for "OOMKilled" in events)
* Check container logs: `kubectl logs <pod-name> -c <container-name>` (Often, the application doesn't log anything before being killed)
* Monitor memory usage: Use monitoring tools (Prometheus, Grafana, Datadog) to observe actual memory consumption over time.
* Increase `resources.limits.memory` gradually and monitor.
* Profile the application for memory leaks.
* Consider `resources.requests.memory` to ensure the scheduler allocates enough memory.

---

### 4. `Evicted`

**Description:**
A Pod is terminated and moved from a node by the Kubelet because the node is running low on resources (memory, disk space, inodes).

**Possible Causes:**
* **Node Resource Pressure:** The node is running out of memory, disk space, or inodes.
* **Burstable QoS Class:** Pods with Burstable QoS can be evicted if their requests are met but they exceed limits and the node is under pressure.
* **Node-level issues:** Logs on the node might indicate underlying problems.

**Troubleshooting & Solution:**
* Check Pod status: `kubectl get pod <pod-name>`
* Describe Pod events: `kubectl describe pod <pod-name>` (Look for "Evicted" in events)
* Check node conditions: `kubectl describe node <node-name>`
* Monitor node resource usage (CPU, memory, disk, inodes).
* Ensure Pods have appropriate `resources.requests` and `limits` to give them a stable QoS.
* Use `PodDisruptionBudgets` (PDBs) to manage voluntary disruptions.
* Add more nodes or scale down workloads on affected nodes.

---

### 5. `Liveness/Readiness Probe Failures`

**Description:**
Kubernetes uses probes to determine if a container is running (liveness) or ready to serve traffic (readiness). If these probes fail, Kubernetes might restart the Pod (`liveness`) or take it out of the service endpoint list (`readiness`), leading to 5xx errors for clients.

**Possible Causes:**
* **Application Unresponsive:** The application inside the container is frozen, in a deadlock, or too busy to respond to the probe.
* **Incorrect Probe Path/Port:** The HTTP/TCP probe path or port is wrong or not exposed by the application.
* **Insufficient Initial Delay:** The application takes longer to start than the `initialDelaySeconds` allows, causing probes to fail prematurely.
* **Resource Starvation:** The application is struggling due to lack of CPU/memory, causing it to respond slowly or not at all.

**Troubleshooting & Solution:**
* Check Pod status: `kubectl get pod <pod-name>`
* Describe Pod events: `kubectl describe pod <pod-name>` (Look for "Liveness probe failed" or "Readiness probe failed")
* Inspect container logs: `kubectl logs <pod-name> -c <container-name>` (The application might log why it's unresponsive)
* Verify probe configuration (path, port, `initialDelaySeconds`, `periodSeconds`, `failureThreshold`).
* Test the probe endpoint manually (e.g., `curl` from within another Pod).
* Adjust resource limits if starvation is suspected.
* Differentiate between Liveness and Readiness probes: Readiness for traffic readiness, Liveness for application health and restart.

---

### 6. `Node Not Ready` / `Node Cordoned`

**Description:**
The node where the Pod is running (or supposed to run) becomes unhealthy or is manually cordoned, preventing new Pods from being scheduled or causing existing Pods to be evicted.

**Possible Causes:**
* **Node Health Issues:** Underlying infrastructure problems (network, hardware, OS).
* **Kubelet Issues:** Kubelet process crashed or is unresponsive on the node.
* **Network Partition:** Node loses network connectivity to the API server or other nodes.
* **Manual Cordon:** An administrator manually cordoned the node for maintenance.

**Troubleshooting & Solution:**
* Check node status: `kubectl get nodes` (Look for `NotReady` or `SchedulingDisabled` status)
* Describe node conditions: `kubectl describe node <node-name>` (Provides detailed information on why the node is not ready)
* Check Kubelet logs on the node.
* Verify network connectivity from the node.
* If cordoned, uncordon after maintenance: `kubectl uncordon <node-name>`
* Investigate underlying infrastructure issues if `NotReady` persists.

---

### 7. Full Disk on Node

**Description:**
A node runs out of disk space, preventing new images from being pulled, new logs from being written, or even the Kubelet from functioning correctly. This often leads to `ImagePullBackOff`, `Evicted` Pods, or `Node Not Ready` states.

**Possible Causes:**
* **Excessive Container Logs:** Unmanaged application logs filling up the disk.
* **Too Many Images:** Docker/containerd images consuming a lot of space.
* **Volume Mount Issues:** Persistent Volumes or other mounted volumes filling up the underlying node disk if not properly managed or if they are host-path volumes.
* **System Logs/Other Data:** OS logs or other system files consuming too much space.

**Troubleshooting & Solution:**
* SSH into the affected node.
* Check disk usage: `df -h`
* Identify large directories: `du -sh /*` or `du -sh /var/log`, `/var/lib/docker` (or `/var/lib/containerd`)
* Clean up old Docker/containerd images: `docker system prune` or `crictl rmi --prune`
* Configure log rotation for container logs (e.g., using `logrotate` on the node or directly in your container runtime configuration).
* Ensure Pods are not writing excessive data to host paths without proper volume management.

---

### 8. Service/Endpoint Misconfiguration

**Description:**
While Pods might be running fine, the Kubernetes Service object might not be correctly routing traffic to them, or the Endpoints object is not updated, causing 5xx errors for clients trying to access the service.

**Possible Causes:**
* **Label Selector Mismatch:** The `selector` in the Service definition does not match the `labels` on the Pods.
* **Incorrect Target Port:** The `targetPort` in the Service definition doesn't match the port the application listens on in the container.
* **Readiness Probe Failure:** If a readiness probe fails, the Pod is removed from the Service's endpoints.
* **Network Policies:** Network policies are inadvertently blocking traffic to the Pods.
* **Kube-proxy Issues:** The `kube-proxy` component on nodes is not correctly updating IPtables/IPVS rules.

**Troubleshooting & Solution:**
* Check Service definition: `kubectl get service <service-name> -o yaml`
* Verify Pod labels: `kubectl get pod <pod-name> --show-labels`
* Inspect Endpoints: `kubectl get endpoints <service-name>` (Ensure healthy Pod IPs are listed)
* Check Pod logs and `describe pod` to ensure readiness probes are passing.
* Review Network Policies: `kubectl get networkpolicy -o yaml`
* Debug `kube-proxy` logs on affected nodes.

---

### 9. DNS Resolution Issues

**Description:**
Applications inside Pods cannot resolve service names, external hostnames, or other DNS entries, leading to connection failures and 5xx errors within the application.

**Possible Causes:**
* **CoreDNS Issues:** CoreDNS Pods are unhealthy or misconfigured.
* **Network Policy Blocking DNS:** Network policies are preventing DNS queries from reaching CoreDNS.
* **Incorrect Pod DNS Configuration:** `dnsPolicy` or `dnsConfig` in the Pod spec is misconfigured.
* **Upstream DNS Server Issues:** CoreDNS cannot resolve external names due to issues with its configured upstream DNS servers.

**Troubleshooting & Solution:**
* Check CoreDNS Pods status: `kubectl get pods -n kube-system -l k8s-app=kube-dns`
* Check CoreDNS logs: `kubectl logs -n kube-system <coredns-pod>`
* Test DNS resolution from within a problematic Pod: `kubectl exec -it <pod-name> -- nslookup <service-name>` or `curl <external-hostname>`
* Verify Network Policies.
* Ensure `dnsPolicy` is `ClusterFirst` (default) unless specific override is needed.
* Check `/etc/resolv.conf` inside the Pod for correct CoreDNS IP.

---

### 10. `Deployment Rollout Stuck` / `ReplicaSet Not Progressing`

**Description:**
A new deployment fails to roll out completely, leaving some old Pods running or no new Pods ready, leading to inconsistent application versions or unavailability and 5xx errors for affected requests.

**Possible Causes:**
* **Underlying Pod Issues:** Any of the above Pod-level issues (`ImagePullBackOff`, `CrashLoopBackOff`, probe failures) preventing new Pods from starting successfully.
* **Insufficient Resources:** Not enough cluster resources (nodes, IP addresses) to accommodate new Pods during a rolling update.
* **Misconfigured Pod/Container:** Error in the new Pod definition preventing it from becoming ready.
* **PodDisruptionBudget (PDB) Blocking:** A PDB is too restrictive, preventing the desired number of Pods from being evicted.

**Troubleshooting & Solution:**
* Check deployment status: `kubectl rollout status deployment/<deployment-name>`
* Describe deployment: `kubectl describe deployment <deployment-name>` (Look for conditions and events)
* Check associated ReplicaSets: `kubectl get rs -l app=<app-label>`
* Investigate the status and events of newly created Pods under the new ReplicaSet.
* Ensure sufficient cluster resources.
* Review PDBs for the application.

---

### 11. Kubernetes API Server Unavailable

**Description:**
If the Kubernetes API server itself is unhealthy or unreachable, `kubectl` commands will fail, and components like Kubelet, controllers, and schedulers won't be able to function, leading to cascading failures and application issues.

**Possible Causes:**
* **Control Plane Node Issues:** Underlying infrastructure issues on the control plane nodes (CPU, memory, network).
* **etcd Issues:** The `etcd` datastore is unhealthy, overloaded, or unreachable.
* **API Server Configuration:** Misconfiguration of the API server itself.
* **Network Connectivity:** Firewall rules or network problems preventing access to the API server.

**Troubleshooting & Solution:**
* Check control plane node health.
* Check API server Pods (usually in `kube-system` namespace) logs: `kubectl logs -n kube-system <kube-apiserver-pod>`
* Check etcd cluster health (if self-managed).
* Verify network connectivity to API server port (usually 6443).
* Ensure sufficient resources for control plane components.

---

### 12. Ingress Controller/Load Balancer Issues

**Description:**
The Ingress controller (e.g., Nginx Ingress, AWS ALB Ingress Controller) or the external load balancer (e.g., AWS ALB, NLB) is misconfigured or unhealthy, preventing external traffic from reaching the Kubernetes Services.

**Possible Causes:**
* **Ingress Object Misconfiguration:** Incorrect host, path, service name, or port in the Ingress resource.
* **Ingress Controller Crash/Unhealthy:** The Ingress controller Pods are `CrashLoopBackOff` or not ready.
* **Load Balancer Health Check Failures:** The load balancer's health checks are failing for the backend Pods/nodes.
* **Security Group/Network ACL Issues:** Firewall rules blocking traffic.
* **SSL/TLS Misconfiguration:** Issues with certificates or TLS settings.

**Troubleshooting & Solution:**
* Check Ingress object events: `kubectl describe ingress <ingress-name>`
* Check Ingress controller Pods status and logs.
* Verify associated Kubernetes Service and its Endpoints.
* Inspect the external load balancer (e.g., in AWS console, check ALB target groups, listeners, security groups, health checks).
* Test connectivity from outside the cluster.

---

### 13. Horizontal Pod Autoscaler (HPA) Issues

**Description:**
The HPA fails to scale Pods up or down effectively, leading to either resource waste (too many Pods) or, more critically, **performance degradation and 5xx errors** due to insufficient Pods during high traffic.

**Possible Causes:**
* **Metric Server Unhealthy:** The Kubernetes Metric Server (required by HPA) is not running or collecting metrics.
* **Incorrect Metrics:** HPA is configured to use custom metrics that are not being reported correctly.
* **Resource Metrics Unavailable:** Pods do not have `resources.requests` defined, so CPU/memory metrics are not available for HPA.
* **Insufficient Resources:** The cluster cannot provide enough nodes/resources to scale up to the desired number of Pods.
* **PDB Conflicts:** PDBs preventing scale down.

**Troubleshooting & Solution:**
* Check HPA status: `kubectl get hpa <hpa-name>`
* Describe HPA: `kubectl describe hpa <hpa-name>` (Look at events, current/desired replicas, and target metrics)
* Check Metric Server Pods status: `kubectl get pods -n kube-system -l k8s-app=metrics-server`
* Ensure `resources.requests` are defined for Pods.
* Verify custom metric pipelines if used.
* Monitor cluster resource utilization.

---

## General Troubleshooting Steps

* **Start with `kubectl get events`**: This often provides the quickest insight into recent cluster activity.
* **`kubectl describe <resource-type>/<name>`**: Provides detailed status, conditions, and events for specific resources (Pods, Deployments, Services, Nodes).
* **`kubectl logs <pod-name> -c <container-name>`**: Essential for application-level debugging.
* **Monitoring Tools**: Utilize Prometheus, Grafana, Datadog, ELK stack (Elasticsearch, Logstash, Kibana) for historical data, trends, and alerts.
* **Check Node Health**: Don't forget to check the underlying nodes (CPU, memory, disk, network) if Pod issues are widespread.
* **Validate Configuration**: Double-check all Kubernetes YAML definitions (Deployment, Service, Ingress, ConfigMaps, Secrets, Network Policies).
* **Isolate the Problem**: Try to narrow down if the issue is application-specific, Kubernetes-level, or infrastructure-level.
* **Rollback**: If a recent deployment or configuration change caused the issue, consider rolling back to a known good state.

---

## Contributing

Feel free to contribute to this repository by adding more common 500-level issues and their solutions/troubleshooting steps.

---

## License

This project is licensed under the MIT License - see the `LICENSE` file for details.
