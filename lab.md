# Lab

https://medium.com/@muppedaanvesh/deploying-nginx-on-kubernetes-a-quick-guide-04d533414967

## Create Kind cluster

https://kind.sigs.k8s.io

```bash
kind create cluster --name mycluster
kind get kubeconfig --name mycluster > ~/.kube/mycluster.yaml
```

## Config

### Simple

```bash
export KUBECONFIG=~/.kube/mycluster.yaml
```

### Kubie

https://github.com/kubie-org/kubie


## Run pod

```bash
kubectl run nginx-pod --image=nginx --restart=Never --port=80 -n default

kubectl get pods

kubectl logs nginx-pod
```

### Port Forward

```bash
kubectl port-forward pod/nginx-pod 8080:80

curl localhost:8080
```

### Delete pod

```bash
kubectl delete pod nginx-pod
```

## Create Deployment & Service

* Skapa filen [deploment.yaml](lab/1/deployment.yaml)
* Skapa filen [serice.yaml](lab/1/service.yaml)

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

### Port Forward

```bash
kubectl port-forward svc/nginx-svc 8080:80

curl localhost:8080
```

### Commands

```bash
kubectl get pods

kubectl get pods -o wide

kubectl get pod nginx-deployment-756d4cb589-qftt6

kubectl get pod -o yaml nginx-deployment-756d4cb589-qftt6

kubectl describe pod nginx-deployment-756d4cb589-qftt6

kubectl describe svc nginx-svc

kubectl logs nginx-deployment-756d4cb589-qftt6

kubectl exec -it nginx-deployment-756d4cb589-qftt6 -- sh
```

## Replicas

Ändra replicas: 2 -> 1 i `deployment.yaml`

```bash
kubectl apply -f deployment.yaml
```

## Delete pod

```bash
kubectl delete pod nginx-deployment-756d4cb589-k6867
```

The pod is recreated.

## Delete deployment & Service

```bash
kubectl delete svc nginx-svc
kubectl delete deploment nginx-deployment
```
or:
```bash
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml
```

## Delete Kind Cluster

```bash
kind delete cluster --name mycluster

rm ~/.kube/mycluster.yaml
```
