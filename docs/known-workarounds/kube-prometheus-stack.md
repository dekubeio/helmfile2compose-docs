---
description: "Converting kube-prometheus-stack to Docker Compose: excluding Grafana's k8s-sidecar, and provisioning dashboards and datasources statically."
---

# kube-prometheus-stack

kube-prometheus-stack is a Helm chart that deploys Prometheus, Grafana, and a constellation of exporters and operators. Most of it converts cleanly via the [servicemonitor extension](https://docs.dekube.io/catalogue/#servicemonitor). Grafana does not.

The problem: Grafana in kube-prometheus-stack ships with **k8s-sidecar** containers that watch ConfigMaps/Secrets via the Kubernetes API at runtime. They provision dashboards and datasources by polling the apiserver for labeled ConfigMaps. This has no compose equivalent — there is no apiserver to poll.

> *The acolyte built a mirror-temple, faithful in every stone — yet the oracles within fell silent, for the gods they consulted resided in a firmament this world had never known. The prayers were correct; the heavens were absent.*
>
> — *Cultes des Goules, On Oracles Without Firmament (take my word for it)*

## What to exclude

The sidecar containers and the admission webhooks serve no purpose in compose:

```yaml
# dekube.yaml
exclude:
  - kube-prometheus-stack-grafana-sidecar-grafana*
  - kube-prometheus-stack-admission*
```

## Grafana override

The original Grafana service uses `kiwigrid/k8s-sidecar` as the main image (the sidecar container is first in the pod spec). Override it with the actual Grafana image and provide env vars directly:

```yaml
# dekube.yaml
overrides:
  kube-prometheus-stack-grafana:
    image: docker.io/grafana/grafana:12.3.1  # adapt to your version
    container_name: null
    environment:
      GF_SECURITY_ADMIN_USER: $secret:grafana-admin-secret:username
      GF_SECURITY_ADMIN_PASSWORD: $secret:grafana-admin-secret:password
      GF_PATHS_DATA: /var/lib/grafana/
      GF_PATHS_LOGS: /var/log/grafana
      GF_PATHS_PLUGINS: /var/lib/grafana/plugins
      GF_PATHS_PROVISIONING: /etc/grafana/provisioning
    volumes:
      - ./configmaps/kube-prometheus-stack-grafana/grafana.ini:/etc/grafana/grafana.ini:ro
      - ./data/kube-prometheus-stack-grafana:/var/lib/grafana
      - /var/lib/grafana-search
      - ./configmaps/kube-prometheus-stack-grafana-dashboards-custom/my-dashboard.json:/var/lib/grafana/dashboards/custom/my-dashboard.json:ro
      - ./configmaps/kube-prometheus-stack-grafana/dashboardproviders.yaml:/etc/grafana/provisioning/dashboards/dashboardproviders.yaml:ro
      - ./configmaps/kube-prometheus-stack-grafana-config-dashboards/provider.yaml:/etc/grafana/provisioning/dashboards/sc-dashboardproviders.yaml:ro
      - ./configmaps/kube-prometheus-stack-grafana/datasources.yaml:/etc/grafana/provisioning/datasources/datasources.yaml:ro
```

Adapt the secret name, dashboard paths, and **Grafana image version** to your setup — the version shown above is a snapshot that will go stale. The `dashboardproviders.yaml`, `datasources.yaml` and `my-dashboard.json` files only exist once you've set the chart values below.

helmfile2compose writes a ConfigMap to `configmaps/<name>/` only when a container mounts it — and into `configmaps/<name>_<hash>/` when that mount uses `items`. The ConfigMaps above are mounted without `items`, so their paths are plain `configmaps/<name>/`. The ones the sidecars read through the API (the chart's datasource and built-in dashboards) are never mounted, so they never reach the disk.

## Datasource provisioning

In K8s, the k8s-sidecar populates `/etc/grafana/provisioning/datasources/` from the labeled `kube-prometheus-stack-grafana-datasource` ConfigMap, fetched through the API. No container mounts it, so it isn't written to disk. Have the Grafana subchart render the datasource into its own ConfigMap instead, in the values of your compose environment:

```yaml
# kube-prometheus-stack values (compose environment)
grafana:
  sidecar:
    datasources:
      enabled: false
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: Prometheus
          type: prometheus
          url: http://kube-prometheus-stack-prometheus.monitoring:9090
          isDefault: true
```

The Grafana container mounts that key, so it lands in `configmaps/kube-prometheus-stack-grafana/datasources.yaml`. Mount it as shown in the override above.

If the datasource references the Prometheus K8s Service by its FQDN (e.g. `kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090`), it resolves natively via network aliases — no replacement needed.

## Dashboard provisioning

Same story: the chart's built-in dashboards are ConfigMaps only the sidecar reads, so they aren't written. Declare the dashboards you want and their provider through the Grafana subchart values. It then mounts them into the Grafana container, and helmfile2compose writes them out:

```yaml
# kube-prometheus-stack values (compose environment)
grafana:
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
        - name: custom
          folder: custom
          type: file
          options:
            path: /var/lib/grafana/dashboards/custom
  dashboards:
    custom:
      my-dashboard:
        json: |
          { "title": "My dashboard" }
```

The provider lands in `configmaps/kube-prometheus-stack-grafana/dashboardproviders.yaml` and each dashboard in `configmaps/kube-prometheus-stack-grafana-dashboards-<provider>/<name>.json`. Mount them as shown in the override above.

## Other components to exclude

kube-prometheus-stack also includes several components that serve no purpose in compose:

- `kube-prometheus-stack-operator` — the Prometheus Operator itself (no CRDs to reconcile in compose)
- `kube-prometheus-stack-kube-state-metrics` — needs the Kubernetes API
- `kube-prometheus-stack-prometheus-node-exporter` — needs host-level access

These are **not** auto-excluded by helmfile2compose — add them manually to your `exclude:` list:

```yaml
# dekube.yaml
exclude:
  - kube-prometheus-stack-grafana-sidecar-grafana*
  - kube-prometheus-stack-admission*
  - kube-prometheus-stack-operator
  - kube-prometheus-stack-kube-state-metrics
  - kube-prometheus-stack-prometheus-node-exporter
```
