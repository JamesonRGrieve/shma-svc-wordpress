# WordPress Service Role

This role renders a WordPress workload definition for the shma control plane. It
focuses on secure defaults, persistent user content, and operational guidance so
that upgrades can be performed without downtime.

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
