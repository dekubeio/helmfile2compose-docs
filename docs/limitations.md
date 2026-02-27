# Limitations

This page covers the limitations most relevant to distribution users. For the exhaustive technical breakdown — including what's ignored, what's silently dropped, and the full existential crisis — see [Limitations on docs.dekube.io](https://docs.dekube.io/limitations/).

## Key limitations

### Network aliases (nerdctl)

nerdctl compose silently ignores network aliases. Without the [`flatten-internal-urls`](https://docs.dekube.io/catalogue/#flatten-internal-urls) transform, services will fail to reach each other by K8s FQDN. Install the transform or switch to Docker Compose / Podman Compose.

### Startup ordering

Init containers become separate services with `restart: on-failure` — they retry until they succeed, but nothing prevents the main container from starting concurrently. Expect noisy logs on first boot. Everything converges eventually.

### Secrets

Kubernetes Secrets are dumped as plain-text environment variables into `compose.yml`. Do not commit it to version control.

### CRDs

Operator-managed resources are skipped unless a loaded extension handles them. See the [extension catalogue](https://docs.dekube.io/catalogue/) for available converters (cert-manager, Keycloak, trust-manager, ServiceMonitor).

### CronJobs

Not converted. No compose equivalent.

### emptyDir volumes

Not shared between init containers and the main container in compose. Map to a named volume manually if needed.

### Bind mount permissions

Handled automatically by the bundled `fix-permissions` transform — non-root containers get a `chown` init service. No manual intervention needed in most cases.

---

For the full list, rationale, and workarounds: [docs.dekube.io/limitations](https://docs.dekube.io/limitations/).
