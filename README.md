## Install Redis for RateLimit in Envoy AI Gateway

    k apply -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/refs/heads/main/examples/token_ratelimit/redis.yaml

-----------------
```bash
helm upgrade -i aieg-crd oci://docker.io/envoyproxy/ai-gateway-crds-helm \
  --version v0.0.0-latest \
  --namespace envoy-ai-gateway-system \
  --create-namespace
```
```bash
helm upgrade -i aieg oci://docker.io/envoyproxy/ai-gateway-helm \
  --version v0.0.0-latest \
  --namespace envoy-ai-gateway-system \
  --create-namespace

kubectl wait --timeout=2m -n envoy-ai-gateway-system deployment/ai-gateway-controller --for=condition=Available
```
```bash
helm upgrade -i aieg-crd oci://docker.io/envoyproxy/ai-gateway-crds-helm --version v0.0.0-latest --namespace envoy-ai-gateway-system --take-ownership
helm upgrade -i aieg oci://docker.io/envoyproxy/ai-gateway-helm --version v0.0.0-latest --namespace envoy-ai-gateway-system
```
```bash
helm upgrade -i eg oci://docker.io/envoyproxy/gateway-helm \
  --version v0.0.0-latest \
  --namespace envoy-gateway-system \
  --create-namespace \
  -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/manifests/envoy-gateway-values.yaml

kubectl wait --timeout=2m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```

## Basis Usage
```bash
kubectl apply -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/basic/basic.yaml
```
## command output
```text
gatewayclass.gateway.networking.k8s.io/envoy-ai-gateway-basic created
gateway.gateway.networking.k8s.io/envoy-ai-gateway-basic created
clienttrafficpolicy.gateway.envoyproxy.io/client-buffer-limit created
aigatewayroute.aigateway.envoyproxy.io/envoy-ai-gateway-basic created
aiservicebackend.aigateway.envoyproxy.io/envoy-ai-gateway-basic-testupstream created
backend.gateway.envoyproxy.io/envoy-ai-gateway-basic-testupstream created
deployment.apps/envoy-ai-gateway-basic-testupstream created
service/envoy-ai-gateway-basic-testupstream created
envoyproxy.gateway.envoyproxy.io/envoy-ai-gateway-basic created
```
```bash
kubectl get all | grep ai
```
## Command output
```text
pod/envoy-ai-gateway-basic-testupstream-689c96d86f-t46s2     1/1     Running   0          6m18s
service/envoy-ai-gateway-basic-testupstream      ClusterIP   10.43.207.244   <none>        80/TCP                       6m18s
deployment.apps/envoy-ai-gateway-basic-testupstream     1/1     1            1           6m18s
replicaset.apps/envoy-ai-gateway-basic-testupstream-689c96d86f     1         1         1       6m18s
```
```bash
kubectl get svc -n envoy-gateway-system \
  --selector=gateway.envoyproxy.io/owning-gateway-namespace=default,gateway.envoyproxy.io/owning-gateway-name=envoy-ai-gateway-basic
```
## Command output
```text
NAME                                            TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
envoy-default-envoy-ai-gateway-basic-21a9f8f8   LoadBalancer   10.43.80.164   <pending>     80:31488/TCP   6m35s
```

## if the port 80 is occupied change to 8080
```bash
k edit svc envoy-default-envoy-ai-gateway-basic-21a9f8f8 -oyaml -n envoy-gateway-system
```
```bash
root@sumit-thanos-vm:~/envoy-openai# kubectl get svc -n envoy-gateway-system \
  --selector=gateway.envoyproxy.io/owning-gateway-namespace=default,gateway.envoyproxy.io/owning-gateway-name=envoy-ai-gateway-basic
NAME                                            TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)          AGE
envoy-default-envoy-ai-gateway-basic-21a9f8f8   LoadBalancer   10.43.80.164   98.70.41.123   8080:30280/TCP   53m
```

## OpenAI Configuration
curl -O https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/examples/basic/openai.yaml
vi openai.yaml ---->> paste openai key in secret section

```bash
kubectl apply -f openai.yaml
```
## Command output
```text
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
## Command output
```text
pod/envoy-default-envoy-ai-gateway-basic-21a9f8f8-84d876dc8-rbmfd condition met
```



------------------


helm get values aieg -n envoy-ai-gateway-system --all -o yaml >> aieg-values.yaml

helm get values eg -n envoy-gateway-system -o yaml > my-values.yaml

helm upgrade eg oci://docker.io/envoyproxy/gateway-helm \
  -n envoy-gateway-system \
  -f my-values.yaml
