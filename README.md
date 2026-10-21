# Day 17: Kubernetes Services & Networking (Docker Desktop / Kind Local Cluster)

## Objective
Give an application a stable in-cluster network address, resolve it by DNS name from another Pod, observe load balancing across active replicas, monitor EndpointSlices, and diagnose why a Service stops routing traffic when its selector is misconfigured.

## Architecture & Key Concepts
- **ClusterIP Service:** The default virtual IP exposing a Service internally within the cluster, giving Pods a stable address even as Pod IPs change.
- **Service Selector:** A label query (`app: hello`) used by a Service to dynamically match and route traffic to healthy backend Pods.
- **EndpointSlice:** A scalable Kubernetes API object that tracks backend IP addresses (`10.1.0.x`) and target ports associated with a Service.
- **CoreDNS Name Resolution:** Resolves internal Service names (e.g., `hello` or `hello.default.svc.cluster.local`) directly to the Service ClusterIP.
- **`port` vs `targetPort`:** `port` is the external-facing port exposed by the Service (e.g., `80`); `targetPort` is the application port container listens on (e.g., `8080`).
- **Load Balancing:** Kube-proxy distributes incoming traffic across all Ready backend Pod endpoints attached to the Service.

## Cluster Setup & Baseline Verification
- Verified active cluster context (`docker-desktop`) and running control plane nodes using `kubectl config current-context` and `kubectl get nodes`.
- Inspected baseline deployments and services using `kubectl get deployments` and `kubectl get services`.

## Workload Deployment & ClusterIP Exposure
- Created a 3-replica test deployment using `agnhost` on port `8080` with label `app=hello`.
- Applied `svc-clusterip.yaml` to create a `ClusterIP` Service exposing port `80` mapped to `targetPort: 8080`.
- Verified Service creation, assigned ClusterIP (`10.111.166.197`), and active EndpointSlices using `kubectl get svc hello -o wide` and `kubectl get endpointslices`.

## CoreDNS Name Resolution & Load Balancing Verification
- Launched an interactive temporary client Pod (`busybox:1.36`).
- Verified internal DNS resolution for both short name (`hello`) and fully qualified domain name (`hello.default.svc.cluster.local`).
- Executed a multi-request `wget` loop (`http://hello/hostname`) to confirm HTTP traffic was distributed across all 3 backend Pod replicas.

## Endpoint Monitoring & Dynamic Self-Healing
- Cross-referenced backend IP addresses in EndpointSlices (`10.1.0.25`, `10.1.0.26`, `10.1.0.27`) directly with Pod IPs.
- Deleted a backend Pod (`hello-756774f8c5-6q8sg`) to test dynamic endpoint updating.
- Confirmed that Kubernetes terminated the old Pod, removed its IP from EndpointSlices, launched a replacement Pod (`hello-756774f8c5-w4n5z`), and automatically added the new Pod IP (`10.1.0.33`) back to the active endpoints.

## Selector Failure Induction & Diagnostics
- Deliberately corrupted the Service selector to an invalid label (`app: wrong-app`) using `kubectl patch service`.
- Verified that while the Service object and ClusterIP remained active, all backend endpoints disappeared (`<unset>`).
- Proved that DNS still resolved the Service ClusterIP, but application requests failed due to missing backend Pod endpoints.

## Selector Restoration & Self-Healing Fix
- Patched the Service selector back to its valid label (`app: hello`).
- Verified immediate restoration of all 3 backend endpoints in EndpointSlices and `kubectl describe svc hello`.
- Re-executed the client request loop to confirm successful HTTP traffic routing and backend load balancing.

## Repository Structure
```text
day17-services/
├── README.md
├── svc-clusterip.yaml
├── architecture.png
└── evidence/
    ├── service-endpoints.png
    ├── dns-load-balancing.png
    ├── pod-replacement.png
    └── selector-failure-fix.png
    
```

## Cleanup

kubectl delete service hello

kubectl delete deployment hello
