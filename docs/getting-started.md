# Using helmfile2compose with your own project

If you need to use this tool, I am truly sorry.

You're here because someone asked for a Docker Compose. A colleague, a client, a mass of uninitiated who do not care that Kubernetes exists for a reason. You have explained. They do not care. They want `docker compose up` and they want it yesterday.

I've been there. The first time, I ignored the request. The second time, I built a tool. The third time, I went back to my own platform, said "why not" — and the tool stopped being a tool. At least you're warned about what it does, which is more than most ecosystems offer before pulling you in.

!!! note "helmfile2compose vs dekube-engine"
    **dekube-engine** is the bare conversion engine — an empty pipeline with no built-in converters. **helmfile2compose** is the distribution: dekube-engine bundled with 8 extensions (workloads, indexers, HAProxy, Caddy) into a single `helmfile2compose.py`. This is what you download, run, and ship. When this page says "helmfile2compose", it means the distribution — the thing you actually use.

> *He who renders the celestial into the mundane does not ascend — he merely ensures that both realms now share his suffering equally.*
>
> — *Necronomicon, On the Folly of Downward Translation (or so I recall)*

For the dark and twisted ritual underlying the conversion — what gets converted, how, and what unholy transformations are applied — see [Architecture](https://docs.dekube.io/understand/architecture/).

## Requirements

- Python 3.10+
- `pyyaml` (`pip install pyyaml`)
- `helmfile` + `helm` (only if rendering from helmfile directly)
- **Docker Compose** (v2) — recommended for running the generated output. nerdctl compose requires the `flatten-internal-urls` transform to work (see [Network aliases](https://docs.dekube.io/limitations/#network-aliases-nerdctl)). Podman Compose works (v1.0.6+).

## Before you start: ingress controller

Your helmfile already uses an ingress controller. That choice is baked into your Ingress manifests — annotations, `ingressClassName` — and you're not going to change it for a compose migration. helmfile2compose needs a **rewriter** that speaks your controller's annotation dialect and translates it into reverse proxy configuration. If your controller isn't supported, your Ingress manifests will be silently skipped and you'll have no reverse proxy — which is the single most visible thing that breaks.

Two things happen with Ingress manifests, and they're handled by two different pieces. **Rewriters** read your controller-specific annotations (HAProxy, Nginx, Traefik) and translate them into a generic format. The **IngressProvider** is the actual compose service that handles requests — by default **Caddy**, bundled with the distribution. An [Nginx provider](https://github.com/dekubeio/dekube-provider-nginx) is also available as an extension if you prefer Nginx as your reverse proxy. Your K8s cluster had HAProxy or Nginx; in compose, the ingress provider takes over, using what the rewriter extracted. Both are pluggable — rewriters can be [installed or written](https://docs.dekube.io/extend/extensions/writing-rewriters/), and the ingress provider can be [replaced](https://docs.dekube.io/extend/extensions/writing-ingressproviders/). For the full picture, see [concepts](https://docs.dekube.io/understand/concepts/).

| Controller | Rewriter | Status |
|------------|----------|--------|
| **HAProxy** | bundled | stable |
| **Nginx** | [dekube-rewriter-nginx](https://github.com/dekubeio/dekube-rewriter-nginx) (extension) | stable |
| **Traefik** | [dekube-rewriter-traefik](https://github.com/dekubeio/dekube-rewriter-traefik) (extension) | POC |

If you use something else (Contour, Ambassador, Istio, AWS ALB, etc.) — standard Ingress `host`/`path`/`backend` fields are always read, so basic routing works, but controller-specific annotations won't translate. Check this *before* investing time in the rest of the setup.

CRDs won't crash the tool if unsupported — they're skipped with a warning. But "skipped" means the resources they would have produced are missing, and how much that hurts varies:

- Some CRDs are **backbone** — skip them and your stack is broken, not simplified. Install the matching extension.
- Some are **nice to have** — internal TLS, for instance, can often be disabled in your helmfile environment instead. Your stack works without it.
- Some are **ignorable** — monitoring in compose is [its own discussion](https://docs.dekube.io/catalogue/#servicemonitor).

There's also a fourth category: **live CRDs** that are fundamentally incompatible. Operators that watch the apiserver, apps that do leader election via Lease objects, runtime API consumers — these have no compose equivalent and no extension will fix them. They're [documented in concepts](https://docs.dekube.io/understand/concepts/#whats-behind-the-wall). If you're going to maintain dekube for your stack long-term, understanding the tier system is worth the detour — though not mandatory and not recommended for your mental health.

Ingress is not optional either.

!!! note "kubernetes2simple"
    [kubernetes2simple](https://k2s.dekube.io/) exists as an all-inclusive distribution — helmfile2compose + every extension, one script, zero config. It's convenient for a quick test, but probably not what you want long-term: it enables transforms like `bitnami` that silently replace images, which may not be desired for your stack. As a maintainer, you'll want to pick your extensions deliberately.

**fix-permissions** is bundled with the distribution and active by default. It generates a busybox init service that fixes bind mount ownership for non-root containers — in most cases a net gain. If you don't want it, disable it in `dekube.yaml`:

```yaml
extensions:
  fix-permissions:
    enabled: false
```

See [per-extension config](https://docs.dekube.io/reference/config/#per-extension-config-extensions) for the full mechanism.

## Installation

The [quick start](index.md#quick-start) uses dekube-manager to download everything automatically. Here's the manual approach, for when you want full control over what you install — or full accountability for what went wrong.

Download `helmfile2compose.py` from the [latest helmfile2compose release](https://github.com/dekubeio/helmfile2compose/releases/latest).

If your stack uses CRDs that have an [dekube extension](https://docs.dekube.io/catalogue/) (Keycloak, cert-manager, trust-manager), grab the extension `.py` files from their repos too and drop them in `.dekube/extensions/`:

```
.dekube/extensions/
├── keycloak.py            # from dekube-provider-keycloak
└── cert_manager.py        # from dekube-converter-cert-manager
```

This is the same directory that [dekube-manager](https://manager.dekube.io/docs/) uses when installing extensions automatically — so switching to the manager later is seamless.

## Preparing your helmfile

You need Helm to escape Helm. The irony writes itself.

Create a dedicated environment (e.g. `compose`) that disables K8s-only infrastructure. These components have no meaning in compose and will be auto-excluded on first run anyway, but disabling them avoids rendering useless manifests:

```yaml
# environments/compose.yaml
certManager:
  enabled: false
ingress:
  enabled: false
reflector:
  enabled: false
```

Everything else stays enabled — the tool needs to see your Deployments, Services, ConfigMaps, Secrets, and Ingress resources to do its job.

If your stack uses CRDs that have an [dekube extension](https://docs.dekube.io/catalogue/) (Keycloak, cert-manager, trust-manager), keep those enabled — the extensions you should have previously acquired will handle them.

## First run

This is the point of no return. Everything before this was theoretical. After this, you have a compose stack, and you will spend the rest of the evening staring at it.

```bash
python3 helmfile2compose.py --helmfile-dir ~/my-project -e compose \
  --extensions-dir .dekube/extensions --output-dir .
docker compose up -d
```

This renders the helmfile and converts in one step.

**No helmfile?** Use `--from-dir` instead of `--helmfile-dir` to skip the helmfile render step entirely. Point it at any directory of pre-rendered K8s YAML (from `helm template`, `kustomize build`, or plain manifests). The conversion is identical — `--helmfile-dir` just runs `helmfile template` for you first.

On first run, the tool creates `dekube.yaml` with sensible defaults:
- All PVCs registered as host-path bind mounts under `./data/`
- K8s-only workloads auto-excluded (any workload whose name contains `cert-manager`, `ingress`, or `reflector` — this targets controller Deployments, not Ingress resources)
- Project name derived from the source directory

**Verify the output**: `compose.yml` and `Caddyfile` should exist in your output directory. Run `docker compose config` to validate the generated compose file. If the Caddyfile is empty or missing host blocks, your ingress controller is probably not supported.

**Stop here and review `dekube.yaml`.** Do not skip this. The tool guessed, and its guesses are educated but soulless. You will almost certainly need to:
- Adjust volume paths
- Exclude workloads that make no sense outside K8s (operators, CRD controllers, etc.)
- Add overrides for images that need replacing (e.g. bitnami → vanilla, unless you enjoy debugging Bitnami's entrypoint scripts at 3 AM)

See [Configuration](configuration.md) for the full reference.

### CLI flags

| Flag | Description |
|------|-------------|
| `--helmfile-dir` | Directory containing `helmfile.yaml` or `helmfile.yaml.gotmpl` (default: `.`) |
| `-e`, `--environment` | Helmfile environment (e.g. `compose`, `local`) |
| `--from-dir` | Skip helmfile, read pre-rendered YAML from this directory |
| `--output-dir` | Where to write output files (default: `.`) |
| `--compose-file` | Name of the generated compose file (default: `compose.yml`) |
| `--extensions-dir` | Directory containing extension `.py` files |

`--from-dir` and `--helmfile-dir` serve the same purpose (providing input manifests). If both are specified, `--from-dir` takes priority.

### Output files

- `compose.yml` — services (incl. Caddy reverse proxy), volumes
- `Caddyfile` (or `Caddyfile-<project>` when `disable_ingress: true`) — reverse proxy config from Ingress manifests
- `dekube.yaml` — persistent config (see [Configuration](configuration.md))
- `configmaps/` — generated files from ConfigMap volume mounts
- `secrets/` — generated files from Secret volume mounts

## Check known issues and workarounds

Before debugging something for hours, check the [known workarounds](known-workarounds/index.md). Recurring tentacles have been identified, sliced, and served — no need to catch the same kraken twice.

If your stack includes Bitnami charts (Redis, PostgreSQL, Keycloak), install the [bitnami transform](https://github.com/dekubeio/dekube-transform-bitnami) — it handles the worst of it automatically.

## What works well

This section exists to give you false confidence. It worked for two of my platforms — I built this thing for them — and it seems to work for mijn-bureau. Smile and wave — but brace for impact.

- **Deployments, StatefulSets, DaemonSets, Jobs** — converted to compose services with the right image, env, command, volumes, and ports. Init containers and sidecars get their own services.
- **ConfigMaps and Secrets** — resolved inline into environment variables, or generated as files when volume-mounted.
- **Services** — network aliases (K8s FQDNs resolve natively via compose DNS), alias resolution, port remapping. If your K8s Service remaps port 80 to targetPort 8080, the tool rewrites URLs and env vars automatically.
- **Ingress** — converted to a Caddy reverse proxy with automatic TLS. Path-based routing, host-based routing, catch-all backends. Backend SSL annotations supported.
- **PVCs** — registered in config as bind mounts. `volumeClaimTemplates` (StatefulSets) included.
- **CRDs** — with [extensions](https://docs.dekube.io/catalogue/), Keycloak, cert-manager, and trust-manager CRDs are fully converted.

## What needs manual help

And here's where the false confidence evaporates.

- **Unsupported CRDs** — Zalando PostgreSQL, Strimzi Kafka, etc. are skipped with a warning unless you write an extension. You'll need to add equivalent services manually via the `services:` section in config.
- **Bitnami images** — often need replacing with vanilla equivalents via `overrides:`. Bitnami images have opinions about environment variables, init scripts, and volume paths that don't always translate well.
- **S3 virtual-hosted style** — compose DNS can't resolve `bucket.minio:9000`. Force path-style in your app config and use a `replacement` if needed.
- **CronJobs** — not converted. Run them externally or use a sleep-loop wrapper (but please don't).

See [Limitations](limitations.md) for the distribution-level summary, or [docs.dekube.io/limitations](https://docs.dekube.io/limitations/) for the exhaustive technical list.

## Ingress controllers — details {#ingress-controllers}

Still here? Good. This section documents annotation coverage and configuration — the part where you discover which annotations survived the trip and which ones didn't.

### Annotations handled

| Controller | Annotations |
|------------|-------------|
| **HAProxy** | `haproxy.org/path-rewrite` → strip prefix (scoped per path on multi-path rules), `haproxy.org/server-ssl` + `server-ca` → backend TLS, `server-sni` |
| **Nginx** | `rewrite-target`, `backend-protocol`, `enable-cors`, `proxy-body-size`, `configuration-snippet` (partial) |
| **Traefik** | `router.tls`, standard Ingress path rules. No middleware CRD support. |

HAProxy is built into the helmfile2compose distribution. Nginx and Traefik are extensions — install them with [dekube-manager](https://manager.dekube.io/docs/) or drop the `.py` file in your `extensions/` directory.

### Custom ingress class names

If your cluster uses custom `ingressClassName` values (e.g. `haproxy-internal`, `nginx-dmz`), add a mapping in `dekube.yaml` (see [ingress_types](https://docs.dekube.io/reference/config/#full-schema)):

```yaml
ingress_types:
  haproxy-internal: haproxy
  haproxy-external: haproxy
  nginx-dmz: nginx
```

Without this, helmfile2compose won't recognize the class and the Ingress is skipped with a warning.

## Recommended workflow

If you've made it this far and your stack boots, congratulations — you are now a maintainer of something that was never meant to exist. Here's how to keep it from collapsing:

1. **One helmfile, two environments.** Keep your K8s environment as-is. Add a `compose` environment that disables cluster-only components. Same charts, same values (mostly), different targets.

2. **Ship a `generate-compose.sh`.** A wrapper script that downloads dekube-manager, installs helmfile2compose + extensions, runs the conversion, and maybe generates secrets. See stoatchat-platform or lasuite-platform for examples.

3. **Ship a `dekube.yaml.template`.** Pre-configure excludes, overrides, and volume mappings that are specific to your project. The generate script copies it to `dekube.yaml` on first run. Users then customize their copy.

4. **Pin a release.** Use `--distribution-version` in dekube-manager and `==version` for extensions. Don't point at `latest`. The tool mutates between releases — that's documented in the changelog, which is more than most ecosystems offer. Pin your versions. I don't — but you should be aware that my sanity left a long time ago, by now. The heretic pins his own.

## Compatible projects

- **[stoatchat-platform](https://github.com/baptisterajaut/stoatchat-platform)** — 15 services. The first patient. Worked on the first try, which was the most dangerous outcome.
- **[lasuite-platform](https://github.com/baptisterajaut/lasuite-platform)** — 22 services + 11 init jobs. The second patient. Tentacles started appearing around the volumeClaimTemplates.

Both ship with `generate-compose.sh` and `dekube.yaml.template`. Reading their setup is probably more useful than anything I could write here.

Then it went very wrong on my own proprietary helmfile — operators, SOPS encryption, multi-environment, the full horror — and now even **[mijn-bureau-infra](https://github.com/MinBZK/mijn-bureau-infra)** (~30 services, nested helmfiles, Bitnami everywhere) seems to work with it. See [About](https://docs.dekube.io/about/) for the full explanation of how this went wrong.

## Garbage in, garbage out

The engine does **zero validation** of its input. Bad manifests in, ugly Python traceback out. Make sure your helmfile works on a real Kubernetes cluster before feeding it to dekube — it's a downstream consumer, not a linter. See [Concepts — No validation by design](https://docs.dekube.io/understand/concepts/#no-validation-by-design) for the full sermon.

## Final warning

This tool works. It has been tested on real helmfiles with real users. But it is, fundamentally, an act of desecration — stripping Kubernetes of everything that makes it Kubernetes (scheduling, scaling, self-healing, network policies, RBAC) and leaving behind a flat list of containers. Every edge case you hit is a reminder that you are running something that was designed for orchestration on a machine that has no orchestra — and that the person who asked you for a Docker Compose owes you a drink — or the psychiatric bill, whichever comes first.

> *The temple was not translated — it was dismantled, stone by stone, and rebuilt as a shed. The prayers still worked. The architect watched, powerless, as the faithful praised the shed.*
>
> — *De Vermis Mysteriis, On Unnecessary Simplifications (the evidence is circumstantial)*
