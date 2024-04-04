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
sudo snap alias microk8s.helm helm
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

### Initialize new site

When all core services are running, it's time to create sites and their ingress (for access)

#### Deploy site creation job

`<TBU>`

#### Deploy new site ingress

`<TBU>`

```yaml
# Ingress
ingress:
  enabled: true
  annotations:
    # get available classes on your cluster by command
    # kubectl get ingressClass
    kubernetes.io/ingress.class: public
    # kubernetes.io/tls-acme: "true"
    # cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
  - host: erpnext-demo1.getto.dev
    paths:
    - path: /
      pathType: ImplementationSpecific
  tls: []
  #  - secretName: auth-server-tls
  #    hosts:
  #      - auth-server.local
```

```bash
# Check that you have ingress created
kubectl -n ${YOUR_NAMESPACE:-erpnext} get ing
```