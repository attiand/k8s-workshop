# General 

* https://aws.plainenglish.io/kubectl-get-kubernetes-o-architecture-6d4bd97dcaaf
* https://iximiuz.com/en/tags/?tag=nerdctl

# Lab

[lab](lab.md)


# Pod

Smallest and most basic deployable unit. Contains one or more containers. Typically one main container and auxiliary init containers or Sidecars.

* All containers within a single Pod share the exact same IP address and port space. Because they share a network namespace, they can communicate with each other using localhost.

* Shared Storage: A Pod can specify a set of shared storage Volumes. All containers in that Pod can mount and access these volumes, allowing them to share data seamlessly.

* Ephemeral Lifecycle: Pods are mortal and disposable. If a Pod crashes, dies, or its host node fails, Kubernetes does not "fix" it. Instead, it throws the dead Pod away and spins up a new one to replace it.

## Init container

* Run to completion

* Run sequentially

* Block app startup. The main containers in the Pod will not start until all init containers have finished successfully.

* Failure can restart the Pod depending on configuration.

### Common init container use cases

* Waiting for dependencies

* Running database migrations

* Populating shared volumes

* Security and Permissions: An init container can run with higher privileges (like root) to change file permissions on a volume, allowing the main application container to run much more securely as a non-root user.

## Sidecar

Common sidecar use cases: Log Forwarding, Service Mesh Proxies, Monitoring & Metrics and Configuration Syncing.

# Nodes

A Node is a physical or virtual machine that provides the actual compute capacity in a Kubernetes cluster.
This is where all containers/pods actually run. Each node is managed by the Control Plane and runs the components required to operate containers:
kubelet (the node agent), a container runtime (e.g. containerd or CRI-O) and kube-proxy (network management).

List all nodes and their status:

```bash
kubectl get nodes
```

Describe a node, useful for checking potential issues on the node:

```bash
kubectl describe node <node-name>
```

If you want a quick overview of CPU and memory usage, there is also top:

```bash
kubectl top nodes
```

# Network

## Pod network

In Kubernetes, every pod lives in its own isolated network namespace and is assigned a unique internal IP
address from a dedicated subnet (the Pod CIDR that is configured when installing k3s).

Basic rules for the pod network:

* All pods can communicate with all other pods on all nodes without NAT (Network Address Translation).

* The agent on a node (e.g. kubelet) can communicate with all pods on the same node.

* A pod's own IP is the same as the one others see it as (no port mapping is required between pods).

How it works in practice:

* Implemented by a CNI plugin (Container Network Interface, e.g. Cilium, Calico or Flannel).

* CNI sets up virtual network interfaces (veth pairs) that connect the pod's network namespace with the node's network.

# See assigned pod IPs and which node they were scheduled on

```bash
kubectl get pods -o wide
```

# Test communication directly between two pods via IP

```bash
kubectl exec -it <pod-a> -- ping <pod-b-ip>
```

## Host network

When a pod is configured with hostNetwork: true it shares the network namespace directly with the underlying node instead of getting its own isolated namespace.
Typical use cases:

Low-level infrastructure components and CNI agents that must configure the network on the node itself.

Ingress controllers or load balancers that need direct access to the node's external IP/ports without kube-proxy overhead.

DaemonSets for network monitoring or telemetry.

# Namespaces

A Namespace is a way to divide and isolate resources within a single Kubernetes cluster.

It acts as a virtual cluster inside the cluster to separate environments, teams or projects.

What namespaces provide

* Namespace isolation: Different namespaces can have resources with identical names (e.g. a service named web in both dev and prod).

* Network addressing: Affects DNS – internal calls between services in the same namespace only require the short name (backend), while calls across boundaries require backend.namespace

* Access control (RBAC): Permissions can be limited to a specific namespace so that a team can only administer its own resources.

* Resource governance: The ability to set limits for CPU, memory and number of objects via ResourceQuota and LimitRange.

# List all existing namespaces

```bash
kubectl get namespaces

kubectl get ns
```

# Create a new namespace

```bash
kubectl create namespace workshop-demo
``` 

# Run a command against a specific namespace

```bash
kubectl get pods -n workshop-demo
```

# Ingress

