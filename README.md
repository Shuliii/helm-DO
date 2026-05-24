# helm-DO

Helm charts and shared Kubernetes resources for the DigitalOcean cluster.

## Structure

```
helm-DO/
  huatah/       - huatah.co app (frontend, backend, auth, scrapers, mysql)
  igokochi/     - igokochihouse.com app (WIP)
  shared/       - applied directly with kubectl (ingress, cert issuer)
```

## Prerequisites

1. A running DigitalOcean Kubernetes cluster
2. cert-manager installed on the cluster:
   ```
   kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
   ```
3. Secrets created manually (see below — never commit secret values to git)

## Secrets

Create these before running helm install.

### huatah

```
kubectl create secret generic huatah-secret \
  --from-literal=MYSQL_ROOT_PASSWORD=<root password> \
  --from-literal=MYSQL_USER=<app db user> \
  --from-literal=MYSQL_PASSWORD=<app db password> \
  --from-literal=MYSQL_DATABASE=<database name> \
  --from-literal=JWT_SECRET=<jwt signing secret>
```

### igokochi

```
kubectl create secret generic igokochi-secret \
  --from-literal=MYSQL_ROOT_PASSWORD=<root password> \
  --from-literal=MYSQL_USER=<app db user> \
  --from-literal=MYSQL_PASSWORD=<app db password> \
  --from-literal=MYSQL_DATABASE=<database name> \
  --from-literal=JWT_SECRET=<jwt signing secret>
```

Store the actual values in a password manager (not in this repo).

## Deploy order

Sequence matters — cert-manager must exist before the issuer, ingress goes last so
TLS certs are ready before traffic hits the services.

```
# 1. Create secrets (see above)

# 2. Apply cert issuer
kubectl apply -f shared/issuer.yaml

# 3. Install Helm charts
helm install huatah ./huatah
helm install igokochi ./igokochi

# 4. Apply shared ingress (after pods are running)
kubectl apply -f shared/ingress.yaml
```

## Upgrading after changes

```
helm upgrade huatah ./huatah
helm upgrade igokochi ./igokochi
```

For shared resources (ingress, issuer), re-apply with kubectl:
```
kubectl apply -f shared/ingress.yaml
```

## Verify

```
helm list
kubectl get pods
kubectl get ingress
kubectl get certificate
```

The `huatah-co-tls` certificate should show `READY=True` within a few minutes.
If it is missing, cert-manager will recreate it automatically from the ingress annotation.

## Domains

| Domain               | App      |
|----------------------|----------|
| huatah.co            | huatah   |
| www.huatah.co        | huatah   |
| igokochihouse.com    | igokochi |
| www.igokochihouse.com| igokochi |

Both domains share one TLS secret (`huatah-co-tls`) and one LoadBalancer to avoid extra cost.