# Debug container

General tips: https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod

## Kind setup

```bash
kubectl create secret docker-registry my-registry-secret \
  --docker-server=dhi.nya-srv.its.umu.se \
  --docker-username=nya \
  --docker-password='' \
  --docker-email=nya-cm@its.umu.se
```

## Start a tiny container

```bash
kubectl run nginx-pod --image=dhi.nya-srv.its.umu.se/nginx:1-debian --restart=Never --port=80 -n default
```

Try to execute `sh`

```bash
kubectl exec -it nginx-pod -- sh
```

## kubectrl debug

### 1 - Add ephemeral container

Adds a temporary debug container to a running pod. The pod keeps running. The ephemeral container shares the pod’s namespaces. Once added, it cannot be removed.

* The ephemeral container stays attached (possible to re-attach with kubectl attach)
* Require restart of the pod to remove the attached container

```bash
# check container (target)
kubectl describe pod nginx-pod

kubectl debug -it nginx-pod --image=dhi.nya-srv.its.umu.se/nginx:1-debian-dev --target=nginx-pod -- sh

kubectl debug -it nginx-pod --image=nicolaka/netshoot --target=nginx-pod -- sh

# Inside container

ps

curl

ls /proc/1/root/

```

#### 2 - copy-to

Creates a duplicate of the pod with modifications, a new debug container, swapped image, or overridden entrypoint.

Gets no traffic. Original pod is untouched. Delete the copy when you’re done.

Debug for CrashLoopBackOff Pods, change command

```bash
kubectl debug -it nginx-pod --copy-to=my-debugger --container=nginx-pod -- ls
```

##### Change image

Container running same environment, network, and node as `nginx-pod`.

Read env var, read CM, secrets etc.

```bash
kubectl debug -it nginx-pod --copy-to=my-debugger --image=nicolaka/netshoot --container=nginx-pod -- sh
```

#### 3 - Node

Starts a privileged pod on a specific node with full access to the host’s namespaces. The host filesystem is mounted at `/host`.

```bash
kubectl debug node/mycluster-control-plane -it --image=nicolaka/netshoot


# Inside container

ps

ls /host

```


