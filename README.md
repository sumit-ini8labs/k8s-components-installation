markdown
# Envoy AI Gateway Deployment Guide

This repository provides a complete guide for deploying the **Envoy AI Gateway** with **Redis-backed rate limiting** and **OpenAI integration** on a Kubernetes cluster. The Envoy AI Gateway enables intelligent routing, inference pooling, and secure API management for AI workloads.

---

## 📋 Prerequisites

Before you begin, ensure you have the following:

- Kubernetes Cluster (v1.26+)
- Helm 3.0+
- Kubectl CLI
- Valid OpenAI API Key (with available credits)

---

## 🚀 1. Infrastructure Setup

### Install AI Gateway CRDs
The Custom Resource Definitions (CRDs) are required for **InferencePools** and AI-specific routing logic.
```bash
helm upgrade -i aieg-crd oci://docker.io/envoyproxy/ai-gateway-crds-helm \
  --version v0.0.0-latest \
  --namespace envoy-ai-gateway-system \
  --create-namespace
Install AI Gateway Controller
The controller manages the lifecycle of AI proxy instances.

bash
helm upgrade -i aieg oci://docker.io/envoyproxy/ai-gateway-helm \
  --version v0.0.0-latest \
  --namespace envoy-ai-gateway-system \
  --create-namespace

kubectl wait --timeout=2m -n envoy-ai-gateway-system deployment/ai-gateway-controller --for=condition=Available
Install Envoy Gateway (Control Plane)
The Envoy Gateway translates Gateway API resources into Envoy configuration.

bash
helm upgrade -i eg oci://docker.io/envoyproxy/gateway-helm \
  --version v0.0.0-latest \
  --namespace envoy-gateway-system \
  --create-namespace \
  -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/manifests/envoy-gateway-values.yaml

kubectl wait --timeout=2m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
⚙️ 2. Gateway Configuration
Deploy Basic AI Gateway Resources
Create the initial Gateway and HTTPRoute resources.

bash
kubectl apply -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/basic/basic.yaml
Verify Service and Ports
Identify the external entry point for the AI Proxy.

bash
kubectl get svc -n envoy-gateway-system \
  --selector=gateway.envoyproxy.io/owning-gateway-namespace=default,gateway.envoyproxy.io/owning-gateway-name=envoy-ai-gateway-basic
🛠️ 3. OpenAI Provider Setup
Configure API Credentials
Download the provider manifest and update the secret with your valid OpenAI API key.

bash
curl -O https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/basic/openai.yaml

# Edit the file to replace the placeholder with your 'Bearer sk-...' key
vi openai.yaml 

kubectl apply -f openai.yaml
Check Status
bash
kubectl get aigatewayroute,aiservicebackend,backendsecuritypolicy,backendtlspolicy,backend,secret
Wait for pods to be ready:

bash
kubectl wait pods --timeout=2m \
  -l gateway.envoyproxy.io/owning-gateway-name=envoy-ai-gateway-basic \
  -n envoy-gateway-system \
  --for=condition=Ready
Test Connectivity
bash
curl -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {
        "role": "user",
        "content": "Hi."
      }
    ]
  }' \
  $GATEWAY_URL/v1/chat/completions
Expected output:

json
{
  "id": "chatcmpl-xxxx",
  "object": "chat.completion",
  "model": "gpt-4o-mini-2024-07-18",
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "Hello! How can I assist you today?"
      }
    }
  ]
}
⚡ 4. Troubleshooting
Port Conflict Resolution
If port 80 is already occupied (e.g., by Traefik), patch the service to use port 8080:

bash
kubectl edit svc envoy-default-envoy-ai-gateway-basic-21a9f8f8 -n envoy-gateway-system
Then export the new gateway URL:

bash
export GATEWAY_URL=<envoy-default-envoy-ai-gateway-basic_ip:port>
Connectivity Verification
bash
curl -H "Content-Type: application/json" -d '{
  "model": "some-cool-self-hosted-model",
  "messages": [
    {
      "role": "system",
      "content": "Hi."
    }
  ]
}' $GATEWAY_URL:8080/v1/chat/completions
📊 5. InferencePool and Rate Limiting
Enable InferencePool and RateLimit
bash
helm upgrade -i eg oci://docker.io/envoyproxy/gateway-helm \
  --version v0.0.0-latest \
  --namespace envoy-gateway-system \
  --create-namespace \
  -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/manifests/envoy-gateway-values.yaml \
  -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/token_ratelimit/envoy-gateway-values-addon.yaml \
  -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/inference-pool/envoy-gateway-values-addon.yaml
Install Redis for Rate Limiting
Redis stores token usage and rate limit states across gateway instances.

bash
kubectl apply -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/refs/heads/main/examples/token_ratelimit/redis.yaml
Install Gateway API Inference Extension
bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api-inference-extension/releases/download/v1.0.1/manifests.yaml
Enable InferencePool Support
bash
kubectl apply -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/inference-pool/config.yaml

kubectl rollout restart -n envoy-gateway-system deployment/envoy-gateway

kubectl wait --timeout=2m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
🔀 6. Multi-Model Gateway Configuration
You can configure multiple models under the rules section. Example: OpenAI and a custom self-hosted model.

yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: inference-pool-with-aigwroute
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: inference-pool-with-aigwroute
  namespace: default
spec:
  gatewayClassName: inference-pool-with-aigwroute
  listeners:
    - name: http
      protocol: HTTP
      port: 80
---
apiVersion: aigateway.envoyproxy.io/v1alpha1
kind: AIGatewayRoute
metadata:
  name: inference-pool-with-aigwroute
  namespace: default
spec:
  parentRefs:
    - name: inference-pool-with-aigwroute
      kind: Gateway
      group: gateway.networking.k8s.io
  rules:
    # Route for Custom Model
    - matches:
        - headers:
            - type: Exact
              name: x-ai-eg-model
              value: some-cool-self-hosted-model
      backendRefs:
        - name: envoy-ai-gateway-basic-testupstream
    # Route for OpenAI Model
    - matches:
        - headers:
            - type: Exact
              name: x-ai-eg-model
              value: gpt-4o-mini
      backendRefs:
        - name: envoy-ai-gateway-basic-openai
✅ 7. Testing InferencePool
Check the service:

bash
kubectl get svc -n envoy-gateway-system | grep inference-pool
Example output:

text
envoy-default-inference-pool-with-aigwroute-d416582c   LoadBalancer   10.43.141.252   98.70.41.123   8081:32358/TCP   28m
Test with the external IP:

bash
curl -H "Content-Type: application/json" -d '{
  "model": "gpt-4o-mini",
  "messages": [
    {
      "role": "user",
      "content": "Hi."
    }
  ]
}' 98.70.41.123:8081/v1/chat/completions
Expected response:

json
{
  "id": "chatcmpl-xxxx",
  "object": "chat.completion",
  "model": "gpt-4o-mini-2024-07-18",
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "Hello! How can I assist you today?"
      }
    }
  ]
}
