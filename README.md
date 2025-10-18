# WordPress Service Role

This role renders a hardened WordPress workload definition for the shma control
plane. It focuses on secure defaults, persistent media storage, and clear
operational guidance so that upgrades can be performed without downtime. The
rendered manifest defaults to a single replica (`svc_wordpress_replicas: 1`) to
keep shared storage safe for users who rely on NFS or other non-concurrent
object stores. Increase the replica count for high availability only when the
backing storage and plugins support concurrent writers.

## Prerequisites

* Provide a routable hostname via `svc_wordpress_ingress_hostname`. The role
  asserts that `svc_wordpress_url` references the same hostname to prevent
  redirect loops.
* Supply database credentials (`svc_wordpress_db_*`) or dependency exports that
  define `DATABASE_HOST` and `DATABASE_PORT`. Dependency exports are preferred;
  explicit overrides only apply in exceptional topologies. The role validates
  the `dependency_exports` schema before using the exported values.
* Set `svc_wordpress_lxc_config_path` (and optionally
  `svc_wordpress_exports_env_path`) so the rendered manifest and `exports.env`
  file can be collected by the control plane.
* When Redis TLS peer verification is enabled, provide
  `svc_wordpress_redis_tls_secret` containing the CA certificate so the pod can
  validate the cache endpoint.

## Security hardening

### Authentication salts and rotation

Authentication keys and salts are rendered into a dedicated secret that is
mounted at `/run/secrets/wp-salts/wp-salts.php`. Only the PHP include file is
stored in the secret to avoid duplicating data or drifting values between
different keys. WordPress loads the file during bootstrap instead of relying on
environment variables, keeping the values out of `kubectl describe` and
container logs. Rotate the keys periodically by updating both the structured
variables and the secret:

```sh
kubectl create secret generic wordpress-auth --from-literal=AUTH_KEY=... --dry-run=client -o yaml | kubectl apply -f -
```

Applying the updated secret triggers a rolling restart so the salts take effect.
Document the rotation cadence (for example, monthly) alongside the secret
management process.

### HTTPS enforcement and proxy headers

The generated configuration normalises reverse-proxy headers (`X-Forwarded-*`)
and forces HTTPS for admin traffic. When additional restrictions are required
(for example, basic authentication or IP allowlists around `/wp-admin`), add the
appropriate ingress annotations such as `nginx.ingress.kubernetes.io/auth-type`
or `nginx.ingress.kubernetes.io/whitelist-source-range`.

### XML-RPC disabled by default

Requests to `xmlrpc.php` are blocked early in the bootstrap process. If an
integration relies on XML-RPC, remove the guard via
`svc_wordpress_extra_wordpress_config` and consider alternative authentication
mechanisms.

### Must-use plugins

Use the `svc_wordpress_mu_plugins` mapping to ship must-use plugins. Each entry
is rendered into a ConfigMap and mounted under
`/var/www/html/wp-content/mu-plugins`, guaranteeing that hardening plugins (for
example, cache bootstrap or security filters) load on every request. The role
validates that each file contains PHP code (or is empty) so syntax mistakes fail
fast during deployment instead of breaking the container at runtime.

### Service annotations

`svc_wordpress_service_annotations` defaults to Prometheus discovery metadata so
platform monitoring can scrape the rendered service. Override or extend the
mapping to add service-mesh, load-balancer, or policy annotations as required by
your environment.

## Database and caching

### Database connectivity

The workload automatically prefers dependency exports for database host and port
information. Override `svc_wordpress_db_host` or `svc_wordpress_db_port` only
when the consuming environment cannot surface the exports directly. TLS can be
enabled with `svc_wordpress_db_use_tls`. Database passwords must meet a strong
complexity requirement (16+ characters, upper/lower case, digit, special
character); weak credentials fail validation before templates render.

### Redis object cache

Redis credentials are always required. When
`svc_wordpress_enable_object_cache: true`, the manifest sets `WP_CACHE` and the
Redis scheme. Provide a CA certificate via `svc_wordpress_redis_tls_secret` to
validate TLS endpoints. After deployment, confirm that the Redis Object Cache
plugin is installed and connected; the init container no longer assumes the
plugin exists.

### PHP runtime

`svc_wordpress_extra_php_ini` defaults to opinionated OPcache and upload limits
so PHP code remains cached in memory. Tune these values when hosting large code
bases or heavy upload workflows.

### Page caching and CDN purge

Full-page caching (for example, FastCGI cache or a CDN) dramatically improves
latency. Configure an external caching layer and use
`svc_wordpress_extra_environment` or must-use plugins to trigger purge webhooks
when content changes.

### Database query cache guidance

Coordinate with the MariaDB service to enable InnoDB buffer pool sizing and
(binlog-based) point-in-time recovery. Traditional MySQL query cache is disabled
on modern MariaDB releases; instead, ensure the buffer pool is sized correctly
and monitor slow-query logs through the database service.

## Media and uploads

### Persistent storage sizing

