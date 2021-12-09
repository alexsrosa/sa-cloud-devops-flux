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