# General 

https://www.alibabacloud.com/blog/getting-started-with-kubernetes-%7C-kubernetes-container-runtime-interface_596339 

# Lab

[lab](lab.md)

# Nodes

En Node är en fysisk eller virtuell maskin som utgör själva beräkningskapaciteten i ett Kubernetes-kluster. 
Det är här alla containers/poddar faktiskt körs. Varje nod hanteras av kontrollplanet (Control Plane) och kör de nödvändiga komponenterna för att driva containers: 
kubelet (nodagenten), en container runtime (t.ex. containerd eller CRI-O) samt kube-proxy (nätverkshantering).

Lista alla noder och deras status

    kubectl get nodes

Beskriv en nod, bra för att kolla ev fel på noden

    kubectl describe node <nodnamn>

Om man vill se en snapp överblick över CPU och Minnesanvändning så finns också top

    kubectl top nodes

# Network

## Pod network

I Kubernetes lever varje pod i ett eget isolerat nätverksnamnrymd (network namespace) och tilldelas en unik intern IP-
adress ur ett dedikerat subnät (Pod CIDR som ställs in vid installation av k3s).

Grundregler för pod-nätverket:

    Alla poddar kan kommunicera med alla andra poddar på alla noder utan NAT (Network Address Translation).

    Agenten på en nod (t.ex. kubelet) kan kommunicera med alla poddar på samma nod.

    Poddens egen IP är densamma som andra ser den som (ingen portmappning krävs mellan poddar).

Hur det fungerar i praktiken:

    Realiseras av ett CNI-plugin (Container Network Interface, t.ex. Cilium, Calico eller Flannel).

    CNI sätter upp virtuella nätverksgränssnitt (veth-par) som binder samman poddens nätverksnamnrymd med nodens nätverk.

# Se tilldelade pod-IPs och vilken nod de schemalagts på
    
    kubectl get pods -o wide

# Testa kommunikation direkt mellan två poddar via IP

    kubectl exec -it <pod-a> -- ping <pod-b-ip>

## Host network

När en pod konfigureras med hostNetwork: true delar den nätverksnamnrymd direkt med den underliggande noden istället för att få ett eget isolerat namespace.
Typiska användningsområden:

Systemnära infrastrukturkomponenter och CNI-agenter som måste konfigurera nätverket på själva noden.

Ingress controllers eller lastbalanserare som behöver direkt tillgång till nodens externa IP/portar utan kube-proxy-overhead.

DaemonSets för nätverksövervakning eller telemetri.

# Namespaces

# Ingress

# Load balancer

Att förstå hur extern trafik når våra tjänster i Kubernetes handlar om att följa användarens anrop hela vägen in till rätt applikations-pod. Vår arkitektur bygger på tre centrala lager: DNS/VIP, NodePorts och Traefik Ingress Controller.

Trafikflödet steg för steg

* Extern Lastbalanserare (VIP)

Lastbalanseraren tar emot anropet och balanserar trafiken över alla aktiva workernoder.

Kontinuerliga hälsokontroller säkerställer att trafik omedelbart styrs bort från noder som är otillgängliga, i underhållsläge (drain) eller inte svarar.

* DNS & Wildcard

Alla anrop mot klustret (t.ex. *.k8s.foretag.se) pekar mot vår externa lastbalanserares virtuella IP-adress (VIP).
Fördel: Nya tjänster och mikrotjänster behöver ingen separat DNS-konfiguration; så fort en ingress-regel skapas i klustret matchas domänen automatiskt.

* NodePorts som ingångsportar

Traefik exponeras internt i klustret via en Kubernetes Service av typen NodePort.

En NodePort öppnar samma portintervall på samtliga workernoder i klustret:

Motsvarande port 80 & 443: Webbtrafik (HTTP och HTTPS).

Dedikerade portar: Icke-HTTP-trafik (t.ex. SSH, activemq-artemis och liknande).

Det spelar ingen roll vilken specifik nod förfrågan träffar; klustrets interna nätverkslager (kube-proxy) leder trafiken till en aktiv Traefik-pod.

* Traefik (Ingress Controller)

Traefik tar emot anropet från NodePorten och avgör vart det ska:

HTTP/HTTPS (Lager 7): Läser av Host-headern (t.ex. app.k8s.foretag.se), matchar Path-regler, sköter TLS-terminering och lägger på eventuella middlewares (headers, autentisering, rate limiting).

SSH & TCP/UDP (Lager 4): Matchar antingen på port eller via SNI och slussar trafiken transparent till rätt backend.

Trafiken skickas slutligen direkt till mål-poddens interna IP.

# Resources

## Service

Fully qualified domain name. Reduces configuration.

```<namespace>.svc.cluster.local```

## CM (Config Maps)

## Secrets

## Jobs

## Labels

## Annotations

# Helm

Helm is the official package manager for Kubernetes. Go template language.

## Dry run

```bash
helm template .
```

# Kustomize

## Dry run

```bash
kubectl kustomize --enable-helm .
```

# Operators

An Operator is a specialized, domain-specific Controller.

## Controller
A Controller is a native, general-purpose Kubernetes automation loop. It manages standard, built-in Kubernetes resources


Example CNPG

## Custom resources

# Storage

* PVC
* PV

### Longhorn

# ArgoCD

Git-ops

## Application

## ApplicationSet

# Vault

## External secrets

Retention

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
Fixa access, hämta k8s config.

## Kyverno

## Grafana

## S3

# Non K8S

## S3

## DHI (Docker hardened images)

# AI support

## KubeGPT

## K8sGPT
