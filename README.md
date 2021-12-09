# Cloud Devops Flux Project

- [Customization Flux Manifests](https://fluxcd.io/docs/installation/#customize-flux-manifests)
- [Pod Security Policy](https://fluxcd.io/docs/installation/#pod-security-policy)
- [Automate image updates to Git](https://fluxcd.io/docs/guides/image-update/)

## Flux commands

flux reconcile source git flux-system
flux check

### Create Registry file 

```shell
flux create image repository fists-kotlin-cicd-app \ 
  --image=registry.digitalocean.com/sa-container-registry-dev/fists-kotlin-cicd-app \ 
  --interval=1m \
  --export > ./clusters/dev/first-kotlin-cicd-app-registry.yaml
```

### Create Image Policy

```shell
flux create image policy fists-kotlin-cicd-app \
  --image-ref=fists-kotlin-cicd-app \
  --select-semver=5.0.x \
  --export > ./clusters/dev/apps/first-kotlin-cicd-app-policy.yaml
```

### Create Image Update

```shell
flux create image update flux-system \
    --git-repo-ref=flux-system \
    --git-repo-path="./clusters/dev/flux-system" \
    --checkout-branch=main \
    --push-branch=main \
    --author-name=fluxcdbot \
    --author-email=fluxcdbot@users.noreply.github.com \
    --commit-template="{{range .Updated.Images}}{{println .}}{{end}}" \
    --export > ./clusters/dev/flux-system/flux-system-automation.yaml
```

### Consult

flux get images all --all-namespaces

## kubectl 

kubectl port-forward pod/first-kotlin-cicd-app-7c9978b4bf-6n9zq 8888:8080


## Logs do Ingress

kubectl logs -f service/ingress-nginx-controller -n ingress-nginx

## Install sealed

```shell
flux create source helm sealed-secrets \
  --url="https://bitnami-labs.github.io/sealed-secrets" \
  --interval="10m" \
  --export > ./clusters/dev/helm/repositories/sealed-secrets.yaml
```

```shell
flux create helmrelease "sealed-secrets-controller" \
  --release-name="sealed-secrets-controller" \
  --source="HelmRepository/sealed-secrets" \
  --chart="sealed-secrets" \
  --chart-version "$SEALED_SECRETS_CHART_VERSION" \
  --values="sealed-secrets-values-v1.16.1.yaml" \
  --target-namespace="flux-system" \
  --crds=CreateReplace \
  --export > ./clusters/dev/helm/releases/sealed-secrets-v1.16.1.yaml
```

After completing the above steps, `Flux CD` will start your `DOKS` cluster `reconciliation` (in about `one minute` or so, if using the `default` interval). If you don't want to wait, you can always `force` reconciliation via:

```shell
flux reconcile source git flux-system
```

After a few moments, please inspect the Flux CD `Sealed Secrets` Helm release:

```shell
flux get helmrelease sealed-secrets-controller
```

The output looks similar to:

```text
NAME                        READY   MESSAGE                                 REVISION        SUSPENDED 
sealed-secrets-controller   True    Release reconciliation succeeded        1.16.1          False 
```

Look for the `READY` column value - it should say `True`. Reconciliation status is displayed in the `MESSAGE` column, along with the `REVISION` number, which represents the `Helm` chart `version`. Please bear in mind that some releases take longer to complete (like `Prometheus` stack, for example), so please be patient.

**Hints:**

- The `MESSAGE` column will display `Reconciliation in progress`, as long as the `HelmController` is performing the installation for the specified `Helm` chart. If something goes wrong, you'll get another message stating the reason, so please make sure to check Helm release state.
- You can use the `--watch` flag for example: `flux get helmrelease <name> --wait`, to wait until the command finishes. Please bear in mind that in this mode, `Flux` will block your terminal prompt until the default timeout of `5 minutes` occurs (can be overridden via the `--timeout` flag).
- In case something goes wrong, you can search the `Flux` logs, and filter `HelmRelease` messages only:

    ```shell
    flux logs --kind=HelmRelease
    ```

### Exporting the Sealed Secrets Controller Public Key

To be able to `encrypt` secrets, you need the `public key` that was generated by the `Sealed Secrets Controller` when it was deployed by `Flux CD` in your `DOKS` cluster.

First, change directory where you cloned your `Flux CD` Git repository, and do the following (please replace the `<>` placeholders accordingly):

```shell
kubeseal --controller-namespace=flux-system --fetch-cert > pub-sealed-secrets-<YOUR_DOKS_CLUSTER_NAME_HERE>.pem
```

**Note:**

If for some reason the `kubeseal` certificate fetch command hangs, you can use the following steps to work around this issue:

- First, open another terminal window, and `expose` the `Sealed Secrets Controller` service on your `localhost` (you can use `CTRL - C` to terminate, after fetching the public key):

  ```shell
  kubectl port-forward service/sealed-secrets-controller 8080:8080 -n flux-system &
  ```

- Then, you can go back to your working terminal and fetch the public key (please replace the `<>` placeholders accordingly):

  ```shell
  curl --retry 5 --retry-connrefused localhost:8080/v1/cert.pem > pub-sealed-secrets-<YOUR_DOKS_CLUSTER_NAME_HERE>.pem
  ```

## Creating the Prometheus Stack Helm Release

In this step, you will use pre-made manifests to create the `Prometheus` Helm release for `Flux CD`. Then, `Flux` will trigger the `Prometheus` installation process for your `DOKS` cluster. The `Prometheus` stack deploys `Grafana` as well, so you need to set the `administrator` credentials for accessing the `dashboards`. You will learn how to use `kubeseal` CLI with `Sealed Secrets Controller` to encrypt `sensitive` data stored in `Kubernetes Secrets`. Then, you will see how the Flux CD `HelmRelease` manifest is used to `reference` Grafana `credentials` stored in the `Kubernetes Secret`.


```shell
kubectl create secret generic "prometheus-stack-credentials" \
    --namespace flux-system \
    --from-literal=grafana_admin_password=YWRtaW5TQS1MZXZpcwo= \
    --dry-run=client -o yaml | kubeseal --cert=pub-sealed-secrets-sa-cluster-dev.pem \
    --format=yaml > ./clusters/dev/helm/secrets/prometheus-stack-credentials-sealed.yaml
```