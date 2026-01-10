## Install Redis for RateLimit in Envoy AI Gateway

```bash
k apply -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/refs/heads/main/examples/token_ratelimit/redis.yaml
```
-----------------
helm upgrade -i aieg-crd oci://docker.io/envoyproxy/ai-gateway-crds-helm \
  --version v0.0.0-latest \
  --namespace envoy-ai-gateway-system \
  --create-namespace

helm upgrade -i aieg oci://docker.io/envoyproxy/ai-gateway-helm \
  --version v0.0.0-latest \
  --namespace envoy-ai-gateway-system \
  --create-namespace

kubectl wait --timeout=2m -n envoy-ai-gateway-system deployment/ai-gateway-controller --for=condition=Available


helm upgrade -i aieg-crd oci://docker.io/envoyproxy/ai-gateway-crds-helm --version v0.0.0-latest --namespace envoy-ai-gateway-system --take-ownership
helm upgrade -i aieg oci://docker.io/envoyproxy/ai-gateway-helm --version v0.0.0-latest --namespace envoy-ai-gateway-system

helm upgrade -i eg oci://docker.io/envoyproxy/gateway-helm \
  --version v0.0.0-latest \
  --namespace envoy-gateway-system \
  --create-namespace \
  -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/manifests/envoy-gateway-values.yaml

kubectl wait --timeout=2m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available


## Basis Usage


kubectl apply -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/basic/basic.yaml

k edit svc envoy-default-envoy-ai-gateway-basic-21a9f8f8 -oyaml -n envoy-gateway-system ----> if it loadbalancer change port to 80 is occupied change to 8080

## OpenAI Configuration
curl -O https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/basic/openai.yaml
vi openai.yaml ---->> paste openai key in secret section

```bash
kubectl apply -f openai.yaml
```
```md
root@sumit-thanos-vm:~/envoy-openai# k apply -f ./openai.yaml
aigatewayroute.aigateway.envoyproxy.io/envoy-ai-gateway-basic-openai created
aiservicebackend.aigateway.envoyproxy.io/envoy-ai-gateway-basic-openai created
backendsecuritypolicy.aigateway.envoyproxy.io/envoy-ai-gateway-basic-openai-apikey created
backend.gateway.envoyproxy.io/envoy-ai-gateway-basic-openai created
Warning: The v1alpha3 version of BackendTLSPolicy has been deprecated and will be removed in a future release of the API. Please upgrade to v1.
backendtlspolicy.gateway.networking.k8s.io/envoy-ai-gateway-basic-openai-tls created
secret/envoy-ai-gateway-basic-openai-apikey created
```


```bash
kubectl wait pods --timeout=2m \
  -l gateway.envoyproxy.io/owning-gateway-name=envoy-ai-gateway-basic \
  -n envoy-gateway-system \
  --for=condition=Ready
```
```text
root@sumit-thanos-vm:~/envoy-openai# kubectl wait pods --timeout=2m \
  -l gateway.envoyproxy.io/owning-gateway-name=envoy-ai-gateway-basic \
  -n envoy-gateway-system \
  --for=condition=Ready
pod/envoy-default-envoy-ai-gateway-basic-21a9f8f8-84d876dc8-rbmfd condition met
```



------------------


helm get values aieg -n envoy-ai-gateway-system --all -o yaml >> aieg-values.yaml

helm get values eg -n envoy-gateway-system -o yaml > my-values.yaml

helm upgrade eg oci://docker.io/envoyproxy/gateway-helm \
  -n envoy-gateway-system \
  -f my-values.yaml
