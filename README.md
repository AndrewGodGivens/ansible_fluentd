sbog/fluentd
============

Role to install and configure fluentd.

#### Requirements

Ansible 2.4. For `fluentd_in_docker: true`, the `community.docker` collection
must be installed on the controller: `ansible-galaxy collection install
community.docker` (already required if you're also using this project's
`postgres_swarm` role).

#### Role Variables

```yaml
# true if td-agent v4 is installed on the host
fluentd_migrate_to_v5: false
fluentd_del_old_conf: false

# for ssl 
ssl_directory_path: "/etc/ssl_certs"
domain_directory: "example.com"

fluentd_plugins: []
fluentd_config:
  - directive: system
    data:
      - |
        process_name fluentd
        log_level warn
```

#### Running Fluentd in Docker Compose

Set `fluentd_in_docker: true` to deploy Fluentd as a Docker Compose service
instead of the native `fluent-package` install. `fluentd_config` keeps the
exact same meaning and is rendered into the same `/etc/fluent/fluentd.conf`
either way, so an existing native config keeps working unchanged after
switching the flag.

The container uses the `grafana/fluent-plugin-loki` image (ships with
`fluent-plugin-grafana-loki` and `fluent-plugin-systemd` preinstalled) and is
started with its own unmodified `entrypoint.sh` (only the documented
`FLUENTD_CONF` env var is set), so its Bundler-managed gemset is used
exactly as the image intends — no custom command/entrypoint override.
`/var/log`, `{{ log_path }}`, `/var/lib/fluentd` and `ssl_directory_path`
are bind-mounted at identical paths inside the container, so existing
`tail` sources, `pos_file` and cert paths in
`fluentd_config` need no changes.

`fluentd_plugins` (native mode's `fluent-gem install` list) is **not** used
in Docker mode — installing extra gems into this image at container
*startup* was tried and reverted: forcing Bundler to a different vendor path
made it try to rebuild the base fluentd native gems (`cool.io`, `msgpack`,
`strptime`) from source, and the runtime image has no C toolchain to do
that. If you need a plugin beyond `fluent-plugin-grafana-loki`/
`fluent-plugin-systemd`, build your own image at *build* time instead, where
dev tools are available, and point `fluentd_image` at it:

```dockerfile
FROM grafana/fluent-plugin-loki:main
RUN echo "gem 'fluent-plugin-prometheus'" >> /fluentd/Gemfile \
 && bundle install --gemfile=/fluentd/Gemfile
```

Applying the stack is a plain, always-run task using
`community.docker.docker_compose_v2` (idempotent — reconciles the actual
running state every time, not just when the rendered files change). It does
**not** simulate fluentd_image's own entrypoint/plugin loading; check
`docker compose logs fluentd` (or `docker ps`) after a run if `fluentd.conf`
itself has a mistake.

```yaml
fluentd_in_docker: true

# collect logs from Docker Swarm services (and any other container)
# running on this node — Docker's default log-driver is set to fluentd,
# so Docker itself ships every container's stdout/stderr to Fluentd's
# forward input.
fluentd_manage_docker_log_driver: true      # default
fluentd_forward_bind: "127.0.0.1"           # default
fluentd_forward_port: 24224                 # default

# restarting the Docker daemon restarts every container on the node,
# including running Swarm tasks. Off by default — apply manually with
# `systemctl restart docker`, or opt in here.
fluentd_docker_restart_allowed: false        # default
```

To also receive Swarm container logs, add a `forward` source to
`fluentd_config` alongside your existing sources:

```yaml
  - directive: source
    data:
      - |
        @type forward
          bind 0.0.0.0
          port 24224
```

#### Dependencies

None

#### Example Playbook

```yaml
- name: Install and configure Fluentd
  hosts: localhost
  remote_user: root
  roles:
    - fluentd
```

#### License

Apache 2.0

#### Author Information

Stanislaw Bogatkin (https://sbog.ru)
