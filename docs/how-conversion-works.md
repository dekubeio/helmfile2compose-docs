# How the conversion works

Don't misunderstand me. Converting Kubernetes manifests to Docker Compose is an act of heresy. I know it, and [that makes it worse](https://docs.dekube.io/about/#part-i--the-confession).

But you're here, and you want to know what happened to your resources when they went through the tentacle machine. Each section shows what goes in, what comes out, and why it looks like that. For the full architectural deep dive, see the [engine docs](https://docs.dekube.io/understand/architecture/). For what gets lost in translation, see [limitations](limitations.md).

> *"To read the old hymns in the new tongue is not blasphemy — it is archaeology. Every verse that survived the translation reveals what mattered; every verse that was lost reveals what never did."*
>
> — *Necronomicon, On the Archaeology of Manifests (allegedly)*

## Workloads

Deployments, StatefulSets, DaemonSets — they all become Compose services. The distinction between them only matters when you have a scheduler, multiple nodes, and opinions about pod identity. You have none of these.

```yaml
# What goes in
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: myapp
          image: myapp:1.2.0
          ports:
            - containerPort: 8080
          env:
            - name: DATABASE_URL
              value: "postgres://db:5432/myapp"
```

```yaml
# What comes out
services:
  myapp:
    image: myapp:1.2.0
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: "postgres://db:5432/myapp"
```

Replicas are dropped. You're on one machine. Jobs get `restart: on-failure` so they run once and stop. Resource limits (`cpu`, `memory`) are translated to `deploy.resources.limits`. Readiness/liveness probes become `healthcheck` entries (exec, httpGet, tcpSocket). nerdctl ignores both — Docker Compose enforces them.

## ConfigMaps

Two paths depending on how they're consumed.

**As env vars** (`envFrom` or `configMapKeyRef`) — inlined directly into the service. The ConfigMap disappears, its values live in `environment:`.

```yaml
# ConfigMap
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
```

```yaml
# Compose output (if referenced via envFrom)
services:
  myapp:
    environment:
      APP_ENV: "production"
      LOG_LEVEL: "info"
```

**As files** (volume mount) — written to disk and bind-mounted. This is how config files like `nginx.conf` or `application.yaml` typically travel. Both `data` (text) and `binaryData` (base64-encoded binary, e.g. keystores, protobuf) are supported.

Everything is inlined into `compose.yml` rather than using `env_file:` — what you see is what you get.

## Secrets

Same mechanics as ConfigMaps. The interesting case is when a Secret *doesn't exist* in the rendered output — an operator was supposed to create it, or Helm's `lookup` tried to fetch it from a cluster that isn't there. When this happens, a `changeme` placeholder is inserted. Grep your `compose.yml` after conversion.

Your secrets are now plain text in a YAML file. This is what you signed up for. See [limitations — secrets](limitations.md#secrets) for the full existential crisis.

## Services

Kubernetes Services give pods stable DNS names. In Compose there's no kube-proxy, but containers still need to find each other. The solution: network aliases matching the full Kubernetes FQDN hierarchy.

```yaml
# Compose output
services:
  myapp:
    networks:
      default:
        aliases:
          - myapp
          - myapp.default
          - myapp.default.svc
          - myapp.default.svc.cluster.local
```

If a ConfigMap has `DATABASE_HOST=postgres.default.svc.cluster.local`, it still resolves. No rewriting needed. Compose DNS does the rest. This is an intentional design choice with a [long and painful backstory](https://docs.dekube.io/understand/concepts/#the-curse-of-names).

## Ingress

Ingress resources become static reverse proxy rules. Both distributions use **Caddy** as the default reverse proxy (the *ingress provider*), though an [Nginx provider](https://github.com/dekubeio/dekube-provider-nginx) is also available. Controller-specific annotations (HAProxy, Nginx, Traefik) are read by *rewriter* extensions that translate them into provider-agnostic ingress entries — see [rewriter extensions](https://docs.dekube.io/extend/extensions/writing-rewriters/).

`host: myapp.example.com` routing to port 8080 → a reverse proxy rule proxying that hostname to the Compose service. TLS via the provider's CA (Caddy internal CA, certbot, self-signed, or user-provided certs). Controller-specific annotations (rate limiting, redirects, custom headers) are translated when they have a Compose equivalent, skipped otherwise.

Your hostnames need to resolve locally — `*.localhost` works out of the box on most systems, anything else needs `/etc/hosts`.

## PVCs

PVCs become bind mounts under `./data/`. No CSI driver, no dynamic provisioning — just a directory.

```yaml
# Compose output
services:
  myapp:
    volumes:
      - ./data/myapp-data:/var/lib/myapp
```

Paths are customizable in `dekube.yaml`. StatefulSet `volumeClaimTemplates` get the same treatment.

## Init containers

Each init container becomes a separate Compose service with `restart: on-failure`. The main service declares `depends_on` with `condition: service_completed_successfully`, so Docker Compose starts it only after init containers complete. nerdctl ignores `depends_on` — there, everything runs concurrently and converges via retries. See [limitations — startup ordering](limitations.md#startup-ordering) for details.

## Sidecars

Sidecar containers share a network namespace in Kubernetes — they talk over `localhost`. In Compose, they use `network_mode: container:<main>` to achieve the same thing. This mostly works, except when it doesn't — see [limitations — sidecars](https://docs.dekube.io/limitations/#sidecars-and-pod-level-networking).

## CRDs

CRDs are resources that only exist because an operator is watching them. In Compose, there are no operators — so bundled extensions emulate what the operator *would have done*:

- **cert-manager** — generates self-signed certificates as files
- **Keycloak** — converts realm imports into container configuration
- **ServiceMonitor** — skipped silently (no Prometheus to scrape)

Unknown CRDs are skipped with a warning. Need one handled? That's what [third-party extensions](https://docs.dekube.io/extend/extensions/) are for. The engine's [three-tier model](https://docs.dekube.io/understand/concepts/#the-emulation-boundary) explains what can and can't be emulated.

## What gets skipped

HPA, RBAC, NetworkPolicy, PDB, CronJob — anything that only makes sense with a control plane. These are logged and don't affect the output. The [full list](https://docs.dekube.io/limitations/#what-is-ignored) explains the rationale for each.

---

> *"Not all scripture survives the translation. The rites of horizontal scaling, the prayers of admission control, the litanies of role-based access — these were left behind, for the lesser temple has no use for gods it cannot host."*
>
> — *De Vermis Mysteriis, On the Acceptable Losses of Conversion (accounts vary)*
