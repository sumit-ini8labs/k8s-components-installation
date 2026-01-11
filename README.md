# Envoy AI Gateway Deployment Guide

This repository contains the **end-to-end configuration and installation steps** for deploying **Envoy AI Gateway** with:

* OpenAI integration
* Custom/self-hosted model routing
* Redis-backed **token-based global rate limiting**
* InferencePool support

All examples are tested on a Kubernetes cluster using Envoy Gateway and Gateway API.

---

## Architecture Overview

```
Client / App
   |
   |  POST /v1/chat/completions
   |  Headers: x-ai-eg-model, x-user-id
   v
LoadBalancer / Service
   |
   v
Envoy Gateway (Data Plane)
   |
   |-- Header-based routing (AIGatewayRoute)
   |-- Token extraction
   |-- Global Rate Limiting
   |        |
   |        v
   |      Redis
   |
   +--> Custom Model (InferencePool)
   |
   +--> OpenAI (gpt-4o-mini)

AI Gateway Controller (Control Plane)
   - Watches CRDs
   - Programs Envoy dynamically
```

---

## Prerequisites

* Kubernetes Cluster **v1.26+**
* Helm **v3+**
* kubectl CLI
* OpenAI API Key with available credits

---

## 1. Infrastructure Setup

### Install AI Gateway CRDs

Install the Custom Resource Definitions required for **InferencePools** and AI-specific routing.

```bash
helm upgrade -i aieg-crd oci://docker.io/envoyproxy/ai-gateway-crds-helm \
  --version v0.0.0-latest \
  --namespace envoy-ai-gateway-system \
  --create-namespace
```

---

### Install AI Gateway Controller

The controller manages the lifecycle of AI Gateway resources.

```bash
helm upgrade -i aieg oci://docker.io/envoyproxy/ai-gateway-helm \
  --version v0.0.0-latest \
  --namespace envoy-ai-gateway-system \
  --create-namespace

kubectl wait --timeout=2m \
  -n envoy-ai-gateway-system deployment/ai-gateway-controller \
  --for=condition=Available
```

---

### Install Envoy Gateway (Control Plane)

Envoy Gateway translates **Gateway API** and **AI Gateway CRDs** into Envoy configuration.

```bash
helm upgrade -i eg oci://docker.io/envoyproxy/gateway-helm \
  --version v0.0.0-latest \
  --namespace envoy-gateway-system \
  --create-namespace \
  -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/manifests/envoy-gateway-values.yaml

kubectl wait --timeout=2m \
  -n envoy-gateway-system deployment/envoy-gateway \
  --for=condition=Available
```

---

## 2. Gateway Configuration

### Deploy Basic AI Gateway Resources

```bash
kubectl apply -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/basic/basic.yaml
```

---

### Verify Gateway Service

```bash
kubectl get svc -n envoy-gateway-system \
  --selector=gateway.envoyproxy.io/owning-gateway-namespace=default,
           gateway.envoyproxy.io/owning-gateway-name=envoy-ai-gateway-basic
```

---

## 3. Troubleshooting

### Port Conflict Resolution

If port **80** is already in use (e.g. Traefik), update the service to use **8080**:

```bash
kubectl edit svc envoy-default-envoy-ai-gateway-basic-xxxxx -n envoy-gateway-system
```

Set the gateway URL:

```bash
export GATEWAY_URL=<EXTERNAL-IP:PORT>
```

---

### Connectivity Verification (Custom Model)

```bash
curl -H "Content-Type: application/json" \
  -d '{
        "model": "some-cool-self-hosted-model",
        "messages": [{"role": "system", "content": "Hi."}]
      }' \
  $GATEWAY_URL:8080/v1/chat/completions
```

Expected output:

```text
{"choices":[{"message":{"role":"assistant","content":"Go ahead, make my day."}}]}
```

---

## 4. OpenAI Provider Setup

### Configure OpenAI Credentials

```bash
curl -O https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/basic/openai.yaml
vi openai.yaml
kubectl apply -f openai.yaml
```

---

### Verify Resources

```bash
kubectl get aigatewayroute,aiservicebackend,backendsecuritypolicy,backendtlspolicy,backend,secret
```

```bash
kubectl wait pods --timeout=2m \
  -l gateway.envoyproxy.io/owning-gateway-name=envoy-ai-gateway-basic \
  -n envoy-gateway-system \
  --for=condition=Ready
```

---

### Test OpenAI Integration

```bash
curl -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Hi."}]
  }' \
  $GATEWAY_URL/v1/chat/completions
```

---

## 5. InferencePool & Rate Limiting Setup

### Enable InferencePool and RateLimit Add-ons

```bash
helm upgrade -i eg oci://docker.io/envoyproxy/gateway-helm \
  --version v0.0.0-latest \
  --namespace envoy-gateway-system \
  -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/manifests/envoy-gateway-values.yaml \
  -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/token_ratelimit/envoy-gateway-values-addon.yaml \
  -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/inference-pool/envoy-gateway-values-addon.yaml
```

---

### Install Redis

```bash
kubectl apply -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/token_ratelimit/redis.yaml
```

---

### Install Gateway API Inference Extension

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api-inference-extension/releases/download/v1.0.1/manifests.yaml
```

---

### Enable InferencePool Support

```bash
kubectl apply -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/inference-pool/config.yaml
kubectl rollout restart -n envoy-gateway-system deployment/envoy-gateway
kubectl wait --timeout=2m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```

---

## 6. Gateway & AIGatewayRoute with Multiple Models

```yaml
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
    - matches:
        - headers:
            - name: x-ai-eg-model
              type: Exact
              value: some-cool-self-hosted-model
      backendRefs:
        - name: envoy-ai-gateway-basic-testupstream
    - matches:
        - headers:
            - name: x-ai-eg-model
              type: Exact
              value: gpt-4o-mini
      backendRefs:
        - name: envoy-ai-gateway-basic-openai
```

---

## 7. Configure Token-Based Rate Limiting

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: BackendTrafficPolicy
metadata:
  name: openai-token-limit-policy
  namespace: default
spec:
  targetRefs:
    - name: inference-pool-with-aigwroute
      kind: Gateway
      group: gateway.networking.k8s.io
  rateLimit:
    type: Global
    global:
      rules:
        - clientSelectors:
            - headers:
                - name: x-user-id
                  type: Distinct
                - name: x-ai-eg-model
                  type: Exact
                  value: gpt-4o-mini
          limit:
            requests: 100
            unit: Hour
          cost:
            request:
              from: Number
              number: 0
            response:
              from: Metadata
              metadata:
                namespace: io.envoy.ai_gateway
                key: llm_total_token
```

---

### Rate Limit Test

```bash
curl -H "Content-Type: application/json" \
  -H "x-user-id: user123" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Hello!"}]}' \
  <INFERENCE-POOL-IP:PORT>/v1/chat/completions
```

Expected behavior after limit is exceeded:

```text
{"type":"error","error":{"type":"OpenAIBackendError","code":"504","message":"upstream request timeout"}}
```

---

## Summary

* Single gateway endpoint for multiple LLM backends
* Header-based routing
* Token-aware global rate limiting
* Redis-backed consistency
* Production-ready AI traffic control
