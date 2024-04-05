# Install erpnext on Microk8s for local development

## Prerequisites

This guideline is written with machine with these specs:

- CPU: 4 vCPU
- RAM: 16 GB
- SSD: 100 GB
- OS: Ubuntu 22.04 Server (minimal setup)
- Logged in account has `sudo` privilege

The VM should be in clean state to avoid conflict with any other packages.

## Install Microk8s

```bash
# Update your OS
sudo apt update
sudo apt dist-upgrade -y

# Install microk8s
sudo snap install microk8s --classic

# Activate microk8s addons
microk8s enable dns storage ingress helm3
```

For easier usage, you should alias microk8s's commands

```bash
sudo snap alias microk8s.kubectl kubectl
sudo snap alias microk8s.helm3 helm
```

## Install ErpNext

### Prepare Helm repo and version

Firstly, add erpnext to your local helm repo.

```bash
helm repo add frappe https://helm.erpnext.com
helm repo update
```

Secondly, we can check the version of erpnext's chart

```bash
helm search repo erpnext --versions
```

In this guide, I picked the latest chart version at this time: `7.0.53`.
Let's check the available values of the chart by command:

```bash
helm show values frappe/erpnext --version ${ERPNEXT_VERSION:-7.0.53}
```

### Prepare deployment values

For the simple development environment, we will just override some values, so let's create a `values.yaml` file with this content

```yaml
# Storage
persistence:
  worker:
    enabled: true
    size: 8Gi
    # get available classes on your cluster by command
    # kubectl get sc
    storageClass: "microk8s-hostpath"
```

For production, you should take care `autoscaling` and `resources` to ensure your cluster can handle your workload.

### Install core services and check the status

```bash
#install the chart
helm upgrade --install --create-namespace \
  ${YOUR_APP_NAME:-erp1} frappe/erpnext --version ${ERPNEXT_VERSION:-7.0.53} \
  -n ${YOUR_NAMESPACE:-erpnext} \
  -f values.yaml
```

Watch pod creation status, all your pod should be `Running`, except conf-bench should be `Completed`

```bash
kubectl -n ${YOUR_NAMESPACE:-erpnext}  get pod -w
```

### Deploy new site

When all core services are running, it's time to create sites and their ingress (for access)

Prepare values for creating new site, named it `create-new-site.yaml`

```yaml
persistence:
  worker:
    storageClass: microk8s-hostpath

# Site creation job
jobs:
  createSite:
    enabled: true
    siteName: "erpnext-demo1.getto.dev"
    adminPassword: "YourSuperScretPassword"

# New site ingress
ingress:
  enabled: true
  ingressName: "erpnext-demo1"
  annotations:
    # Get your cluster ingress class by: kubectl get ingressClass
    kubernetes.io/ingress.class: public
    ## This is for auto request LetsEncrypt certificate if you have a configured certmanager
    #cert-manager.io/cluster-issuer: letsencrypt
    #kubernetes.io/tls-acme: "true"
  hosts:
  - host: erpnext-demo1.getto.dev
    paths:
    - path: /
      pathType: ImplementationSpecific
  tls: []
  ## If you want to serve your site over HTTPS
  #  - secretName: erpnext-demo1-tls
  #    hosts:
  #      - erpnext-demo1.getto.dev
```

Generate the needed template for the job

```bash
helm template ${YOUR_APP_NAME:-erp1} -n ${YOUR_NAMESPACE:-erpnext} \
  frappe/erpnext --version ${ERPNEXT_VERSION:-7.0.53} \
  -s templates/job-create-site.yaml -s templates/ingress.yaml \
  -f create-new-site.yaml > to-apply.yaml
```

Review the output file `to-apply.yaml` if you need to adjust anything before applying

```bash
kubectl apply -n ${YOUR_NAMESPACE:-erpnext} -f to-apply.yaml
```

Watch the creation job complete by

```bash
kubectl -n ${YOUR_NAMESPACE:-erpnext}  get pod -w
```

Check the logs of the job if error happen
```bash
# Find your new-site job by: 
kubectl -n ${YOUR_NAMESPACE:-erpnext} get job

# Then set the JOB_NAME and see its log
kubectl -n ${YOUR_NAMESPACE:-erpnext} logs job/${JOB_NAME:-erp1-erpnext-new-site-20240404050628}
```

Check that you have ingress created

```bash
kubectl -n ${YOUR_NAMESPACE:-erpnext} get ing
```

Your site should be accessible via the configured domain. To add more site (with other domain), just repeat this step.
