# Bootstrap Secrets

These secrets must be created MANUALLY before ArgoCD can manage the cluster.
Run these AFTER ArgoCD is installed but BEFORE applying the root app.

## 1. 1Password Connect Credentials

1. Go to 1Password.com → Integrations → Secrets Automation
2. Create a new "1Password Connect Server"
3. Download the `1password-credentials.json` file
4. Create the secret:

```bash
kubectl create namespace external-secrets

kubectl create secret generic op-credentials \
  --from-file=1password-credentials.json=./1password-credentials.json \
  -n external-secrets
```

## 2. Cloudflare API Token

1. Go to Cloudflare dashboard → API Tokens → Create Token
2. Use "Edit zone DNS" template with:
   - Zone:Zone:Read
   - Zone:DNS:Edit (for s33g.uk)
3. Create the secrets:

```bash
kubectl create namespace cert-manager
kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=YOUR_CLOUDFLARE_API_TOKEN \
  -n cert-manager

kubectl create namespace external-dns
kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=YOUR_CLOUDFLARE_API_TOKEN \
  -n external-dns
```

## Verification

```bash
kubectl get secret op-credentials -n external-secrets
kubectl get secret cloudflare-api-token -n cert-manager
kubectl get secret cloudflare-api-token -n external-dns
```
