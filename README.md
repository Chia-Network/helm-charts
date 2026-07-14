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

1. **`service.enabled: true`** — the default backend routes to the chart's
   Service on `service.port`, so a Service must exist. This is only required when
   using the default route/tcpproxy; if you provide custom `routes` or `tcpproxy`
   that reference other Services, `service.enabled` is not needed.
2. **`contour.httpProxy`** — set `enabled: true` and a `fqdn`. By default a
   single route sends `/` to the Service; override `routes` for custom routing.
   With `tls.enabled: true`, Contour terminates TLS using `tls.secretName`
   (defaults to `<release>-tls`). Set `tls.passthrough: true` to forward
   encrypted traffic to the backend instead — Contour then routes at L4, so the
   chart emits a default `tcpproxy` to the Service (override via `tcpproxy`).
3. **`certificates`** — a list of cert-manager Certificates. Each entry passes its
   `spec` through to the resource as-is (any cert-manager field is supported), and
   `name` defaults to the chart fullname (with an index suffix if there are
   several). Point a cert's `spec.secretName` at the HTTPProxy/Ingress TLS secret
   (which defaults to `<release>-tls`) to wire the two together. When defining more
   than one certificate, set an explicit `name` on each — the `<fullname>-<index>`
   fallback is position-sensitive, so reordering the list would rename (and
   therefore reissue) a Certificate.

Using `<release>-tls` as the shared secret lets a single wildcard `Certificate`
back multiple releases (e.g. production plus review apps) that reuse the same
Secret. When a cert is issued out-of-band, leave `certificates` empty and just
set `tls.secretName` on the HTTPProxy/Ingress.

**Adopting an existing Certificate:** if a `Certificate` of the same name was
previously created outside Helm (e.g. via `kubectl apply`), `helm upgrade` fails
with an ownership error. Either delete it first (the TLS Secret survives, so
there's no reissue) or annotate/label it (`meta.helm.sh/release-name`,
`meta.helm.sh/release-namespace`, `app.kubernetes.io/managed-by=Helm`) so Helm
adopts it in place.
