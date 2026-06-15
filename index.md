# Concepts

All Kubernetes fundamentals, ordered from basics to production-ready knowledge.

| # | Topic | Description |
|---|-------|-------------|
| 01 | [Why Kubernetes](01-why-kubernetes.md) | Why K8s exists, desired state, K8s vs Compose vs Swarm |
| 02 | [Kubernetes Architecture](02-kubernetes-architecture.md) | Control plane, worker nodes, API server, etcd, kubelet |
| 03 | [kubectl & YAML](03-kubectl-and-yaml.md) | kubectl commands, kubeconfig, YAML syntax, declarative management |
| 04 | [Pods](04-pods.md) | Container wrapper, multi-container patterns, lifecycle |
| 05 | [ReplicaSets](05-replicasets.md) | Self-healing, scaling, selectors |
| 06 | [Deployments](06-deployments.md) | Rolling updates, rollbacks, scaling strategies |
| 07 | [Services](07-services.md) | ClusterIP, NodePort, LoadBalancer, DNS |
| 08 | [Namespaces](08-namespaces.md) | Isolation, resource quotas, multi-tenancy |
| 09 | [ConfigMaps & Secrets](09-configmaps-and-secrets.md) | External config, env vars, volume mounts |
| 10 | [Storage](10-storage.md) | PV, PVC, StorageClasses, persistent data |
| 11 | [Ingress](11-ingress.md) | HTTP routing, TLS, path/host-based routing |
| 12 | [Labels, Selectors & Annotations](12-labels-selectors-annotations.md) | Organizing and filtering resources |
| 13 | [DaemonSets, StatefulSets & Jobs](13-daemonsets-statefulsets-jobs.md) | Specialized workload types |
| 14 | [Health Checks](14-health-checks.md) | Liveness, readiness, startup probes |
| 15 | [Resource Management](15-resource-management.md) | CPU/memory requests, limits, HPA |
| 16 | [RBAC & Security](16-rbac-and-security.md) | Roles, RoleBindings, ServiceAccounts |
| 17 | [Scheduling](17-scheduling.md) | Node affinity, taints, tolerations |
| 18 | [Networking Deep Dive](18-networking-deep-dive.md) | CNI, kube-proxy, CoreDNS, NetworkPolicies |
| 19 | [Helm](19-helm.md) | Charts, values, install/upgrade/rollback |
| 20 | [Monitoring & Logging](20-monitoring-and-logging.md) | Prometheus, Grafana, Loki |
| 21 | [Troubleshooting](21-troubleshooting.md) | Debugging flow, common errors |
| 22 | [Hands-On Lab](22-hands-on-lab.md) | 10 guided labs on a real cluster |
