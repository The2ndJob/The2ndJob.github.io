# Module installation

This guide is the next topic in the series of setting up ErpNext. See the previous post for the [installation](./simple-installation-microk8s). Please ensure your installation is up and running before going with following guide.

## HR Module

The module installation should be run by a custom job.
Create file `install-hrms-job.yaml` with following content

```yaml
persistence:
  worker:
    storageClass: microk8s-hostpath

# Site creation job
jobs:
  custom:
    enabled: true
    jobName: "install-hrms"
    backoffLimit: 1
    containers:
      - name: install-hrms
        image: "frappe/erpnext:v15.19.0"
        imagePullPolicy: IfNotPresent
        command: ["bash", "-c"]
        args:
          - >
            bench get-app hrms --branch v15.16.0
            ;bench --site ${SITE_NAME} install-app hrms
        env:
          - name: "SITE_NAME"
            value: "erpnext-demo1.getto.dev"
        securityContext:
          capabilities:
            add:
            - CAP_CHOWN
        volumeMounts:
          - name: sites-dir
            mountPath: /home/frappe/frappe-bench/sites
          - name: logs
            mountPath: /home/frappe/frappe-bench/logs
    restartPolicy: Never
    volumes: 
      - name: sites-dir
        persistentVolumeClaim:
          claimName: erp1-erpnext
          readOnly: false
      - name: logs
        emptyDir: {}
```

Remember to update these fields:

- `image`: Pick the image version that comes with your helm chart. In this guide, it's `v15.19.0`, which comes in chart version `7.0.53`
- `--branch` in `args`: You should pick a released tag on [HRMS repo](https://github.com/frappe/hrms/releases), at the time of writing, it's `v15.16.0`.

Then generate the manifest

```bash
helm template ${YOUR_APP_NAME:-erp1} -n ${YOUR_NAMESPACE:-erpnext} \
  frappe/erpnext --version ${ERPNEXT_VERSION:-7.0.53} \
  -s templates/job-custom.yaml \
  -f install-hrms-job.yaml > to-install-hrms.yaml
```

Review the content of file `to-install-hrms.yaml` before apply it

```bash
kubectl apply -n ${YOUR_NAMESPACE:-erpnext} -f to-install-hrms.yaml
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
kubectl -n ${YOUR_NAMESPACE:-erpnext} logs job/${JOB_NAME:-install-hrms}
```

## CRM module

`<TBU>`

## Education module

`<TBU>`


## Education module

`<TBU>`

## Health module

`<TBU>`
