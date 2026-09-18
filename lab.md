# Lab

https://medium.com/@muppedaanvesh/deploying-nginx-on-kubernetes-a-quick-guide-04d533414967

## Setup Kind

https://kind.sigs.k8s.io

```bash
kind create cluster --name mycluster
kind get kubeconfig --name mycluster > ~/.kube/mycluster.yaml
```

## Kubi

https://github.com/kubie-org/kubie

Alternativ:

```bash
export KUBECONFIG=~/.kube/mycluster.yaml
```

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
kc delete pod nginx-pod
```

## Create deployment & Service

* Skapa fileen [deploment.yaml](lab/1/deployment.yaml)
* Skapa fileen [serice.yaml](lab/1/service.yaml)

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

* get (get -o yaml)
* describe
* logs
* delete
* apply
* top
* exec

## Replicas

Ändra replicas: 2 -> 1 i `deployment.yaml`

```bash
kubectl apply -f deployment.yaml
```

## Delete deployment & Service

```bash
kubectl delete svc nginx-svc
```
eller:
```bash
kubectl delete -f service.yaml
```

```bash
kubectl delete -f deployment.yaml
```

## Debug Container

