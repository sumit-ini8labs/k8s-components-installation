# Envoy AI Gateway Deployment Guide

This repository contains the configuration and installation steps for deploying the Envoy AI Gateway with Redis-backed rate limiting and OpenAI integration on a Kubernetes cluster.

## Prerequisites

- Kubernetes Cluster (v1.26+)
- Helm 3.0+
- Kubectl CLI
- OpenAI API Key (with available credits)

---

## 1. Infrastructure Setup

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

## 4. Troubleshooting

### Port Conflict Resolution
If port 80 is occupied by another Ingress (e.g., Traefik), patch the service to port 8080:
```bash
k edit svc envoy-default-envoy-ai-gateway-basic-21a9f8f8 -oyaml -n envoy-gateway-system
```
```bash
export GATEWAY_URL=<envoy-default-envoy-ai-gateway-basic_ip:port>
```


### Connectivity Verification
```bash
curl -H "Content-Type: application/json"   -d '{
        "model": "some-cool-self-hosted-model",
        "messages": [
            {
                "role": "system",
                "content": "Hi."
            }
        ]
    }'   $GATEWAY_URL:8080/v1/chat/completions
```
### Output
```text
{"choices":[{"message":{"role":"assistant", "content":"Go ahead, make my day."}}]}
```
---

## 3. OpenAI Provider Setup

### Configure API Credentials
Download the provider manifest and update the secret with your valid OpenAI API key.
```bash
curl -O https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/basic/openai.yaml

# Edit the file to replace the placeholder with your 'Bearer sk-...' key
vi openai.yaml 

kubectl apply -f openai.yaml
```
### check status
```bash
k get aigatewayroute,aiservicebackend,backendsecuritypolicy,backendtlspolicy,backend,secret
```
```bash
kubectl wait pods --timeout=2m \
  -l gateway.envoyproxy.io/owning-gateway-name=envoy-ai-gateway-basic \
  -n envoy-gateway-system \
  --for=condition=Ready
```
```bash
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
```
### check the output
```text
root@sumit-thanos-vm:~/envoy-openai# curl -H "Content-Type: application/json" \
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
{
  "id": "chatcmpl-CwWHnsX9rGY095jUQc3KTPrpueVYb",
  "object": "chat.completion",
  "created": 1768063167,
  "model": "gpt-4o-mini-2024-07-18",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Hello! How can I assist you today?",
        "refusal": null,
        "annotations": []
      },
      "logprobs": null,
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 9,
    "completion_tokens": 9,
    "total_tokens": 18,
    "prompt_tokens_details": {
      "cached_tokens": 0,
      "audio_tokens": 0
    },
    "completion_tokens_details": {
      "reasoning_tokens": 0,
      "audio_tokens": 0,
      "accepted_prediction_tokens": 0,
      "rejected_prediction_tokens": 0
    }
  },
  "service_tier": "default",
  "system_fingerprint": "fp_c4585b5b9c"
}
```


---
### Configure InferencePool and RateLimit
# Install inferencePool and RateLimit
```bash
helm upgrade -i eg oci://docker.io/envoyproxy/gateway-helm \
  --version v0.0.0-latest \
  --namespace envoy-gateway-system \
  --create-namespace \
  -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/manifests/envoy-gateway-values.yaml \
  -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/token_ratelimit/envoy-gateway-values-addon.yaml \
  -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/inference-pool/envoy-gateway-values-addon.yaml
```
### Install Redis for Rate Limiting
Redis is required to store token usage and rate limit states across the gateway instances.
```bash
kubectl apply -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/refs/heads/main/examples/token_ratelimit/redis.yaml
```

## Install the Gateway API Inference Extension CRDs and controller:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api-inference-extension/releases/download/v1.0.1/manifests.yaml
```
## After installing InferencePool CRD, enable InferencePool support in Envoy Gateway, restart the deployment, and wait for it to be ready:
```bash
kubectl apply -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/inference-pool/config.yaml

kubectl rollout restart -n envoy-gateway-system deployment/envoy-gateway

kubectl wait --timeout=2m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```

## Create a Gateway and AIGatewayRoute with multiple InferencePool backends:

# You can configure multiple models under rules section, here i have configure openAI and basic custom Model

```bash
cat <<EOF | kubectl apply -f -
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
    # Route for Custom Model -------->>>>>>>>>>
    - matches:
        - headers:
            - type: Exact
              name: x-ai-eg-model
              value: some-cool-self-hosted-model
      backendRefs:
        - name: envoy-ai-gateway-basic-testupstream
    # Route for OpenAI Model -------->>>>>>>>>>
    - matches:
        - headers:
            - type: Exact
              name: x-ai-eg-model
              value: gpt-4o-mini
      backendRefs:
        - name: envoy-ai-gateway-basic-openai
EOF
```


```bash
k get svc -n envoy-gateway-system | grep inference-pool

root@sumit-thanos-vm:~/envoy-openai# k get svc -n envoy-gateway-system | grep inference-pool
envoy-default-inference-pool-with-aigwroute-d416582c   LoadBalancer   10.43.141.252   98.70.41.123   8081:32358/TCP                                     28m
```
## Test from inferencePool Ip
```text
root@sumit-thanos-vm:~/envoy-openai# curl -H "Content-Type: application/json"   -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {
        "role": "user",
        "content": "Hi."
      }
    ]
  }'   98.70.41.123:8081/v1/chat/completions
{
  "id": "chatcmpl-CwXfZvdBrRV4hU3yJ7oYdwKR3rgJw",
  "object": "chat.completion",
  "created": 1768068485,
  "model": "gpt-4o-mini-2024-07-18",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Hello! How can I assist you today?",
        "refusal": null,
        "annotations": []
      },
      "logprobs": null,
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 9,
    "completion_tokens": 9,
    "total_tokens": 18,
    "prompt_tokens_details": {
      "cached_tokens": 0,
      "audio_tokens": 0
    },
    "completion_tokens_details": {
      "reasoning_tokens": 0,
      "audio_tokens": 0,
      "accepted_prediction_tokens": 0,
      "rejected_prediction_tokens": 0
    }
  },
  "service_tier": "default",
  "system_fingerprint": "fp_c4585b5b9c"
}
```




