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

Ett Namespace är ett sätt att dela upp och isolera resurser inom ett och samma Kubernetes-kluster. 

Det fungerar som ett virtuellt kluster inuti klustret för att separera miljöer, team eller projekt.

Vad namespaces ger

    Namnrymdsisolering: Olika namespaces kan ha resurser med identiska namn (t.ex. en service som heter web i både dev och prod).

    Nätverksadressering: Påverkar DNS – interna anrop mellan tjänster i samma namespace kräver bara kortnamnet (backend), medan anrop över gränserna kräver backend.namespace

    Åtkomstkontroll (RBAC): Rättigheter kan begränsas till ett specifikt namespace så att ett team bara kan administrera sina egna resurser.

    Resursstyrning: Möjlighet att sätta gränser för CPU, minne och antal objekt via ResourceQuota och LimitRange.

# Lista alla befintliga namespaces
    kubectl get namespaces

    kubectl get ns

# Skapa ett nytt namespace
    kubectl create namespace workshop-demo

# Kör ett kommando mot ett specifikt namespace
    kubectl get pods -n workshop-demo

# Ingress

Ingress är Kubernetes inbyggda sätt att exponera HTTP- och HTTPS-tjänster mot omvärlden via en gemensam ingång på applikationslagret (Layer 7).

Istället för att varje mikrotjänst kräver en egen extern lastbalanserare eller publik port, fungerar Ingress Controllern som en central reverse proxy och router inuti klustret.

I många on-prem- eller bare-metal-miljöer släpper man inte in extern trafik rakt på poddarna. Istället kopplas en extern lastbalanserare ihop med Ingress Controllern via NodePort:

Klient / Användare anropar publik domän (t.ex. app.exempel.se).

Extern Lastbalanserare (LB): Tar emot trafiken på standardportar (80/443), hanterar eventuell extern failover/hälsa och skickar trafiken vidare till klustrets noder via en tilldelad NodePort (t.ex. port 30080 / 30443).

Ingress Controller (t.ex. Traefik ): Lyssnar på den specifika NodePort-tjänsten på alla noder i klustret och tar emot trafiken.

Ingress-resurser (Reglerna): Controllern matchar inkommande HTTP-header (Host, sökväg, TLS) mot definierade Ingress-regler och routar trafiken direkt till rätt intern Service/Pod IP.

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

En Service är en nätverksabstraktion framför en dynamisk uppsättning poddar. 

Eftersom poddar är förgängliga och får nya IP-adresser när de startas om eller skalas, ger en Service en fast intern IP och ett stabilt DNS-namn.

Kubernetes inbyggda DNS-server (t.ex. CoreDNS) skapar automatiskt DNS-poster för varje Service. Detta minimerar hårdkodad nätverkskonfiguration mellan mikrotjänster.

Ett komplett FQDN har alltid formatet:

<service-name>.<namespace>.svc.cluster.local

Hur anrop förenklas inom klustret:

Samma namespace: En pod i samma namespace kan anropa enbart tjänstens namn

    curl http://backend:8080

Annat namespace: En pod i ett annat namespace anger servicenamn och namespace

    curl http://backend.prod:8080

Fullständigt (FQDN): Används vid explicita behov eller för att undvika DNS-sökdomän-uppslag

    curl http://backend.prod.svc.cluster.local:8080

Precis som poddar har sitt eget nät (Pod CIDR), tilldelas Services virtuella IP-adresser ur ett eget dedikerat subnät som kallas Service CIDR.

Standard i k3s är 10.43.0.0/16 (konfigureras via --service-cidr vid klusterinstallation).

Service-IP (ClusterIP) är inte bunden till något fysiskt eller virtuellt nätverkskort på noderna.

Det är kube-proxy (eller CNI via eBPF/iptables) som fångar upp trafik adresserad till Service CIDR och lastbalanserar den direkt vidare till rätt underliggande Pod-IP.

Vanliga Service-Typer

    ClusterIP: (Standard) Får en intern IP ur Service CIDR. Endast nåbar inifrån klustret.

    NodePort: Öppnar en statisk port (standard 30000–32767) på alla noder. Vidarebefordrar trafiken till en underliggande ClusterIP.

    LoadBalancer: Bygger på NodePort men begär en extern lastbalanserare från underliggande moln/infrastruktur.

    Headless Service (clusterIP: None): Tilldelas ingen virtuell IP alls. DNS-anrop returnerar istället A-records direkt till de matchande poddarnas IP-adresser.

# Lista services i ett namespace (visar ClusterIP, portar och typ)
    kubectl get svc -n backend

# Se vilka faktiska pod-IPs som servicen pekar ut just nu
    kubectl get endpoints api-service -n backend


## CM (Config Maps)

## Secrets

En Secret är ett Kubernetes-objekt avsett för att lagra och hantera känslig data – såsom lösenord, tokens, API-nycklar och SSH/TLS-certifikat 

– utan att hårdkoda dem i container-images eller pod-specar.

Vi använder external-secret-operator som gör att vi kan lagra hemligheterna i Hashicorp Vault och sedan mappa detta till kubernetes secrets.

Det finns två vanliga sätt att ge en container tillgång till en Secret:

Som miljövariabler (Environment Variables):

Bra för enkla lösenord eller konfigurationssträngar.

Monterade som filer (Volumes):

Standard för certifikat, nycklar eller större hemligheter. Kubernetes monterar dem som en tmpfs (RAM-baserad volym i minnet) så att de aldrig skrivs till nodens fysiska disk.

## Jobs

## Labels

Labels are strictly for identifying and selecting objects. Because they are used for querying, they have strict syntax rules (e.g., keys and values are limited to 63 characters).

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

# Storage

* PVC
* PV

Retention

## Longhorn

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
 3. Automatic (or manual) synchronisation.

## Key Benefits

Automated Deployments: No need to write deployment scripts. Argo CD handles the deployment automatically.

Instant Rollbacks: Rolling back is as simple as reverting the commit in Git.

Better Security: Argo CD pulls configurations from Git, you don't need to give your CI pipeline direct administrative access to your Kubernetes cluster.

Disaster Recovery: If your Kubernetes cluster goes down or needs to be rebuilt, you don't lose any configuration. Spin up a new cluster, point Argo CD to your Git repository, and it will recreate the entire environment exactly as it was.

Visibility: Argo CD provides a web UI that visualizes all your running application resources.

## Application

* Source
 * ~~Helm repository URL~~
 * Git branch + path (looks for a `Chart.yaml` or `kustomization.yaml`)
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
Fixa access, hämta k8s config.

## Kyverno

## Grafana

## S3

# Non K8S

## S3

## DHI (Docker hardened images)

# AI support

## K8sGPT

Ett LLM verktyg som kan analysera kluster automatiskt och spotta ur sig en lista över fel som den tycker sig hitta.

(visa exempel)
