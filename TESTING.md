# Testing Guide

## Architecture Note

This deployment uses **Ingress** (nginx), NOT Gateway API. The site is exposed via:
- Ingress resource with `className: nginx`
- nginx ingress controller handles routing

## Testing Substantial Changes

After making substantial changes to the Helm chart or workflow, follow this sequence:

```bash
# 1. Push changes to trigger CI/CD
git push

# 2. Check GitHub Actions workflow status
gh run list --limit 5

# 3. Verify pods are running in the cluster
kubectl -n hikma get pods
```

## Useful Commands

```bash
# Watch pod status
kubectl -n hikma get pods -w

# View deployment logs
kubectl -n hikma logs -l app.kubernetes.io/name=hikma -f

# Check events
kubectl -n hikma get events --sort-by='.lastTimestamp' | tail -20

# Describe pod (for troubleshooting)
kubectl -n hikma describe pod -l app.kubernetes.io/name=hikma
```
