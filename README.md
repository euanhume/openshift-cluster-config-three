# OpenShift GitOps Cluster Configuration

This repository contains the GitOps configuration for managing OpenShift clusters using ArgoCD and ApplicationSets.

## Architecture

```
ArgoCD Web UI
├── operators-bootstrap (Application)
│   └── operators (ApplicationSet)
│       ├── cert-manager-operator (Application)
│       └── compliance-operator (Application)
```

## Prerequisites

- OpenShift cluster with admin access
- `oc` CLI configured and authenticated
- Git repository access (update repo URLs to match your fork)

## Fresh Installation Steps

### 1. Bootstrap GitOps Operator & ArgoCD

Deploy the OpenShift GitOps operator and ArgoCD instance:

```bash
# Deploy the GitOps operator and ArgoCD instance
# This includes retry logic for timing issues
until oc apply -k bootstrap/; do sleep 2; done
```

This creates:
- `openshift-gitops-operator` namespace with the GitOps operator
- `openshift-gitops` namespace with ArgoCD instance
- ArgoCD configured with OpenShift OAuth integration
- RBAC configured for `cluster-admins` group

### 2. Wait for ArgoCD to be Ready

Wait for all ArgoCD components to be running:

```bash
# Check ArgoCD pods are ready
oc get pods -n openshift-gitops

# Get ArgoCD URL
echo "ArgoCD URL: https://$(oc get route -n openshift-gitops openshift-gitops-server -o jsonpath='{.spec.host}')"
```

### 3. Deploy the Bootstrap Application

Deploy the bootstrap Application that manages the ApplicationSet:

```bash
# Deploy the AppProject first
oc apply -f apps/operators/appproject.yaml

# Deploy the bootstrap Application
oc apply -f apps/bootstrap-app.yaml
```

### 4. Verify Deployment

Check that everything is working:

```bash
# Check all Applications
oc get applications -n openshift-gitops

# Check ApplicationSets
oc get applicationsets -n openshift-gitops

# Check operator deployments
oc get subscriptions -n cert-manager-operator
oc get subscriptions -n openshift-compliance
```

## What Gets Created

After successful deployment, you'll have:

- **ArgoCD Web UI** with OpenShift OAuth login
- **operators-bootstrap** Application managing the ApplicationSet
- **operators** ApplicationSet dynamically discovering operator directories
- **cert-manager-operator** Application managing cert-manager
- **compliance-operator** Application managing compliance operator

## Adding New Operators

To add a new operator:

1. Create a new directory under `operators/`:
   ```bash
   mkdir operators/my-new-operator
   ```

2. Copy the template files:
   ```bash
   cp -r operators/cert-manager-operator/* operators/my-new-operator/
   ```

3. Update the files with your operator details:
   - `namespace.yaml` - Change namespace name
   - `operatorgroup.yaml` - Change name and namespace
   - `subscription.yaml` - Change name, namespace, and channel

4. Commit and push:
   ```bash
   git add operators/my-new-operator/
   git commit -m "Add my-new-operator"
   git push
   ```

The ApplicationSet will automatically discover and deploy the new operator!

## Accessing ArgoCD

1. **Get the URL:**
   ```bash
   oc get route -n openshift-gitops openshift-gitops-server -o jsonpath='{.spec.host}'
   ```

2. **Login Options:**
   - **OpenShift OAuth:** Click "LOG IN VIA OPENSHIFT" (recommended)
   - **Admin User:** Use admin credentials if needed

3. **Users in the `cluster-admins` group get full admin access**

## Repository Structure

```
.
├── bootstrap/                 # GitOps operator + ArgoCD setup
│   ├── argocd/
│   │   └── argocd.yaml       # ArgoCD instance configuration
│   ├── kustomization.yaml    # Bootstrap resources
│   ├── namespace.yaml        # Operator namespace
│   ├── operatorgroup.yaml    # Operator group
│   └── subscription.yaml     # GitOps operator subscription
├── apps/
│   ├── operators/
│   │   ├── appproject.yaml   # Operators AppProject
│   │   ├── applicationset.yaml # ApplicationSet for operators
│   │   └── kustomization.yaml
│   └── bootstrap-app.yaml    # Bootstrap Application
└── operators/                # Operator configurations
    ├── cert-manager-operator/
    └── compliance-operator/
```

## Troubleshooting

### ArgoCD Login Issues
- Ensure you're in the `cluster-admins` group
- Check SSO status: `oc get argocd -n openshift-gitops openshift-gitops -o jsonpath='{.status.sso}'`

### ApplicationSet Not Creating Apps
- Verify the ApplicationSet exists: `oc get applicationsets -n openshift-gitops`
- Check ApplicationSet events: `oc describe applicationset -n openshift-gitops operators`

### Operator Installation Issues
- Check subscription status: `oc get subscriptions -A`
- Verify operator pod status: `oc get pods -n <operator-namespace>`

## Customization

### Update Repository URL
Update the `repoURL` in these files to match your fork:
- `apps/operators/applicationset.yaml`
- `apps/operators/appproject.yaml`
- `apps/bootstrap-app.yaml`

### Modify RBAC
Edit `bootstrap/argocd/argocd.yaml` to change:
- Default policy (`role:readonly`)
- Group mappings (`cluster-admins`)
- Scope settings