Ingress is Kubernetes' built-in way to expose HTTP and HTTPS services to the outside world through a common entry point at the application layer (Layer 7).

Instead of each microservice requiring its own external load balancer or public port, the Ingress Controller acts as a central reverse proxy and router inside the cluster.

In many on-prem or bare-metal environments, external traffic is not let straight onto the pods. Instead, an external load balancer is connected to the Ingress Controller via NodePort:

Client / User calls a public domain (e.g. app.example.com).

External Load Balancer (LB): Receives the traffic on standard ports (80/443), handles any external failover/health checks and forwards the traffic to the cluster's nodes via an assigned NodePort (e.g. port 30080 / 30443).

Ingress Controller (e.g. Traefik): Listens on the specific NodePort service on all nodes in the cluster and receives the traffic.

Ingress resources (the rules): The controller matches the incoming HTTP header (Host, path, TLS) against defined Ingress rules and routes the traffic directly to the correct internal Service/Pod IP.

# Load balancer

Understanding how external traffic reaches our services in Kubernetes is about following the user's request all the way in to the correct application pod. Our architecture is built on three central layers: DNS/VIP, NodePorts and the Traefik Ingress Controller.

The traffic flow step by step

* External Load Balancer (VIP)

The load balancer receives the request and balances the traffic across all active worker nodes.

Continuous health checks ensure that traffic is immediately steered away from nodes that are unavailable, in maintenance mode (drain) or not responding.

* DNS & Wildcard

All calls to the cluster (e.g. *.k8s.company.com) point to our external load balancer's virtual IP address (VIP).
Advantage: New services and microservices need no separate DNS configuration; as soon as an ingress rule is created in the cluster the domain is matched automatically.

* NodePorts as entry ports

Traefik is exposed internally in the cluster via a Kubernetes Service of type NodePort.

A NodePort opens the same port range on all worker nodes in the cluster:

Corresponding to port 80 & 443: Web traffic (HTTP and HTTPS).

Dedicated ports: Non-HTTP traffic (e.g. SSH, activemq-artemis and similar).

It does not matter which specific node the request hits; the cluster's internal network layer (kube-proxy) routes the traffic to an active Traefik pod.

* Traefik (Ingress Controller)

Traefik receives the request from the NodePort and decides where it should go:

HTTP/HTTPS (Layer 7): Reads the Host header (e.g. app.k8s.company.com), matches path rules, handles TLS termination and applies any middlewares (headers, authentication, rate limiting).

SSH & TCP/UDP (Layer 4): Matches either on port or via SNI and passes the traffic transparently to the correct backend.

The traffic is finally sent directly to the target pod's internal IP.

# Resources

## Service

A Service is a network abstraction in front of a dynamic set of pods.

Because pods are ephemeral and get new IP addresses when they are restarted or scaled, a Service provides a fixed internal IP and a stable DNS name.

Kubernetes' built-in DNS server (e.g. CoreDNS) automatically creates DNS records for each Service. This minimizes hardcoded network configuration between microservices.

A complete FQDN always has the format:

`<service-name>.<namespace>.svc.cluster.local`

How calls are simplified within the cluster:

Same namespace: A pod in the same namespace can call just the service name


```bash
curl http://backend:8080
```

Other namespace: A pod in another namespace specifies the service name and namespace

```bash
curl http://backend.prod:8080
```

Fully qualified (FQDN): Used for explicit needs or to avoid DNS search-domain lookups

```bash
curl http://backend.prod.svc.cluster.local:8080
```

Just like pods have their own network (Pod CIDR), Services are assigned virtual IP addresses from their own dedicated subnet called the Service CIDR.

The default in k3s is `10.43.0.0/16` (configured via --service-cidr at cluster installation).

The Service IP (ClusterIP) is not bound to any physical or virtual network interface on the nodes.

It is kube-proxy (or CNI via eBPF/iptables) that intercepts traffic addressed to the Service CIDR and load balances it directly onward to the correct underlying Pod IP.

Common Service types

* ClusterIP: (Default) Gets an internal IP from the Service CIDR. Only reachable from inside the cluster.

* NodePort: Opens a static port (default 30000–32767) on all nodes. Forwards the traffic to an underlying ClusterIP.

