## Install Redis for RateLimit in Envoy AI Gateway

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

```bash
helm install my-redis bitnami/redis \
  --version 24.1.0 \
  -n redis \
  --create-namespace
```
