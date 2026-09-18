# General 

https://www.alibabacloud.com/blog/getting-started-with-kubernetes-%7C-kubernetes-container-runtime-interface_596339 

# Lab

[lab](lab.md)

# Nodes

# Network

## Pod network

I Kubernetes lever varje pod i ett eget isolerat nätverksnamnrymd (network namespace) och tilldelas en unik intern IP-
adress ur ett dedikerat subnät (Pod CIDR).

## Host network

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

* CM (Config Maps)
* Secrets
* Jobs

## Labels

## Annotations

# Helm

# Kustomize

# Operators

vs Controller

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

Fluent bit

# Kyverno

## Resource limits

### Request

### Limits

# Orange

 * k3s
   * Rancher
 * Kyverno
 * Grafana
 * S3

Fixa access

# Non K8S

## S3

## DHI (Docker hardened images)

# AI support

## KubeGPT

## K8sGPT