* LoadBalancer: Builds on NodePort but requests an external load balancer from the underlying cloud/infrastructure.

* Headless Service (clusterIP: None): Is assigned no virtual IP at all. DNS calls instead return A records directly to the matching pods' IP addresses.

# List services in a namespace (shows ClusterIP, ports and type)

```bash
kubectl get svc -n backend
```

# See which actual pod IPs the service points to right now

```bash
kubectl get endpoints api-service -n backend
```

## CM (Config Maps)

A ConfigMap is an API object used to store non-sensitive data in the form of key-value pairs.

Its main purpose is to separate environment-specific configuration from images.

A Pod can read CMs in 3 different ways

* Environment variables: Map the keys to environment variables

* Mounted volumes: Mount your ConfigMap as a file or directory inside the container. This is perfect for passing in entire configuration files.

* Command-Line Arguments: Pass the values into the container's start command.

## Secrets

A Secret is a special CM for storing and managing sensitive data – such as passwords, tokens, API keys and SSH/TLS certificates 

– without hardcoding them in container images or pod specs.

We use `external-secret-operator` which lets us store the secrets in Hashicorp Vault and then map them to kubernetes secrets.

There are two common ways to give a container access to a Secret:

As environment variables:

Good for simple passwords or configuration strings.

Mounted as files (Volumes):

Standard for certificates, keys or larger secrets. Kubernetes mounts them as a tmpfs (RAM-based volume in memory) so that they are never written to the node's physical disk.

## Labels

Labels are strictly for identifying and selecting objects. Because they are used for querying, they have strict syntax rules (e.g., keys and values have limited length).

```bash
kubectl get pods  --show-labels
```

List pods with a specific label:

```bash
kubectl get pods -l sub-system=mind
```

## Annotations

Annotations are used to store non-identifying metadata (like a build ID, a commit hash, or configuration settings for a third-party tool). They can hold much larger amounts of data, but you cannot use a selector to filter objects based on an annotation.

```bash
kubectl describe pod my-pod
```

# Helm

The official package manager for Kubernetes, Helm utilizes the Go template language. Helm charts are usually provided directly by the software vendors.

## Create

```bash
mkdir mychart
helm create mychart

# dry run
helm template mychart
```

## Example

https://github.com/helm/examples

```bash
helm repo add examples https://helm.github.io/examples

# dry run
helm template ahoy examples/hello-world

helm install ahoy examples/hello-world

helm uninstall ahoy
```

# Kustomize

Kustomize is a tool built natively into Kubernetes that lets you customize YAML files for different environments (like dev, stage, and prod) without modifying the original manifests.

Instead of using complex templates with variables (like Helm), Kustomize relies on a base and overlay architecture to merge configurations together.

* Built into `kubectl`
* Built-in Generators: Kustomize can automatically generate ConfigMaps and Secrets from external files or literal values
* Transformers, Patches manifests
* Can generate from Helm charts `--enable-helm`

```
./base/kustomization.yaml
./base/deployment.yaml
./base/service.yaml
./envs/lab/kustomization.yaml
./envs/stage/kustomization.yaml
```

## Dry run

```bash
kubectl kustomize --enable-helm .
```

## Execute

```bash
kubectl apply -k --enable-helm .
```

# Storage

Kubernetes manages storage by decoupling the application's need for disk from the underlying storage infrastructure.

This is done via two central building blocks: PersistentVolume (PV) and PersistentVolumeClaim (PVC).

PV vs PVC

PersistentVolume (PV):

The actual storage resource (e.g. local disk, NFS, iSCSI or a SAN block).

*   Is cluster-wide (does not belong to a namespace).

*   Created either statically by a cluster administrator or dynamically via a StorageClass.

PersistentVolumeClaim (PVC):

*   A request from a user/pod.

*   Is bound to a specific namespace.

*   Specifies needs: size (e.g. 10Gi), access mode and any StorageClass.

When a PVC is created, Kubernetes looks for a matching PV and binds them together (status: Bound) in a 1:1 relationship.

Access Modes - Specify how the volume may be mounted by nodes

*   RWO - ReadWriteOnce mounted for reading and writing by a single node at a time (common for block storage/local disk)

