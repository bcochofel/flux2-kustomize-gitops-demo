# flux2-kustomize-gitops-demo

GitOps demo with Flux2 and Kustomize

## Dependencies

Install gnupg and SOPS.

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

**NOTE:** If you have any previously created secret for `sops` you should apply it now.

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

### Mozilla SOPS

Check [this](https://toolkit.fluxcd.io/guides/mozilla-sops/) to create the GPG key and the `sops-gpg` secret.

After creating you can encrypt secrets, on the `sops-secrets` folder using the pub key.

## Observability and Istio Mesh

For now the observability namespace is outside Istio Mesh since there are some
issues like:

* alermanager service monitor not showing
* thanos querier without stores
* prometheus operator jobs don't complete because sidecar doesn't exit [(check here)](https://medium.com/redbox-techblog/handling-istio-sidecars-in-kubernetes-jobs-c392661c4af7)

To put everything on the mesh uncomment the lines from:

* observability/staging/namespace.yaml
* observability/staging/kube-prometheus-stack-values.yaml

More info [here](https://istio.io/latest/docs/ops/integrations/prometheus/)

### Workarounds

Patch AdmissionWebhooks for Prometheus Operator are Job:, and since Jobs don't
finish because of istio-proxy we can add the following annotations:

```yaml
  values:
    prometheusOperator:
      admissionWebhooks:
        patch:
          podAnnotations:
            sidecar.istio.io/inject: "false"
```

To get Thanos Query DNS Stores working we need to add listenLocal on Prometheus:

```yaml
  values:
    prometheus:
      prometheusSpec:
        listenLocal: true
        thanos:
          baseImage: quay.io/thanos/thanos
          version: v0.19.0
          listenLocal: true
```

To scrape alertmanager add listenLocal:

```yaml
  values:
    alertmanager:
      alertmanagerSpec:
        listenLocal: true
```

you can use mTLS:

```yaml
  values:
    alertmanager:
      serviceMonitor:
        scheme: "https"
        tlsConfig:
          caFile: /etc/prom-certs/root-cert.pem
          certFile: /etc/prom-certs/cert-chain.pem
          keyFile: /etc/prom-certs/key.pem
          insecureSkipVerify: true
```

## Create AlertManager Config Secret

To create AlertManager configuration secret create a YAML file (`/tmp/alertmanager.yaml`) with the contents:

```yaml
alertmanager:
  config:
    global:
      slack_api_url: '<slack_webhook_url>'
      resolve_timeout: 5m
    route:
      group_by: ['job']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'slack'
      routes:
      - match:
          alertname: Watchdog
        receiver: 'null'
    receivers:
    - name: 'null'
    - name: 'slack'
      slack_configs:
      - channel: '#notifications'
    templates:
    - '/etc/alertmanager/config/*.tmpl'
```

**Note:** Replace <slack_webhook_url> with the Slack URL

then create the secret (on the `sops-secrets` folder):

```bash
kubectl -n observability create secret generic alertmanager \
  --from-file=values.yaml=/tmp/alertmanager.yaml \
  --dry-run=client -o yaml > alertmanager.yaml
```

and finally encrypt the secret:

```bash
sops --encrypt --in-place alertmanager.yaml
```

## Connecting to Virtual Services

To check the External IP for the Istio Ingress Gateway use:

```bash
kubectl get svc istio-ingressgateway -n istio-system
```

After checking the IP you need to add some entries on your hosts file.

Example using IP 192.168.77.105 (from the MetalLB Production pool):

```bash
192.168.77.105 prometheus.demo.lab
192.168.77.105 thanos.demo.lab
192.168.77.105 grafana.demo.lab
192.168.77.105 alertmanager.demo.lab
192.168.77.105 tracing.demo.lab
192.168.77.105 bookinfo.demo.lab
```

You can now connect to the Web interface using those addresses.

**NOTE:** Since the TLS certificates are self-signed your browser will complaint.

## References

* [Mozilla SOPS](https://toolkit.fluxcd.io/guides/mozilla-sops/#gitops-workflow)
* [Istio Operator](https://istio.io/latest/docs/setup/install/operator/)
* [Flux GitOps Toolkit](https://toolkit.fluxcd.io/)
* [Istio / Prometheus](https://istio.io/latest/docs/ops/integrations/prometheus/)
* [Envoy Access Logs](https://istio.io/latest/docs/tasks/observability/logs/access-log/)
* [Prometheus Community charts](https://github.com/prometheus-community/helm-charts)
* [Grafana Community charts](https://github.com/grafana/helm-charts)
* [Bitnami charts](https://github.com/bitnami/charts)