The uploads volume (`{{ svc_wordpress_name }}-uploads`) backs
`svc_wordpress_uploads_path`. Start with at least 2 GiB per 10,000 media assets
and adjust based on your retention policy. Override `svc_wordpress_uploads_size`
per environment and monitor usage with PVC metrics. For deployments with very
large libraries (100 GiB+), allocate dedicated storage classes optimised for
throughput.

### Object storage and credentials

Enable S3-compatible uploads by setting `svc_wordpress_enable_object_storage:
true` and providing the bucket details and credentials. Secrets surface through
environment variables (`S3_UPLOADS_KEY`/`S3_UPLOADS_SECRET`) so the container no
longer mounts writable credential files. Update
`svc_wordpress_extra_wordpress_config` if the selected plugin requires
additional constants.

### CDN integration and mixed content

Expose a CDN-backed URL by defining `WP_CONTENT_URL` or
`WP_PLUGIN_URL` through `svc_wordpress_extra_environment`. Validate that the CDN
matches `svc_wordpress_url` to avoid mixed-content warnings. When the CDN
supports purge webhooks, connect them to WordPress publish/update hooks so new
content invalidates stale cache entries.

### Image optimisation

Integrate an optimisation service such as ShortPixel, Imagify, or a containerised
image processor. The role mounts `/tmp` as a `tmpfs`, which is suitable for
short-lived optimisation jobs. Document the chosen plugin and required API keys
in your service inventory.

### Custom media paths

Override `svc_wordpress_uploads_path` to relocate the uploads directory. Ensure
matching updates to any object storage plugin configuration so relative paths
remain aligned.

## Multisite deployments

Set `svc_wordpress_enable_multisite: true` and populate `svc_wordpress_multisite`
with the network topology (domain, path, and numeric identifiers). The rendered
manifest drops a dedicated `wp-config.d/multisite.php` include that holds the
network constants while keeping per-site domain routing under the ingress
configuration.

## Background jobs and cron

WordPress’ internal cron is disabled in favour of an explicit Kubernetes CronJob.
The schedule, container image, and timeout are controlled via
`svc_wordpress_cron_schedule`, `svc_wordpress_cron_image`, and
`svc_wordpress_cron_timeout_seconds`. The job invokes
`{{ svc_wordpress_url }}/wp-cron.php?doing_wp_cron=1` with retry logic to process
queued tasks even when the site receives little foreground traffic.

## Health probes and reliability

* **Readiness:** Performs an HTTP GET against
  `svc_wordpress_readiness_path` (default `/wp-admin/install.php`) so failed
  database connections surface before traffic is routed without requiring curl
  inside the container image.
* **Startup:** Polls the admin installer endpoint with generous delays to avoid
  flapping during plugin activation or schema migrations.
* **Liveness:** Uses the same HTTP-based probe against
  `svc_wordpress_liveness_path` (default `wp-cron.php`) to ensure PHP-FPM and
  Apache respond to HTTP requests.

Increase probe timeouts via the corresponding variables for heavily loaded
installations.

## Backups and disaster recovery

* **Database:** Follow the MariaDB service runbook to schedule logical dumps and
  enable binlog retention for point-in-time recovery.
* **Uploads:** Back up the uploads PVC or the object storage bucket separately
  from the database. Restic or Velero integrations work well for PVC snapshots.
* **Disaster recovery:** Test restore procedures regularly—document the steps to
  recreate the Kubernetes secrets, restore the database, and resynchronise media
  assets.
* **Staging environments:** Clone the production database and media into a
  staging namespace by reusing this role with a different ingress hostname. This
  keeps plugin/theme updates isolated until they are validated.

## CI/CD and zero-downtime updates

* Use GitOps or CI pipelines to manage configuration changes. Reference secrets
  via the platform export mechanism rather than embedding values in the pipeline.
* During upgrades, rely on Kubernetes rolling updates; the readiness probe will
  only succeed once WordPress can talk to the database.
* Document plugin/theme deployment workflows. When distributing custom code,
  bake it into the OCI image or mount it through ConfigMaps/Secrets so the main
  container remains immutable.
* Automate plugin and core updates through scheduled GitOps changes or CI jobs
  that refresh the underlying image. Record the procedure so operators know how
  to trigger emergency security updates.
* Multi-region deployments require read replicas and global load balancing that
  are outside the scope of this role. Document regional replication and CDN
  strategies if global availability is required.

## Troubleshooting

* **White screen of death:** Check recent deployments and PHP logs in the pod.
  Temporarily disable suspect plugins by removing them from the mu-plugins map
  or the underlying storage.
* **Database connection errors:** Verify the exported database host/port in
  `exports.env` and confirm the Kubernetes secret contains the expected
  password.
* **Permission issues:** Remember that the root filesystem is read-only; mount
  writable volumes (PVCs or `emptyDir`) for any path that must support writes.
* **Redis connectivity:** Ensure the Redis TLS secret is mounted and that the
  certificate matches the hostname presented by the cache endpoint.

For additional instrumentation, consider enabling application performance
monitoring plugins through the mu-plugins mechanism so telemetry loads on every
request.
