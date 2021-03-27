# flux2-kustomize-gitops-demo

GitOps demo with Flux2 and Kustomize

## Install Flux

```bash
curl -s https://toolkit.fluxcd.io/install.sh | sudo bash
```

## Bootstrap staging

```bash
export GITHUB_TOKEN=<your token>
export GITHUB_USER=<your username>
export GITHUB_REPO=flux2-kustomize-gitops-demo
```

pre-flight check

```bash
flux check --pre
```

bootstrap cluster

```bash
flux bootstrap github \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/staging
```

watch Helm releases installation

```bash
watch flux get helmreleases --all-namespaces
```

watch flux reconciliation

```bash
watch flux get kustomizations
```
