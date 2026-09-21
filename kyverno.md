# Kyverno

Policy engine, https://kyverno.io

* Pull images from `docker.nya-srv.its.umu.se` only.
* Verify image signature
* Resource limits

## Available policies

https://kyverno.io/policies/

## Get cluster policys

```bash
kubectl describe cpol -n kyverno
```

## Get all reports

```bash
kubectl get policyreports -A
```

## Resource limits

### Request

Tells the Kubernetes scheduler how much CPU and memory a container must have available.

The scheduler adds up the requests of all containers in a Pod and searches for a Node with enough unallocated capacity. If no Node has enough capacity, the Pod remains in a Pending state.

### Limits

Tells the container runtime the hard maximum amount of CPU and memory the container is allowed to use.
