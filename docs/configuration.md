---
description: "The dekube.yaml reference for helmfile2compose: engine keys plus distribution-specific keys like distribution_version and depends, with a full example."
---

# Configuration reference

All persistent configuration lives in `dekube.yaml`. This file is created on first run and preserved across re-runs. User edits are never overwritten.

This is your territory. The engine converts; the config file is where you tell it what to ignore, what to override, and what to pretend was never there. Think of it as the leash on a machine that has no business existing — short enough to control, long enough to be useful.

## Full example

```yaml
name: my-platform
volume_root: ./data
extensions:
  caddy:
    email: admin@example.com

distribution_version: v3.1.0
depends:
  - keycloak
  - cert-manager==0.3.0
  - trust-manager

volumes:
  data-postgresql:
    driver: local
  myapp-data:
    host_path: app
  other:
    host_path: ./custom

exclude:
  - prometheus-operator
  - meet-celery-*

replacements:
  - old: 'path_style_buckets = false'
    new: 'path_style_buckets = true'

overrides:
  redis-master:
    image: redis:7-alpine
    command: ["redis-server", "--requirepass", "$secret:redis:redis-password"]
    volumes: ["$volume_root/redis:/data"]
    environment: null

services:
  minio-init:
    image: quay.io/minio/mc:latest
    restart: on-failure
    entrypoint: ["/bin/sh", "-c"]
    command:
      - mc alias set local http://minio:9000 $secret:minio:rootUser $secret:minio:rootPassword
        && mc mb --ignore-existing local/my-bucket
```

## Engine keys

These are the controls that let you steer the heresy — what to exclude, what to override, what to pretend doesn't exist. All documented in the engine reference; they work identically in helmfile2compose:

- **[`name`](https://docs.dekube.io/reference/config/#full-schema)** — compose project name (auto-detected on first run)
- **[`volume_root`](https://docs.dekube.io/reference/config/#full-schema)** — base path for PVC bind mounts (default: `./data`)
- **[`volumes`](https://docs.dekube.io/reference/config/#full-schema)** — PVC-to-volume mappings (auto-populated on first run)
- **[`exclude`](https://docs.dekube.io/reference/config/#full-schema)** — workload names to skip (`fnmatch` wildcards)
- **[`replacements`](https://docs.dekube.io/reference/config/#full-schema)** — global find/replace in env vars, ConfigMap files, and proxy upstreams
- **[`overrides`](https://docs.dekube.io/reference/config/#full-schema)** — deep merge into generated services (`null` deletes keys). Every generated `environment` value gets its `$` doubled (`$$`) so compose doesn't interpolate it, but `overrides:` values stay raw — you keep `${VAR}` compose interpolation there, and a `$secret:` reference inside an override still resolves and escapes correctly.
- **[`services`](https://docs.dekube.io/reference/config/#full-schema)** — custom compose services added verbatim
- **[`extensions`](https://docs.dekube.io/reference/config/#per-extension-config-extensions)** — per-extension config (Caddy email/TLS, enable/disable)
- **[`ingress_types`](https://docs.dekube.io/reference/config/#full-schema)** — custom `ingressClassName` → rewriter mapping
- **[`disable_ingress`](https://docs.dekube.io/reference/config/#full-schema)** — skip reverse proxy generation
- **[`network`](https://docs.dekube.io/reference/config/#full-schema)** — external compose network override

!!! note "Upgrading: `$` escaping changed"
    If you were hand-escaping `$$` in chart values or `replacements:` to work around passwords getting mangled by compose interpolation, remove it now — you'd otherwise get `$$$$`. Conversely, a `${VAR}` you meant for compose interpolation but that arrives through chart values or `replacements:` is now taken literally; move it into `overrides:`, which stays raw.

See the **[full engine configuration reference](https://docs.dekube.io/reference/config/)** for detailed descriptions, examples, placeholders (`$secret:`, `$volume_root`), and legacy key migration.

## Distribution-specific keys

These keys are for the package manager, not the engine. The engine doesn't care what version it is; the manager cares so you don't wake up to a breaking change on a Tuesday morning.

### `distribution_version`

Pin the distribution version for dekube-manager.

```yaml
distribution_version: v3.1.0
```

`core_version` is accepted as a backwards-compatible alias.

### `distribution`

Select which distribution dekube-manager installs. Default: `helmfile2compose`. Use `core` for the bare engine (`dekube.py`).

```yaml
distribution: helmfile2compose
```

### `depends`

List of [dekube extensions](https://docs.dekube.io/catalogue/) required by this project. dekube-manager reads this list and installs them automatically.

```yaml
depends:
  - keycloak
  - cert-manager==0.3.0
  - trust-manager
```

Bare names pull the latest release. Pin with `==version` for reproducibility (recommended — see [Your project](getting-started.md#recommended-workflow)).

See [dekube-manager — declarative dependencies](https://manager.dekube.io/docs/#declarative) for override behavior and details.
