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

## 3) Install Elasticsearch

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
### Verification for Elasticsearch using port-forwarding
```bash
kubectl port-forward svc/elasticsearch-master 9200:9200 -n elastic
```
open a new terminal and send a curl request ;
```bash
curl --insecure -u elastic:MyDevPassword! https://localhost:9200
```

## 4) Install Kibana

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
```


### Create a NodePort service to expose Kibana externally.

 ```bash
 kubectl create service nodeport kibana-external \
  --tcp=5601:5601 \
  --node-port=30007


kubectl patch service kibana-external -p '{"spec": {"selector": {"app": "kibana"}}}'
 ```

### Accessing Kibana
To access Kibana, open the following URL in your browser: http://nodeip:30007

