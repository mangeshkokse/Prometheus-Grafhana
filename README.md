
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
