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
