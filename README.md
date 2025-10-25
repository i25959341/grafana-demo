# Monitoring Demo Stack

Grafana + Prometheus + podinfo sample application, provisioned with Kustomize to demonstrate dashboards and alerting data flow.

## Prerequisites
- Docker (required by kind)
- [kind](https://kind.sigs.k8s.io/) ≥ 0.20
- kubectl configured to talk to the kind cluster

## Spin Up Kubernetes
```bash
kind create cluster --name grafana-demo
kubectl config use-context kind-grafana-demo
```

## Deploy Monitoring Stack
From the repository root, apply the root Kustomization to install Prometheus, Grafana, Elasticsearch, and the podinfo demo app (includes the demo-data job that seeds an Elasticsearch index):
```bash
kustomize build --enable-helm monitoring | kubectl apply -f -
kubectl rollout status deploy/prometheus -n monitoring
kubectl rollout status deploy/grafana -n monitoring
kubectl rollout status deploy/podinfo -n monitoring
kubectl rollout status statefulset/elasticsearch-master -n monitoring
```

## Access the UIs
```bash
# Grafana
kubectl port-forward svc/grafana 3000:3000 -n monitoring
# Prometheus
kubectl port-forward svc/prometheus 9090:9090 -n monitoring
```
- Grafana UI: http://localhost:3000 (default admin user/pass is `admin`/`admin`)
- Prometheus UI: http://localhost:9090

Grafana auto-provisions two data sources:
- **Prometheus** (default) for metrics.
- **Elasticsearch** pointing at `https://elasticsearch-master:9200` with credentials `elastic/changeme` and TLS verification disabled for the demo.

In Grafana, the pre-provisioned **Prometheus** data source and **Podinfo Overview** dashboard load automatically. Generate traffic with:
```bash
kubectl port-forward svc/podinfo 9898:9898 -n monitoring 
curl -s localhost:9898/ 
```
Refresh the dashboard to confirm request metrics appear.

In Grafana’s Explore view, switch to the Elasticsearch data source and query `demo-metrics` (for example `value: *`) to verify the seeded documents.

## Tear Down
```bash
kustomize build --enable-helm monitoring | kubectl delete -f -
kind delete cluster --name grafana-demo
```

## Extension Plan
- Carve out a reusable base under `monitoring/base` that contains the shared namespace, Prometheus, Grafana, podinfo, and Elasticsearch definitions.
- Create environment overlays (e.g. `monitoring/overlays/dev` and `monitoring/overlays/prod`) that reference the base and layer on environment-specific patches—image tags, replica counts, Helm values, and secrets.
- Update delivery tooling: either have CI/CD call `kustomize build monitoring/overlays/<env>` per environment, or adopt Argo CD with an app-of-apps pattern that points at the overlays for continuous delivery.
- Expand observability by adding a log-forwarder that ships pod logs into Elasticsearch, and extend the Grafana provisioning with an Elasticsearch datasource plus a log dashboard to demonstrate metrics + logs in one view.
