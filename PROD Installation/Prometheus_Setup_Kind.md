# Kubernetes Cluster Einrichtung mit Kind, Traefik und Prometheus

Alle Dateien zur Einrichtung liegen unter:  
`E:\kind\PROD Installation`

Alle Commands werden aus dem Ordner  
`E:\kind\PROD Installation` ausgeführt.

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

## 4. Kube-Prometheus-Stack installieren

```bash
kubectl create namespace monitoring
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring --values values.yaml
```

---

## 5. Beispielapplikationen Images laden und ausführen

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

## 6. Ingress für Applikationen, Prometheus und Grafana erstellen

```bash
kubectl apply -f ingress/go-ingress.yaml
kubectl apply -f ingress/python-ingress.yaml
kubectl apply -f ingress/nodejs-ingress.yaml
kubectl apply -f ingress/dotnet-ingress.yaml
kubectl apply -f ingress/prometheus-ingress.yaml
kubectl apply -f ingress/grafana-ingress.yaml
```

---

## 7. ServiceAccount erstellen

```bash
kubectl apply -f serviceaccount.yaml
```

---

## 8. Prometheus Instanz im Namespace Monitoring erstellen

```bash
kubectl apply -f prometheus.yaml
```

---

## 9. ServiceMonitors erstellen

```bash
kubectl apply -f servicemonitors/servicemonitor-apps.yaml
```

Optional

```bash
kubectl apply -f servicemonitors/servicemonitor-kube-state-metrics.yaml
kubectl apply -f servicemonitors/servicemonitor-node-exporter.yaml
```


---

## 10. Zugriff auf Applikationen per Port-Forwarding

```bash
kubectl port-forward -n traefik service/traefik 8080:80
```

---

## 11. Überprüfung & Konfiguration in Prometheus / Grafana

- In **Prometheus** unter **Status → Service Discovery** erscheinen die ServiceMonitors.  
- Unter **Status → Target Health** prüfen, ob Scraping funktioniert.  
- In **Grafana** eine neue **Data Source** erstellen:  
  - Name: `prometheus-applications`  
  - URL: `http://prometheus-operated.monitoring:9090`
- Dashboard importieren:  
  `monitoring/prometheus/kubernetes/prometheus-operator/dashboard.json`

---

