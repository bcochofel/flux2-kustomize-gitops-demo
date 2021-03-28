# flux2-kustomize-gitops-demo

GitOps demo with Flux2 and Kustomize

## Bootstrap staging cluster

The bootstrap process has some manual steps:

* install `istioctl` binary
* install Istio Operator (using `istioctl` binary)
* install `flux` binary
* bootstrap Flux

After the manual steps the cluster uses GitOps.

### Install `istioctl` binary

check [this](https://istio.io/latest/docs/ops/diagnostic-tools/istioctl/) for more info.

```bash
curl -sL https://istio.io/downloadIstioctl | sh -
sudo cp .istioctl/bin/istioctl /usr/local/bin
```

### Install Istio Operator

check [this](https://istio.io/latest/docs/setup/install/operator/) for more info.

```bash
istioctl operator init
```

### Install `flux` binary

```bash
curl -s https://toolkit.fluxcd.io/install.sh | sudo bash
```

### `flux` bootstrap

```bash
export GITHUB_TOKEN=<your token>
export GITHUB_USER=<your username>
export GITHUB_REPO=<your repository>
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

## References

* [Istio Operator](https://istio.io/latest/docs/setup/install/operator/)
* [Flux GitOps Toolkit](https://toolkit.fluxcd.io/)
* [Istio / Prometheus](https://istio.io/latest/docs/ops/integrations/prometheus/)
* [Envoy Access Logs](https://istio.io/latest/docs/tasks/observability/logs/access-log/)
