# WordPress Service Role

This role renders a WordPress workload definition for the shma control plane. It
focuses on secure defaults, persistent user content, and operational guidance so
that upgrades can be performed without downtime. The rendered manifest defaults
to a single replica (`svc_wordpress_replicas: 1`) to keep shared storage safe
for users who rely on NFS or other non-concurrent object stores. Increase the
replica count for high availability only when the backing storage and plugins
support concurrent writers.

## Zero-downtime updates

* Run at least two replicas of the ingress controller and database to avoid
  single points of failure.
* Apply updates with `kubectl apply` and rely on Kubernetes rolling updates so
  that the new pod becomes Ready (`/wp-login.php`) before the previous pod is
  terminated.
* Keep the persistent uploads volume attached so that media assets remain
  available to both the old and new pods during the rollout.
* Flush Redis object cache entries after the rollout if plugins introduce new
  cache keys.

The readiness probe matches the `/wp-login.php` endpoint so the rolling update
only proceeds when the new pod can serve authenticated traffic.

## Plugin and theme persistence

* User-generated content is stored under `wp-content/uploads`, which is backed
  by the `{{ svc_wordpress_name }}-uploads` PersistentVolumeClaim.
* Custom themes and plugins should be packaged into OCI images or mounted from a
  read-only ConfigMap/Secret to keep the container image immutable.
* Cache directories are ephemeral `emptyDir` mounts and are safe to clear on pod
  restart.

## Additional configuration

* Provide WordPress authentication keys and salts via the
  `svc_wordpress_auth_keys` variable.
* Mount extra PHP configuration snippets by setting `svc_wordpress_extra_php_ini`;
  the contents are rendered into `/usr/local/etc/php/conf.d/zz-custom.ini`.
* The role writes an `exports.env` file alongside the rendered LXC manifest (or
  at `svc_wordpress_exports_env_path` when overridden). The file exposes
  `APP_FQDN`, `APP_PORT`, and `BACKEND_IP={{ svc_wordpress_name }}` so edge
  automation can configure upstream routing. For example:

  ```env
  APP_FQDN=wordpress.example.com
  APP_PORT=443
  BACKEND_IP=wordpress
  ```
* Enable S3-compatible object storage by setting
  `svc_wordpress_enable_object_storage: true`, providing the bucket and region,
  and wiring the credentials into `svc_wordpress_object_storage_access_key` and
  `svc_wordpress_object_storage_secret_key`. Supply those values via vaults or
  the platform's secret export mechanism (for example,
  `svc_wordpress_object_storage_access_key: "{{ lookup('env', 'S3_ACCESS_KEY') }}"`).
  The role renders a Kubernetes Secret named
  `{{ svc_wordpress_name }}-object-storage` and mounts the credentials through
  environment variables consumed by the S3 plugin.
