# helm-charts

## How to add repository

```shell
helm repo add chia-network https://chia-network.github.io/helm-charts
```

## Prometheus metrics (`charts/generic`)

To expose `/metrics` for Prometheus Operator scrapes, wire **container port → Service port → ServiceMonitor** with a consistent port **name** `metrics`:

1. **`deployment.additionalPorts`** — add a named port, e.g. `- name: metrics`, `containerPort: <listener>`, `protocol: TCP`.
2. **`service.additionalPorts`** — expose the same path with `name: metrics`, `port: <listener>` (or another cluster port), and `targetPort: metrics` (string name matching the container port name).
3. **`serviceMonitor`** — set `enabled: true`. `endpointPort` defaults to `metrics` and must match the **Service** port name (not the container port number).

If the target cluster does not install `monitoring.coreos.com/v1` ServiceMonitors, leave `serviceMonitor.enabled` false and scrape via static config or another mechanism.

## Contour HTTPProxy + cert-manager TLS (`charts/generic`)

For clusters that route with [Contour](https://projectcontour.io/) and issue TLS
with [cert-manager](https://cert-manager.io/), the chart can render a
`projectcontour.io/v1` `HTTPProxy` and a `cert-manager.io/v1` `Certificate`
instead of (or alongside) a plain Ingress. Both require the respective CRDs and
controllers to already be installed in the cluster.

1. **`service.enabled: true`** — the HTTPProxy routes to the chart's Service on
   `service.port`, so a Service must exist.
2. **`contour.httpProxy`** — set `enabled: true` and a `fqdn`. By default a
   single route sends `/` to the Service; override `routes` for custom routing.
   With `tls.enabled: true`, Contour terminates TLS using `tls.secretName`
   (defaults to `<release>-tls`).
3. **`certManager.certificate`** — a cert-manager Certificate. Set `enabled: true`,
   provide `dnsNames`, and point `issuerRef.name` at a configured
   `Issuer`/`ClusterIssuer`. cert-manager writes the issued cert/key into
   `secretName` (defaults to `<release>-tls`), which matches the HTTPProxy/Ingress
   TLS default so the two line up.

The `<release>-tls` default lets a single wildcard `Certificate` back multiple
releases (e.g. production plus review apps) that reuse the same Secret. When a
cert is issued out-of-band, leave `certManager.certificate.enabled` false and
just set the `tls.secretName` on the HTTPProxy/Ingress.
