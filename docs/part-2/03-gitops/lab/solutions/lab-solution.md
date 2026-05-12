# GitOps Lab Solutions

Solutions are integrated into the lab instructions as the exercises build upon each other. Here are key verification commands and expected outcomes:

## Task 2: Bootstrap Verification

```bash
flux check

# Expected output:
# ✔ Kubernetes 1.28.x >=1.26.0-0
# ✔ Flux 2.2.x installed
# ✔ source-controller: deployment ready
# ✔ kustomize-controller: deployment ready
# ✔ helm-controller: deployment ready
# ✔ notification-controller: deployment ready
```

## Task 3: Application Deployment Verification

```bash
kubectl get deployment podinfo
# NAME      READY   UP-TO-DATE   AVAILABLE   AGE
# podinfo   2/2     2            2           2m

kubectl get pods -l app=podinfo
# NAME                       READY   STATUS    RESTARTS   AGE
# podinfo-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
# podinfo-xxxxxxxxxx-xxxxx   1/1     Running   0          2m

curl localhost:9898
# {
#   "hostname": "podinfo-xxxxxxxxxx-xxxxx",
#   "version": "6.5.3",
#   "runtime": "go1.21.3",
#   "color": "#34577c",
#   "message": "greetings from podinfo"
# }
```

## Task 4: Helm Release Verification

```bash
flux get helmreleases
# NAME     REVISION     SUSPENDED   READY   MESSAGE
# redis    18.x.x       False       True    Release reconciliation succeeded

kubectl get pods -l app.kubernetes.io/name=redis
# NAME              READY   STATUS    RESTARTS   AGE
# redis-master-0    1/1     Running   0          3m

# Test Redis connection
kubectl run redis-client --rm -it --image redis -- redis-cli -h redis-master ping
# PONG
```

## Task 5: Image Automation Verification

```bash
flux get image repository podinfo
# NAME      LAST SCAN                   SUSPENDED   READY   MESSAGE
# podinfo   2024-01-01T10:00:00Z        False       True    successful scan: found X tags

flux get image policy podinfo
# NAME      LATEST IMAGE                               READY   MESSAGE
# podinfo   ghcr.io/stefanprodan/podinfo:6.5.4        True    Latest image tag for '6.x.x'

# Check for automated commit in repository
git log --oneline -n 5
# Shows automated commits from fluxcdbot when images update
```

## Task 6: Suspend/Resume Verification

```bash
# After suspend
flux get kustomizations
# NAME            REVISION        SUSPENDED   READY   MESSAGE
# apps            main@sha1:xxx   True        True    kustomization is suspended...

# After manual scale
kubectl get deployment podinfo
# NAME      READY   UP-TO-DATE   AVAILABLE   AGE
# podinfo   5/5     5            5           10m

# After resume
flux resume kustomization apps
flux get kustomizations
# NAME            REVISION        SUSPENDED   READY   MESSAGE
# apps            main@sha1:xxx   False       True    Applied revision: main@sha1:xxx

kubectl get deployment podinfo
# NAME      READY   UP-TO-DATE   AVAILABLE   AGE
# podinfo   2/2     2            2           11m
# (back to 2 replicas as defined in Git)
```

## Task 7: Notifications

After setup, any deployment will trigger Slack notification:

```json
{
  "text": "Kustomization apps reconciliation succeeded",
  "attachments": [
    {
      "color": "good",
      "title": "apps/default",
      "fields": [
        {
          "title": "Revision",
          "value": "main@sha1:abc123",
          "short": true
        },
        {
          "title": "Status",
          "value": "Applied",
          "short": true
        }
      ]
    }
  ]
}
```

## Common Issues and Solutions

### Issue: Bootstrap fails with authentication error

```bash
# Solution: Ensure GITHUB_TOKEN has correct permissions
# Required scopes: repo, admin:repo_hook

# Regenerate token and retry
export GITHUB_TOKEN=<new-token>
flux bootstrap github --owner=$GITHUB_USER --repository=$GITHUB_REPO --branch=main --path=./clusters/dev-cluster --personal
```

### Issue: Kustomization not reconciling

```bash
# Check Kustomization status
flux get kustomizations apps

# View detailed error
kubectl describe kustomization apps -n flux-system

# Force reconciliation
flux reconcile kustomization apps --with-source

# Check logs
flux logs --kind=Kustomization --name=apps
```

### Issue: HelmRelease stuck in "InstallFailed"

```bash
# Check HelmRelease
flux get helmreleases

# View events
kubectl describe helmrelease redis -n default

# Common issues:
# 1. Chart version not found - update version in HelmRelease
# 2. Values incorrect - validate against chart schema
# 3. Resource conflict - delete existing resources

# Force reconciliation
flux reconcile helmrelease redis
```

### Issue: Image automation not creating commits

```bash
# Check ImageRepository scan
flux get image repository podinfo

# Check ImagePolicy
flux get image policy podinfo

# Verify write access
# ImageUpdateAutomation needs write access to repository

# Check automation status
flux get image update

# View logs
flux logs --kind=ImageUpdateAutomation
```

## Repository Structure Example

Final repository structure:

```text
fleet-infra/
├── .git/
├── README.md
├── clusters/
│   └── dev-cluster/
│       ├── flux-system/
│       │   ├── gotk-components.yaml
│       │   ├── gotk-sync.yaml
│       │   └── kustomization.yaml
│       ├── apps.yaml
│       ├── sources.yaml
│       ├── image-automation.yaml
│       └── notifications.yaml
└── apps/
    ├── base/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── redis-release.yaml
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── production/
        └── kustomization.yaml
```

## Complete Workflow

1. **Developer pushes code** → New container image built
2. **ImageRepository scans** → Detects new version matching policy
3. **ImagePolicy evaluates** → Determines latest version
4. **ImageUpdateAutomation commits** → Updates manifests in Git
5. **GitRepository syncs** → Detects changes in repo
6. **Kustomization reconciles** → Applies updated manifests
7. **Deployment rolls out** → New pods created
8. **Notification sent** → Team alerted via Slack

## Key Benefits Demonstrated

1. **Single Source of Truth** - Git contains desired state
2. **Automated Reconciliation** - Drift corrected automatically
3. **Audit Trail** - Git history shows all changes
4. **Rollback Capability** - Revert Git commits to rollback
5. **Multi-Environment** - Same process for all environments
6. **Automated Updates** - Images updated automatically
7. **Visibility** - Notifications keep team informed

## Next Steps

- Set up production GitOps workflow
- Implement approval process for production
- Add automated testing before deployment
- Configure multi-cluster management
- Integrate with existing CI/CD
