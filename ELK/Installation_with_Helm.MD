# ELK (Elastic + Kibana) Installation with Helm

This document describes a structured, repeatable installation of Elasticsearch and Kibana using the official Elastic Helm charts.

## Overview
- Helm charts used: `elastic/elasticsearch`, `elastic/kibana`
- Kubernetes namespace: `elastic`
- Security note: avoid embedding plaintext passwords in source files for production; use sealed secrets or Kubernetes secrets.

## Prerequisites
- kubectl configured to your cluster
- Helm 3.x installed and configured
- Enough cluster resources for Elasticsearch and Kibana (storage, CPU, memory)
- (Optional) Node with public IP if you want NodePort access to Kibana

## 1) Add Helm repo
```bash
helm repo add elastic https://helm.elastic.co
helm repo update
```

## 2) Create namespace
```bash
kubectl create namespace elastic
```

## 3) Recommended: use values files (preferred)
Create two YAML files to capture settings for repeatability:

examples/elasticsearch-values.yaml
```yaml
replicas: 2
minimumMasterNodes: 1

resources:
  requests:
    cpu: "200m"
    memory: "512Mi"
  limits:
    cpu: "500m"
    memory: "1Gi"

volumeClaimTemplate:
  resources:
    requests:
      storage: 10Gi

security:
  enabled: true
  tls:
    enabled: true

# Use a secure method in production instead of storing passwords in plaintext
secret:
  password: "MyDevPassword!"

esJavaOpts: "-Xms256m -Xmx256m"
imageTag: "8.15.5"
```

examples/kibana-values.yaml
```yaml
imageTag: "8.5.1"

# Make Kibana point to the Elasticsearch service in the same namespace
elasticsearchHosts: "https://elasticsearch-master:9200"
elasticsearchUsername: "elastic"
elasticsearchPassword: "MyDevPassword!"
elasticsearchCertificateSecret: "elasticsearch-master-certs"

resources:
  requests:
    cpu: "100m"
    memory: "256Mi"
  limits:
    cpu: "300m"
    memory: "512Mi"
```

Then install using:
```bash
helm install elasticsearch elastic/elasticsearch \
  --namespace elastic \
  -f examples/elasticsearch-values.yaml

helm install kibana elastic/kibana \
  --namespace elastic \
  -f examples/kibana-values.yaml
```

## 4) Alternative: inline `--set` install (quick test)
If you prefer quick inline configuration (less reproducible):
```bash
helm install elasticsearch elastic/elasticsearch \
  --namespace elastic \
  --set replicas=2 \
  --set minimumMasterNodes=1 \
  --set resources.requests.cpu="200m" \
  --set resources.requests.memory="512Mi" \
  --set resources.limits.cpu="500m" \
  --set resources.limits.memory="1Gi" \
  --set volumeClaimTemplate.resources.requests.storage="10Gi" \
  --set security.enabled=true \
  --set security.tls.enabled=true \
  --set secret.password="MyDevPassword!" \
  --set esJavaOpts="-Xms256m -Xmx256m" \
  --set imageTag="8.15.5"
```

Kibana inline (quick test):
```bash
helm install kibana elastic/kibana \
  --namespace elastic \
  --version 8.5.1 \
  --set imageTag="8.5.1" \
  --set elasticsearchHosts="https://elasticsearch-master:9200" \
  --set elasticsearchUsername="elastic" \
  --set elasticsearchPassword="MyDevPassword!" \
  --set elasticsearchCertificateSecret="elasticsearch-master-certs" \
  --set resources.requests.cpu="100m" \
  --set resources.requests.memory="256Mi" \
  --set resources.limits.cpu="300m" \
  --set resources.limits.memory="512Mi"
```

## 5) Verification

Port-forward to test Elasticsearch locally:
```bash
kubectl port-forward svc/elasticsearch-master 9200:9200 -n elastic
# in another terminal:
curl --insecure -u elastic:MyDevPassword! https://localhost:9200
```

Check Kibana pod logs and health:
```bash
kubectl get pods -n elastic
kubectl logs -l app=kibana -n elastic
```

## 6) Expose Kibana (NodePort example)
Create a NodePort service and bind it to pods with label `app=kibana`:

```bash
kubectl create service nodeport kibana-external \
  --tcp=5601:5601 \
  --node-port=30007 \
  -n elastic

# Ensure the Service selects Kibana pods:
kubectl patch service kibana-external -n elastic -p '{"spec": {"selector": {"app": "kibana"}}}'
```

Access from a browser:
- http://<node-ip>:30007

Notes:
- For production use an Ingress with TLS or a LoadBalancer service.
- NodePort is suitable for quick testing only.

## 7) Troubleshooting & tips
- Pods stuck in CrashLoopBackOff: check pod logs and events:
  kubectl describe pod <pod> -n elastic
  kubectl logs <pod> -n elastic
- PVC not bound: verify StorageClass and available capacity
- TLS/auth errors: ensure `elasticsearchCertificateSecret` references the correct secret created by the chart or provide your own certificates
- To avoid plaintext secrets in git: use SealedSecrets, HashiCorp Vault, or SOPS-encrypted files
- For development, you can set resources to smaller values, but follow Elastic's official recommendations for production.

## 8) Cleanup
```bash
helm uninstall kibana -n elastic
helm uninstall elasticsearch -n elastic
kubectl delete namespace elastic
```

## 9) References
- Elastic Helm charts: https://github.com/elastic/helm-charts
- Chart values documentation:
  - Elasticsearch: https://www.elastic.co/guide/en/elasticsearch/reference/current/helm.html
  - Kibana: https://www.elastic.co/guide/en/kibana/current/helm.html

---

If you'd like, I can:
- Convert the example values into actual files and commit them to the repo (examples/elasticsearch-values.yaml and examples/kibana-values.yaml).
- Add a simple Ingress example for Kibana (with TLS).
- Replace the current file in the repo with this README.
