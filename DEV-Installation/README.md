# Kubernetes Cluster Einrichtung mit Kind, Traefik und Prometheus

Alle Dateien zur Einrichtung liegen unter:  
`E:\K8s-Monitoring\PROD Installation`

Alle Commands werden aus dem Ordner  
`E:\K8s-Monitoring Installation\DEV-Installation` ausgeführt.

---

## 1. Kind Cluster erstellen

```bash
kind create cluster --name prod --image kindest/node:v1.33.1 --config kind.yaml
```

---

## 2. Cert-Manager installieren

```bash
kubectl create namespace cert-manager
helm install cert-manager oci://quay.io/jetstack/charts/cert-manager --version v1.18.2 --namespace cert-manager --set crds.enabled=true
```

---

## 3. Traefik Installation

```bash
kubectl create namespace traefik
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm install traefik traefik/traefik --namespace=traefik --set service.type=NodePort --set service.nodePorts.http=32041 --set service.nodePorts.https=31079
helm upgrade traefik traefik/traefik -n traefik --set watchNamespace=""
```

---

## 4. Portainer Agent Installation

```bash
kubectl apply -f https://downloads.portainer.io/ce2-33/portainer-agent-k8s-nodeport.yaml
```

Der Dev-Cluster dem Prod-Cluster hinzufügen, indem zunächst die IP des Dev-Controllers ermittelt wird.

```bash
kubectl config use-context kind-dev
docker inspect dev-control-plane | Select-String "IPAddress"
```

Auf Portainer Kubernetes-Environtment via NodePort hinzufügen.

```bash
[IP-Adresse]+[Port], beispielsweise 172.18.0.13:30778
```

---

## 6. Beispielapplikationen Images laden und ausführen

```bash
docker compose -f prometheus/docker-compose.yaml build

kind load docker-image docker.io/library/go-application:latest --name prod
kind load docker-image docker.io/library/dotnet-application:latest --name prod
kind load docker-image docker.io/library/python-application:latest --name prod
kind load docker-image docker.io/library/nodejs-application:latest --name prod
```

Namespaces erstellen:

```bash
kubectl create namespace go-app
kubectl create namespace dotnet-app
kubectl create namespace python-app
kubectl create namespace nodejs-app
```

Deployments ausrollen:

```bash
kubectl apply -f prometheus/go-application/deployment.yaml -n go-app
kubectl apply -f prometheus/dotnet-application/deployment.yaml -n dotnet-app
kubectl apply -f prometheus/python-application/deployment.yaml -n python-app
kubectl apply -f prometheus/nodejs-application/deployment.yaml -n nodejs-app
```

---

## 7. Ingress für Applikationen erstellen

```bash
kubectl apply -f ingress/go-ingress.yaml
kubectl apply -f ingress/python-ingress.yaml
kubectl apply -f ingress/nodejs-ingress.yaml
kubectl apply -f ingress/dotnet-ingress.yaml
```

---

## 8. ServiceMonitors und Endpoints für Applikationen erstellen

```bash
kubectl config use-context kind-dev
kubectl apply -f services/services-applications-node-port.yaml

kubectl config use-context kind-prod
kubectl apply -f PROD Installation/servicemonitors/servicemonitor-dev-apps.yaml 
```

### Cross-Cluster Monitoring Architecture

#### Dev Cluster (Source)
1. **Application** – Running on port **5000**  
2. **NodePort Service** – Exposes app externally via port **30085**

#### Prod Cluster (Monitoring)
3. **Headless Service** – Defines the external service name  
4. **Static Endpoints** – Points to dev cluster `IP:30085`  
5. **ServiceMonitor** – Tells Prometheus to scrape the service

### The Complete Flow
```
Prometheus (prod) → ServiceMonitor → Service → Endpoints → Dev Cluster NodePort → Application (dev)
```
### Why Each Component is Needed

- **NodePort Service (dev):** Makes internal app accessible from outside the cluster  
- **Headless Service (prod):** Provides a Kubernetes service abstraction for external targets  
- **Static Endpoints (prod):** Bridges Kubernetes service discovery to external IPs  
- **ServiceMonitor (prod):** Configures Prometheus Operator to scrape the service

---

## 9. kube-state-metrics installieren

service.yaml wird eventuell nicht benötigt. Je nachdem ob NodePort genutzt wird oder nicht.

```bash
kubectl apply -k kube-state-metrics
```
In dem Prod-Cluster einen ServiceMonitor und Endpoint erstellen, der auf kube-state-metrics des Dev-Clusters zeigt.

```bash
kubectl config use-context kind-prod
kubectl apply -f servicemonitors/servicemonitor-dev-kube-state-metrics.yaml
```

---