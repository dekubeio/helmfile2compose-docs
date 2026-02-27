# helmfile2compose

The distribution that generates a working `docker compose` stack from your Kubernetes manifests. One source of truth, one script, one command.

> *He who renders the celestial into the mundane does not ascend — he merely ensures that both realms now share his suffering equally.*
>
> — *Necronomicon, On the Folly of Downward Translation (I think)*

## What is this?

**helmfile2compose** is a [dekube](https://dekube.io) distribution — the [dekube-engine](https://docs.dekube.io/engine/) bundled with 8 extensions into a single `helmfile2compose.py`. It takes your Kubernetes manifests (from helmfile, Helm, or plain YAML) and produces a `compose.yml` + whatever configfile your proxy server will use, and everything needed to run your stack with `docker compose up`.

For the engine internals, extension development, and ecosystem architecture, see [docs.dekube.io](https://docs.dekube.io/).

## Quick start

```bash
# Download the distribution
curl -fsSL https://raw.githubusercontent.com/dekubeio/dekube-manager/main/dekube-manager.py -o dekube-manager.py
python3 dekube-manager.py

# Convert and run
python3 .dekube/helmfile2compose.py --helmfile-dir ~/my-project -e compose --output-dir .
docker compose up -d
```

See [Getting started](getting-started.md) for the full setup guide.

## Documentation

- **[Getting started](getting-started.md)** — installation, first run, adapting helmfile2compose for your own helmfile
- **[Configuration](configuration.md)** — `dekube.yaml` deep dive: volumes, overrides, secrets, replacements
- **[dekube-manager](https://manager.dekube.io/docs/)** — installing helmfile2compose and extensions via the package manager
- **[Operations](operations.md)** — day-to-day: updating, data management, troubleshooting
- **[Advanced](advanced.md)** — cohabiting with existing infrastructure, multiple projects, disabling Caddy
- **[Known workarounds](known-workarounds/index.md)** — sushi recipes for the tentacles that don't fit
- **[Glossary](glossary.md)** — terms, acronyms, and Lovecraftian vocabulary decoded
- **[Limitations](limitations.md)** — what gets lost in translation

## Compatible projects

- **[stoatchat-platform](https://github.com/baptisterajaut/stoatchat-platform)** — 15 services. Chat platform (Revolt rebranded).
- **[lasuite-platform](https://github.com/baptisterajaut/lasuite-platform)** — 22 services + 11 init jobs. Collaborative suite (La Suite Num.).
- **[mijn-bureau-infra](https://github.com/numerique-gouv/mijn-bureau-infra)** — ~30 services. Dutch government digital workplace. Requires `nginx` and `bitnami` extensions.

## The ecosystem

| Repo | What it is |
|------|------------|
| [kubernetes2simple](https://github.com/dekubeio/kubernetes2simple) | Turnkey distribution — helmfile2compose + all extensions + automagic bootstrap script. |
| [dekube-engine](https://github.com/dekubeio/dekube-engine) | Bare conversion engine — empty registries, no opinions. |
| [helmfile2compose](https://github.com/dekubeio/helmfile2compose) | This distribution — core + 8 bundled extensions → single `helmfile2compose.py`. |
| [dekube-manager](https://github.com/dekubeio/dekube-manager) | Package manager — downloads distribution + extensions, resolves dependencies. |
| [Extension catalogue](https://docs.dekube.io/catalogue/) | Single-file modules: providers, converters, transforms, rewriters. |
| [dekube-docs](https://docs.dekube.io/) | Engine documentation, extension development guides. |