*   ROX - ReadOnlyMany mounted as read-only by many nodes simultaneously

*   RWX - ReadWriteMany mounted for reading and writing by multiple nodes simultaneously (requires a filesystem such as NFS).


## Longhorn

Longhorn is a distributed, open-source block storage solution designed directly for Kubernetes.

It turns local storage on cluster nodes into a fault-tolerant, replicated and distributed storage network.

How Longhorn works in practice

* Synchronous replication: Each volume is split into a defined number of copies (replicas, often 3 by default) that are spread across different nodes in the cluster.

* Microservices per volume: Longhorn runs a dedicated controller and volume engine per active volume via containers on the nodes.

* iSCSI underneath: Pods connect to their volumes via the node's local iSCSI interface created by Longhorn's CSI driver.

* If a node with a running pod dies, Kubernetes can reschedule the pod to another node, and Longhorn immediately connects to one of the existing replicas there.

Central features

* Built-in StorageClass: Automatically registers longhorn as a storage class, which makes it easy to dynamically provision PVs via ordinary PVCs.

* Snapshots and Backups: Built-in support for scheduled snapshots locally as well as asynchronous backups to external S3-compatible storage or NFS.

* Web UI

* ReadWriteMany (RWX) support: Can, via an integrated NFS server, offer volumes shared between multiple nodes simultaneously.

# Operators

An Operator is a specialized, domain-specific Controller.

## Controller
A Controller is a native, general-purpose Kubernetes automation loop. It manages standard, built-in Kubernetes resources

Example: [CNPG](https://cloudnative-pg.io/)

## Custom resources (CRD)

Allows you to extend the Kubernetes API with your own custom objects, allowing Kubernetes to manage them exactly as it manages native, built-in resources like Pods, Deployments, or Services.

Example: [CNPG Cluster](cnpg/simple-cluster.yaml)

```bash
kubectl get crds

kubectl get clusters.postgresql.cnpg.io
```

# ArgoCD

Git-ops, The Git repository is the single source of truth.

Operator loop:
 1. Check specified repositories for new commits.
 2. Detecting drift.
 3. Automatic (or manual) synchronization.

## Key Benefits

Automated Deployments: No need to write deployment scripts. Argo CD handles the deployment automatically.

Instant Rollbacks: Rolling back is as simple as reverting the commit in Git.

Better Security: Argo CD pulls configurations from Git, you don't need to give your CI pipeline direct administrative access to your Kubernetes cluster.

Disaster Recovery: If your Kubernetes cluster goes down or needs to be rebuilt, you don't lose any configuration. Spin up a new cluster, point Argo CD to your Git repository, and it will recreate the entire environment exactly as it was.

Visibility: Argo CD provides a web UI that visualizes all your running application resources.

## Application

* Source
  * Git branch + path (looks for file `Chart.yaml` or `kustomization.yaml`)
  * ~~Helm repository URL~~
* Destination (cluster to sync)
* Sync Policy (automatic or manual)

## ApplicationSet

Operator that creates ArgoCD application from a directory or file structure.

# Logs

## Loki

* Fluent bit
* S3 storage

# Kyverno

[Kyverno](kyverno.md)


## Debug Container

[Debug Container](debug.md)

# Orange

## k3s

### Rancher

Set up access, fetch k8s config.

## Kyverno

* validate-limits policy

## Grafana

## S3

# Non K8S

## S3

Amazon S3 (Simple Storage Service) is a highly scalable, cloud-based object storage service.

Object Storage: It saves files as "objects" inside containers called "buckets" rather than a traditional folder hierarchy.

Key: The unique name of the object. This is how you identify and retrieve the data, ex `images/2026/summer/vacation.jpg`

Value: The actual raw data you are storing (the bytes of the photo, video, or document)

Metadata: A set of name-value pairs that describe the object

## DHI (Docker hardened images)

https://hub.docker.com/hardened-images/catalog

DHI Community (Free) and DHI Select & Enterprise (Paid)

* Stripped images down to minimize their security attack surface
* Signed by Docker
* Non-Root Execution
* Stripped Permissions

Special registry name `dhi.nya-srv.its.umu.se`
