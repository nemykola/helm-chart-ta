## Install & Rollback Commands

### 1. Cluster Bootstrap (cluster-config chart)

This chart creates namespaces, RBAC, quotas/limits and NetworkPolicies.

#### Dev (lp-dev)

```bash
helm upgrade --install cluster-config-dev ./cluster-config \
  -f cluster-config/values-dev.yaml
```
#### Prod (lp-prod)

```bash
helm upgrade --install cluster-config-prod ./cluster-config \
  -f cluster-config/values-prod.yaml
```

### Rollback cluster-config
```bash 
# Show revision history
helm history cluster-config-prod -n lp-prod

# Roll back to a previous revision (example: 2)
helm rollback cluster-config-prod 2 -n lp-prod
```

### 2. Application Umbrella Chart (editor, renderer, assets, publisher)

This chart deploys all application services behind a single Ingress.

#### Dev (lp-dev)

```bash
helm upgrade --install application-dev ./application \
  -f application/values-dev.yaml
```
#### Prod (lp-prod)

```bash
helm upgrade --install application-prod ./application \
  -f application/values-prod.yaml
```

### Rollback cluster-config
```bash 
# Show revision history
helm history application-prod -n lp-prod

# Roll back to a previous revision (example: 2)
helm rollback application-prod 2 -n lp-prod
```


CI/CD for these Helm charts will be discussed during the interview
