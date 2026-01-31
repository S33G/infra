# GitHub Repository Setup for ArgoCD

Complete guide for setting up a private GitHub repository with SSH authentication for ArgoCD GitOps.

---

## Step 1: Create GitHub Repository

1. Go to https://github.com/new
2. Repository name: `infra`
3. Visibility: **Private**
4. Don't initialize with anything
5. Click "Create repository"

---

## Step 2: Generate SSH Key for ArgoCD

```bash
# Generate SSH key for ArgoCD
ssh-keygen -t ed25519 -C "argocd@k3s-homelab" -f ~/.ssh/argocd_github -N ""

# Display the public key (copy this)
cat ~/.ssh/argocd_github.pub
```

**Save the output** - you'll need to add this public key to GitHub.

---

## Step 3: Add SSH Key to GitHub

1. Copy the public key output from the command above
2. Go to https://github.com/settings/ssh/new
3. Title: `ArgoCD K3s Homelab`
4. Key: Paste the public key
5. Click "Add SSH key"

---

## Step 4: Configure Git Remote and Push

```bash
cd /home/seeg/infra

# Add the remote
git remote add origin git@github.com:s33g/infra.git

# Verify remote was added
git remote -v

# Check current branch name
git branch

# Push to GitHub (use your actual branch name)
git push -u origin main
# Or if your branch is 'master':
# git push -u origin master
```

---

## Step 5: Update ArgoCD Root App

```bash
cd /home/seeg/infra/kube

# Update root-app.yaml with SSH URL
sed -i 's|https://github.com/seeg/infra.git|git@github.com:s33g/infra.git|g' bootstrap/argocd/root-app.yaml

# Verify the change
grep "repoURL:" bootstrap/argocd/root-app.yaml

# Should show: repoURL: git@github.com:s33g/infra.git

# Commit and push
git add bootstrap/argocd/root-app.yaml
git commit -m "fix: update repository URL to SSH format"
git push
```

---

## Step 6: Add Repository Credentials to ArgoCD

```bash
# Create repository secret with SSH key (using file)
kubectl create secret generic repo-infra-ssh \
  --from-literal=type=git \
  --from-literal=url=git@github.com:s33g/infra.git \
  --from-file=sshPrivateKey=$HOME/.ssh/argocd_github \
  -n argocd

# OR pass raw key directly (read from file):
kubectl create secret generic repo-infra-ssh \
  --from-literal=type=git \
  --from-literal=url=git@github.com:s33g/infra.git \
  --from-literal=sshPrivateKey="$(cat $HOME/.ssh/argocd_github)" \
  -n argocd

# Label it so ArgoCD recognizes it
kubectl label secret repo-infra-ssh argocd.argoproj.io/secret-type=repository -n argocd

# Verify secret was created
kubectl get secret repo-infra-ssh -n argocd
```

**Expected output**: Should show the secret exists with 3 data fields (type, url, sshPrivateKey).

---

## Step 7: Reapply Root App and Watch Sync

```bash
# Reapply the root application
kubectl apply -f bootstrap/argocd/root-app.yaml

# Watch applications sync
kubectl get applications -n argocd -w

# Press Ctrl+C to stop watching

# Check application status
kubectl get applications -n argocd
```

**Expected status**: 
- `root` application should show `Synced` and `Healthy`
- Child applications should start appearing (metallb, longhorn, cert-manager, etc.)

---

## Verification

### Check Repository Connection

```bash
# Via ArgoCD CLI (if installed)
argocd repo list

# Via kubectl
kubectl get secret -n argocd | grep repo
```

### Check Application Sync Status

```bash
# List all applications
kubectl get applications -n argocd

# Get detailed status
kubectl describe application root -n argocd

# Check if child applications are created
kubectl get applications -n argocd | grep -E "metallb|longhorn|cert-manager"
```

### Check ArgoCD Logs (if issues)

```bash
# Application controller logs
kubectl logs -n argocd deployment/argocd-application-controller --tail=50

# Repo server logs
kubectl logs -n argocd deployment/argocd-repo-server --tail=50
```

---

## Troubleshooting

### SSH Key Permission Issues

```bash
# Ensure correct permissions on SSH key
chmod 600 ~/.ssh/argocd_github
chmod 644 ~/.ssh/argocd_github.pub
```

### "Repository not found" Error

```bash
# Verify SSH key was added to GitHub
ssh -T git@github.com

# Should show: "Hi s33g! You've successfully authenticated..."

# If it fails, re-add the public key to GitHub
cat ~/.ssh/argocd_github.pub
```

### Root App Shows "ComparisonError"

```bash
# Check if repository secret exists
kubectl get secret repo-infra-ssh -n argocd -o yaml

# Verify the URL in the secret matches root-app.yaml
kubectl get secret repo-infra-ssh -n argocd -o jsonpath='{.data.url}' | base64 -d

# Delete and recreate the root app
kubectl delete application root -n argocd
kubectl apply -f bootstrap/argocd/root-app.yaml
```

### Child Applications Not Created

```bash
# Check root app status
kubectl describe application root -n argocd

# Manually trigger refresh
kubectl patch application root -n argocd \
  --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```

---

## Post-Setup

Once ArgoCD successfully syncs:

1. **Access ArgoCD UI**: 
   ```bash
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   ```
   Open: https://localhost:8080
   - Username: `admin`
   - Password: (from setup guide)

2. **Monitor Deployment**:
   ```bash
   # Watch all pods across namespaces
   watch kubectl get pods -A
   
   # Watch applications sync
   watch kubectl get applications -n argocd
   ```

3. **Verify Infrastructure Components**:
   ```bash
   # MetalLB
   kubectl get ipaddresspool -n metallb-system
   
   # Longhorn
   kubectl get pods -n longhorn-system
   
   # cert-manager
   kubectl get clusterissuer
   
   # External Secrets
   kubectl get clustersecretstore
   ```

---

## Security Notes

- **Private Key**: The ArgoCD SSH private key is stored in Kubernetes secret `repo-infra-ssh`
- **Access**: Only ArgoCD has access to this key within the cluster
- **GitHub Token Alternative**: If you prefer HTTPS, you can use a GitHub Personal Access Token instead
- **Key Rotation**: Regenerate SSH keys periodically and update both GitHub and ArgoCD

---

## Alternative: HTTPS with Personal Access Token

If you prefer HTTPS instead of SSH:

```bash
# Create GitHub Personal Access Token:
# GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
# Generate new token with 'repo' scope

# Create repository secret
kubectl create secret generic repo-infra-https \
  --from-literal=type=git \
  --from-literal=url=https://github.com/s33g/infra.git \
  --from-literal=username=s33g \
  --from-literal=password=YOUR_GITHUB_TOKEN \
  -n argocd

kubectl label secret repo-infra-https argocd.argoproj.io/secret-type=repository -n argocd

# Update root-app.yaml to use HTTPS URL
sed -i 's|git@github.com:s33g/infra.git|https://github.com/s33g/infra.git|g' \
  bootstrap/argocd/root-app.yaml
```

---

## Summary

✅ GitHub repository created (private)  
✅ SSH key generated and added to GitHub  
✅ Git remote configured  
✅ Code pushed to GitHub  
✅ ArgoCD repository credentials configured  
✅ Root application syncing  
✅ Infrastructure components deploying  

**Next**: Wait for all applications to sync, then verify services are accessible via ingress routes.
