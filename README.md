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

## kubectl 

kubectl port-forward pod/first-kotlin-cicd-app-7c9978b4bf-6n9zq 8888:8080


## Logs do Ingress

kubectl logs -f service/ingress-nginx-controller -n ingress-nginx