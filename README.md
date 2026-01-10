# Envoy AI Gateway Deployment Guide

This repository contains the configuration and installation steps for deploying the Envoy AI Gateway with Redis-backed rate limiting and OpenAI integration on a Kubernetes cluster.

## Prerequisites

- Kubernetes Cluster (v1.26+)
- Helm 3.0+
- Kubectl CLI
- OpenAI API Key (with available credits)

---

## 1. Infrastructure Setup

### Install Redis for Rate Limiting
Redis is required to store token usage and rate limit states across the gateway instances.
```bash
kubectl apply -f [https://raw.githubusercontent.com/envoyproxy/ai-gateway/refs/heads/main/examples/token_ratelimit/redis.yaml](https://raw.githubusercontent.com/envoyproxy/ai-gateway/refs/heads/main/examples/token_ratelimit/redis.yaml)
```

### Install AI Gateway CRDs
Install the Custom Resource Definitions required for InferencePools and AI-specific routing logic.
```bash
helm upgrade -i aieg-crd oci://docker.io/envoyproxy/ai-gateway-crds-helm \
  --version v0.0.0-latest \
  --namespace envoy-ai-gateway-system \
  --create-namespace
```

### Install AI Gateway Controller
The controller manages the lifecycle of the AI proxy instances.
```bash
helm upgrade -i aieg oci://docker.io/envoyproxy/ai-gateway-helm \
  --version v0.0.0-latest \
  --namespace envoy-ai-gateway-system \
  --create-namespace

kubectl wait --timeout=2m -n envoy-ai-gateway-system deployment/ai-gateway-controller --for=condition=Available
```

### Install Envoy Gateway (Control Plane)
This component translates Gateway API resources into Envoy configuration.
```bash
helm upgrade -i eg oci://docker.io/envoyproxy/gateway-helm \
  --version v0.0.0-latest \
  --namespace envoy-gateway-system \
  --create-namespace \
  -f [https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/manifests/envoy-gateway-values.yaml](https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/manifests/envoy-gateway-values.yaml)

kubectl wait --timeout=2m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```

---

## 2. Gateway Configuration

### Deploy Basic AI Gateway Resources
Create the initial Gateway and HTTPRoute resources.
```bash
kubectl apply -f [https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/basic/basic.yaml](https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/basic/basic.yaml)
```

### Verify Service and Ports
Identify the external entry point for the AI Proxy.
```bash
kubectl get svc -n envoy-gateway-system \
  --selector=gateway.envoyproxy.io/owning-gateway-namespace=default,gateway.envoyproxy.io/owning-gateway-name=envoy-ai-gateway-basic
```

---

## 3. OpenAI Provider Setup

### Configure API Credentials
Download the provider manifest and update the secret with your valid OpenAI API key.
```bash
curl -O [https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/basic/openai.yaml](https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/basic/openai.yaml)

# Edit the file to replace the placeholder with your 'Bearer sk-...' key
vi openai.yaml 

kubectl apply -f openai.yaml
```

### Verify Deployment Readiness
```bash
kubectl wait pods --timeout=2m \
  -l gateway.envoyproxy.io/owning-gateway-name=envoy-ai-gateway-basic \
  -n envoy-gateway-system \
  --for=condition=Ready
```

---

## 4. Troubleshooting

### Port Conflict Resolution
If port 80 is occupied by another Ingress (e.g., Traefik), patch the service to port 8080:
```bash
kubectl patch svc <SERVICE_NAME> \
  -n envoy-gateway-system \
  --type='merge' \
  -p '{"spec":{"ports":[{"name":"http","port":8080,"targetPort":8080}]}}'
```

### Connectivity Verification
Test the gateway directly using curl:
```bash
curl -v -X POST http://<EXTERNAL_IP>:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "Hello via Envoy!"}],
    "model": "gpt-4o-mini"
  }'
```

### Maintenance and Backup
Export current Helm values for disaster recovery or migration:
```bash
helm get values aieg -n envoy-ai-gateway-system --all -o yaml > aieg-values.yaml
helm get values eg -n envoy-gateway-system -o yaml > eg-values.yaml
```
