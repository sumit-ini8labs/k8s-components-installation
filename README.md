## Install Redis for RateLimit in Envoy AI Gateway


1. install redis for ratelimit 

    k apply -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/refs/heads/main/examples/token_ratelimit/redis.yaml

2. Install crds
    helm upgrade -i aieg-crd oci://docker.io/envoyproxy/ai-gateway-crds-helm \
      --version v0.0.0-latest \
      --namespace envoy-ai-gateway-system \
      --create-namespace
3. Install envoy ai controller
    helm upgrade -i aieg oci://docker.io/envoyproxy/ai-gateway-helm \
      --version v0.0.0-latest \
      --namespace envoy-ai-gateway-system \
      --create-namespace

    kubectl wait --timeout=2m -n envoy-ai-gateway-system deployment/ai-gateway-controller --for=condition=Available

4. Take ownershiip

    helm upgrade -i aieg-crd oci://docker.io/envoyproxy/ai-gateway-crds-helm --version v0.0.0-latest --namespace envoy-ai-gateway-system --take-ownership
    helm upgrade -i aieg oci://docker.io/envoyproxy/ai-gateway-helm --version v0.0.0-latest --namespace envoy-ai-gateway-system

5. install eg
    helm upgrade -i eg oci://docker.io/envoyproxy/gateway-helm \
      --version v0.0.0-latest \
      --namespace envoy-gateway-system \
      --create-namespace \
      -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/manifests/envoy-gateway-values.yaml
    
    kubectl wait --timeout=2m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available


## Basis Usage

    kubectl apply -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/basic/basic.yaml
```

    kubectl get all | grep ai

```
    kubectl get svc -n envoy-gateway-system \
      --selector=gateway.envoyproxy.io/owning-gateway-namespace=default,gateway.envoyproxy.io/owning-gateway-name=envoy-ai-gateway-basic

```
## if the port 80 is occupied change to 8080

    k edit svc envoy-default-envoy-ai-gateway-basic-21a9f8f8 -oyaml -n envoy-gateway-system


## OpenAI Configuration
    curl -O https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/basic/openai.yaml
    vi openai.yaml ---->> paste openai key in secret section
```
    kubectl apply -f openai.yaml
```
    kubectl wait pods --timeout=2m \
      -l gateway.envoyproxy.io/owning-gateway-name=envoy-ai-gateway-basic \
      -n envoy-gateway-system \
      --for=condition=Ready
```



------------------


helm get values aieg -n envoy-ai-gateway-system --all -o yaml >> aieg-values.yaml

helm get values eg -n envoy-gateway-system -o yaml > my-values.yaml

helm upgrade eg oci://docker.io/envoyproxy/gateway-helm \
  -n envoy-gateway-system \
  -f my-values.yaml
