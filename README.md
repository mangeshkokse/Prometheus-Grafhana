
# centralized Prometheus monitoring setup

To set up a centralized Prometheus monitoring solution with Prometheus Agents running in each EKS cluster, using Remote Write to a central Prometheus, and storing long-term data in AWS S3, follow this complete guide with YAML configurations.

## Architecture Overview
```css
[EKS Cluster 1] → Prometheus Agent → Remote Write → [Central Prometheus] → [Grafana]
[EKS Cluster 2] → Prometheus Agent → Remote Write → [Central Prometheus] → [Grafana]
[EKS Cluster 3] → Prometheus Agent → Remote Write → [Central Prometheus] → [Grafana]
[EKS Cluster 4] → Prometheus Agent → Remote Write → [Central Prometheus] → [Grafana]
[EKS Cluster 5] → Prometheus Agent → Remote Write → [Central Prometheus] → [Grafana]
                                   ↓
                               [AWS S3]
```

- Each EKS cluster runs a Prometheus Agent, which forwards metrics to the central Prometheus via Remote Write.
- The central Prometheus stores recent data and forwards long-term storage to AWS S3.

## Step 1: Deploy Prometheus Agent in Each EKS Cluster
Each EKS cluster runs a Prometheus Agent instead of a full Prometheus instance. The agent does not store metrics locally; it only forwards them to the central Prometheus.

**1.1 Install Prometheus Agent using Helm**
```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus-agent prometheus-community/prometheus \
  --namespace monitoring --create-namespace \
  --set server.enabled=false \
  --set alertmanager.enabled=false \
  --set prometheus.prometheusSpec.remoteWrite[0].url="http://central-prometheus.monitoring.svc.cluster.local:9090/api/v1/write"
```
**1.2 Custom `values.yaml` for Prometheus Agent**
If you want to use helm upgrade --values values.yaml, use the following configuration:
```yaml
server:
  enabled: false

alertmanager:
  enabled: false

prometheus:
  prometheusSpec:
    scrapeInterval: 15s
    evaluationInterval: 15s
    remoteWrite:
      - url: "http://central-prometheus.monitoring.svc.cluster.local:9090/api/v1/write"
    storageSpec:
      emptyDir: {}
    additionalScrapeConfigs:
      name: additional-scrape-configs
      key: scrape-configs.yaml
```
Apply the configuration:
```sh
helm upgrade --install prometheus-agent prometheus-community/prometheus -f values.yaml -n monitoring
```
## Step 2: Deploy Central Prometheus in a Dedicated Cluster
The central Prometheus instance collects metrics from all Prometheus Agents.

**2.1 Install Central Prometheus using Helm**
```sh
helm install central-prometheus prometheus-community/prometheus \
  --namespace monitoring --create-namespace \
  --set persistence.enabled=true \
  --set persistence.size=50Gi
```
**2.2 Custom values.yaml for Central Prometheus**
```yaml
server:
  persistentVolume:
    enabled: true
    size: 50Gi

alertmanager:
  enabled: true

prometheus:
  prometheusSpec:
    scrapeInterval: 15s
    evaluationInterval: 15s
    retention: 10d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp2
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
    remoteWrite:
      - url: "s3://your-bucket/prometheus-data"
    additionalScrapeConfigs:
      name: additional-scrape-configs
      key: scrape-configs.yaml
```
Apply the configuration:
```sh
helm upgrade --install central-prometheus prometheus-community/prometheus -f values.yaml -n monitoring
```
## Step 3: Configure AWS S3 Storage for Long-Term Metrics
Prometheus does not support direct AWS S3 storage. Use Thanos or Cortex for long-term storage.

**3.1 Deploy Thanos Sidecar**
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install thanos bitnami/thanos \
  --namespace monitoring --create-namespace \
  --set storeGateway.enabled=true \
  --set bucketConfig.secretName=thanos-bucket \
  --set bucketConfig.secretKey=bucket-config.yaml
```
**3.2 Create a Kubernetes Secret for AWS S3**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: thanos-bucket
  namespace: monitoring
type: Opaque
stringData:
  bucket-config.yaml: |
    type: S3
    config:
      bucket: your-bucket-name
      endpoint: s3.amazonaws.com
      access_key: YOUR_ACCESS_KEY
      secret_key: YOUR_SECRET_KEY
```
Apply it:
```sh
kubectl apply -f thanos-bucket-secret.yaml
```

## Step 4: Deploy Grafana in the Central Cluster
Grafana will visualize metrics from Central Prometheus.

**4.1 Install Grafana using Helm**
```sh
helm install grafana prometheus-community/grafana \
  --namespace monitoring --create-namespace \
  --set adminPassword='Admin123' \
  --set service.type=LoadBalancer
```
**4.2 Configure Grafana Data Source**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasource
  namespace: monitoring
data:
  prometheus-datasource.yaml: |
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://central-prometheus.monitoring.svc.cluster.local:9090
      access: proxy
      isDefault: true
```
Apply:
```sh
kubectl apply -f grafana-datasource.yaml
```
## Verifying Setup
Check Prometheus Remote Write Logs
```sh
kubectl logs -l app=prometheus -n monitoring
```
Check Central Prometheus Targets
```sh
kubectl port-forward svc/central-prometheus-server -n monitoring 9090:9090
```
Go to http://localhost:9090/targets to verify that Prometheus Agents are sending metrics.
Access Grafana
- Find the LoadBalancer IP:
```sh
kubectl get svc -n monitoring grafana
```
- Open Grafana at `http://<LOADBALANCER-IP>:3000` and login with
  - Username: `admin`
  - Password: `Admin123`
- Add Prometheus as a data source (if not added automatically).






