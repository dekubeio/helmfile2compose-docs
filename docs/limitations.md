# Limitations

You replaced the perfectly sane orchestra conductor with a horribly mutated chimpanzee and you wonder why the audience needs earplugs.

This page covers what you lost when you chose compose over Kubernetes, scoped to what matters when running the tool day-to-day. For the exhaustive technical breakdown — including what's ignored, what's silently dropped, and the full existential crisis — see [Limitations on docs.dekube.io](https://docs.dekube.io/limitations/).

## Key limitations

### Network aliases (nerdctl)

nerdctl compose silently ignores network aliases. Without the [`flatten-internal-urls`](https://docs.dekube.io/catalogue/#flatten-internal-urls) transform, services will fail to reach each other by K8s FQDN. Install the transform or switch to Docker Compose / Podman Compose.

### Startup ordering

Init containers become separate services with `restart: on-failure` — they retry until they succeed, but nothing prevents the main container from starting concurrently. Expect noisy logs on first boot. Everything converges eventually.

### Secrets

Kubernetes Secrets are dumped as plain-text environment variables into `compose.yml`. The security model you carefully built is now a flat file on your laptop. Do not commit it to version control.

### CRDs

Operator-managed resources are skipped unless a loaded extension handles them. "Skipped" means the resources they would have produced are missing — your cert-manager `Certificate` becomes a warning line, not a PEM file. See the [extension catalogue](https://docs.dekube.io/catalogue/) for available converters (cert-manager, Keycloak, trust-manager, ServiceMonitor).

### CronJobs

Not converted. No compose equivalent. People use host crontab entries, sleep-loop containers, or quiet denial. None of these are endorsed.

### Long service names

Linux hostnames are limited to 63 characters. Compose uses the service name as the container hostname, and Helm-generated names can easily exceed that limit. dekube automatically sets a truncated `hostname:` on affected services to avoid `sethostname: invalid argument` failures. The compose service name itself is unchanged — only the container hostname is shortened.

### emptyDir volumes

Not shared between init containers and the main container in compose. This matters most when an init container prepares data (seeds a cache, renders templates, copies config) that the main container expects to find on startup. Map to a named volume manually if needed.

### Bind mount permissions

Linux file permissions — the last thing you expect to fight after converting an entire orchestrator. Handled automatically by the bundled `fix-permissions` transform — non-root containers get a `chown` init service. No manual intervention needed in most cases.

---

For the full list, rationale, and workarounds: [docs.dekube.io/limitations](https://docs.dekube.io/limitations/).
