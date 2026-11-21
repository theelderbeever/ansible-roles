# ansible-roles

Home server setup targeting Ubuntu 26.04.

| Role | What it does |
| --- | --- |
| `role_ufw` | ufw default-deny incoming/routed, SSH allowed before enabling |
| `role_docker` | Docker CE from download.docker.com; makes published ports obey ufw |
| `role_tailscale` | Tailscale from pkgs.tailscale.com; optional auth-key join; trusts `tailscale0` in ufw |
| `role_prometheus` | Prometheus server from the Ubuntu archive, plus a local node_exporter via `role_node_exporter` |
| `role_node_exporter` | node_exporter from the Ubuntu archive for edge hosts (also used by `role_prometheus` on core), running as its own user |
| `role_nvidia_gpu_exporter` | [nvidia_gpu_exporter](https://github.com/utkuozdemir/nvidia_gpu_exporter) from upstream `.deb` releases (**NVIDIA driver must already be installed**) |
| `role_unpoller` | UnPoller from golift's packagecloud repo; exports UniFi metrics to Prometheus on :9130 |
| `role_grafana` | Grafana from apt.grafana.com with a provisioned Prometheus datasource |
| `role_pihole` | Native Pi-hole v6, declarative FTL config, adlists + regex lists in gravity |
| `role_traefik` | Traefik binary + systemd, Let's Encrypt via DNS challenge, fronts the above |

## Installing Ansible

With [uv](https://docs.astral.sh/uv/):

```sh
uv tool install ansible-core --with ansible
uv tool upgrade ansible-core                  # later, to update
```

Install `ansible-core`, not `ansible`. The `ansible` PyPI package is only a
bundle of community collections. The `ansible` / `ansible-playbook` / ... commands
come from its `ansible-core` dependency, and `uv tool install` only links a
package's *own* commands, so `uv tool install ansible` leaves you with just
`ansible-community`. `--with ansible` still brings in the collections
(including `community.general`). Newer uv releases also accept
`uv tool install ansible --with-executables-from ansible-core`, which does the same.

## Usage

```sh
ansible-galaxy collection install -r requirements.yml  # skip if installed via --with ansible
cp inventory/group_vars/all/vault.yml.example inventory/group_vars/all/vault.yml
ansible-vault encrypt inventory/group_vars/all/vault.yml
ansible-playbook site.yml --ask-vault-pass --diff            # everything
ansible-playbook site.yml --ask-vault-pass --tags pihole     # one role
```

Each role's `defaults/main.yml` documents every knob. See
[Inventory layout](#inventory-layout) for where to set them.

## Running roles independently

Every role can run on its own (`ansible-playbook site.yml --tags pihole`, or a
playbook listing just that role). No role reads another role's variables or
needs another role to have run first. What you lose by running one alone:

* **Firewall**: each role adds its own ufw rules and installs the `ufw`
  package if missing, but only `role_ufw` *enables* ufw. Set
  `manage_ufw: false` to make every role skip ufw entirely (e.g. on a host
  with a different firewall).
* **Pi-hole without Traefik**: the web UI listens on 18080 but isn't opened
  in ufw; set `pihole_ufw_allow_web_from: [192.168.1.0/24]` (or similar).
* **Grafana without a local Prometheus**: point `grafana_prometheus_url` at
  wherever Prometheus runs.
* **Traefik** only routes; backends that aren't installed just return 502.
  It always needs `traefik_acme_email` and DNS-provider credentials.
* **Prometheus** installs node_exporter on its own host by including
  `role_node_exporter` (`prometheus_install_node_exporter: false` to skip).
  Edge hosts get `role_node_exporter` on its own.
* **UnPoller** is only scraped if Prometheus has the `unpoller` job (it's in
  `group_vars`, not in either role).

Pi-hole and Traefik can be applied in either order on the same host: Pi-hole's
web server is configured for 18080 before its first start, so it never claims
:80.

## Service accounts

| Service | Runs as | Notes |
| --- | --- | --- |
| prometheus | `prometheus` | Ubuntu package default; owns `/var/lib/prometheus` |
| nvidia_gpu_exporter | `nvidia_gpu_exporter` | Name hard-coded by the upstream package/unit; created before install |
| node_exporter | `node-exporter` | The package defaults to `prometheus`; a systemd drop-in moves it to its own account |
| unpoller | `unpoller` | Created by the package; `up.conf` (holds controller credentials) is `root:unpoller 0640` |
| grafana-server | `grafana` | Grafana package default |
| pihole-FTL | `pihole` | Created before the installer runs; name is hard-coded upstream |
| traefik | `traefik` | Config in `/etc/traefik` is `root:traefik` (read-only to Traefik); `acme.json` lives in `/var/lib/traefik` via `StateDirectory=` |
| dockerd, tailscaled | root | Both manage interfaces/iptables, so a separate user isn't appropriate. Membership in the `docker` group (`docker_users`) is root-equivalent. |

Config files are owned by `root` with group read for the service account, and
files holding secrets (`/etc/traefik/traefik.env`, `/etc/default/pihole-FTL`)
are `root:root 0600`. systemd reads them before dropping privileges.

## Notes

### Docker vs ufw
Docker publishes ports via DNAT + FORWARD, which never touches ufw's INPUT
rules, so `-p 8080:80` is world-reachable even with ufw denying 8080.
`role_docker` hooks Docker's `DOCKER-USER` chain into ufw (approach from
[ufw-docker](https://github.com/chaifeng/ufw-docker)), so published ports are
dropped unless you allow them with a ufw **route** rule on the *container* port:

```yaml
docker_ufw_route_rules:
  - { port: "80", proto: tcp, comment: whoami }   # ufw route allow proto tcp from any to any port 80
```

Ports bound to `127.0.0.1` (e.g. `-p 127.0.0.1:8080:80`) are unaffected and the
preferred way to put containers behind Traefik. Container networks can still
reach the LAN/internet (`docker_ufw_trusted_source_networks`). IPv6 isn't
covered; Docker's IPv6 is off by default.

### Pi-hole
* **Port 53**: Ubuntu's systemd-resolved stub listener holds `127.0.0.53:53`,
  colliding with FTL's dnsmasq. The role disables the stub
  (`/etc/systemd/resolved.conf.d/pihole.conf`) and repoints `/etc/resolv.conf`
  at `/run/systemd/resolve/resolv.conf`.
* **Config** is passed as `FTLCONF_*` env vars (`/etc/default/pihole-FTL` via a
  systemd drop-in; deliberately outside `/etc/pihole`, which FTL's prestart
  script `chown -R`s to `pihole` on every start) rather than templating `pihole.toml`, which FTL rewrites
  itself. Settings set this way show as read-only in the web UI. Add more via
  `pihole_ftl_config` using dotted keys (`dns.hosts`, `dns.cnameRecords`, ...).
* The web UI is moved to `:18080` (before first start) so Traefik owns 80/443.
* Adlists and domain/regex entries are upserted into `gravity.db`; entries not
  listed are left alone (nothing is deleted).
* **Web password**: set `vault_pihole_web_password` to manage it, leave it unset
  to keep the installer's random one (`sudo pihole setpassword` to change), or
  set `pihole_web_auth: false` for no login at all.
* Query history (`pihole-FTL.db`) isn't migrated. Copy it from `references/` by
  hand if you want it.

### Traefik
* `traefik_version: latest` resolves via the GitHub API each run; pin a version
  for reproducibility. Binaries live in `/opt/traefik/v<version>/` (root-owned) with
  `/usr/local/bin/traefik` symlinked to the active one.
* Uses a single wildcard cert (`traefik_wildcard_cert: true`) instead of one
  per subdomain. Requires Traefik >= 3.3 (`dnsChallenge.propagation`).
* The insecure `:8080` API from the original config is off; the dashboard is
  served at `traefik.<domain>`. Set `traefik_dashboard_basic_auth_users` to
  protect it.
* Routes are in `dynamic_conf.toml`, which is watched, so changes apply live.

### Tailscale
On the Pi-hole host keep `--accept-dns=false` so MagicDNS doesn't take over the
local resolver. You can point the tailnet's DNS at this host's Tailscale IP
(Pi-hole's `listeningMode: ALL` answers on `tailscale0`). `tailscale_up_args`
only apply on first join; use `tailscale set ...` afterwards.

Without `vault_tailscale_authkey` the role installs and starts `tailscaled`,
then prints the exact `sudo tailscale up ...` command to run once on the host
(it opens a login URL). The node stays logged in across later runs.

### UnPoller
* Needs a read-only account on the UniFi side: a **Limited Admin** with
  read-only access to the Network app, or a local API key on newer UniFi OS
  consoles (`api_key` instead of `user`/`pass`). See
  https://unpoller.com/docs/install/controllerlogin.
* UniFi OS consoles (UDM/UDM Pro/UXG/Cloud Key Gen2+): no `:8443` in the URL.
* Only the Prometheus output is enabled; the package's default InfluxDB
  output is switched off. Prometheus scrapes it via the `unpoller` job in
  `group_vars`.
* Grafana dashboards (Prometheus versions) to import by ID: 11311 Sites,
  11312 USW, 11313 USG, 11314 UAP, 11315 Clients, 11310 Client DPI (needs
  `save_dpi: true`).

### Inventory layout
Groups describe what a machine **does** and what it **has**. A host can be in
any number of groups, and each play in `site.yml` targets one group:

| Group | Meaning | Play gives it |
| --- | --- | --- |
| `core` | the main server (exactly one) | the full stack, incl. Prometheus + its own node_exporter |
| `edge` | satellite devices | node_exporter |
| `gpu` | has an NVIDIA GPU (driver pre-installed) | nvidia_gpu_exporter |

`gpu` is a capability, not a tier: add only the edge (or core) hosts that
actually have one. Future capabilities (e.g. `zfs`, `raspi`) work the same way.

Where variables go (later wins):

| File | For |
| --- | --- |
| `group_vars/all/` | network-wide: `base_domain`, `lan_cidr`, `ansible_user`, secrets (`vault.yml`) |
| `group_vars/<group>/` | everything in that group, e.g. the core stack's config |
| `host_vars/<host>.yml` | one machine: `ansible_host`, NIC names, local users |

Prometheus' scrape targets are built from the inventory: every `edge` host is
scraped on :9100 and every `gpu` host on :9835, at its `scrape_address`
(`ansible_host` by default). Adding a host to a group is enough.

Adding an edge device:

```yaml
# hosts.yml
edge:
  hosts:
    beeverdam-pc:
    new-pi:            # + host_vars/new-pi.yml with ansible_host
```

### NVIDIA GPU exporter
* **Prerequisite, not managed here:** the NVIDIA driver with working
  `nvidia-smi` / `libnvidia-ml.so.1`. The role runs `nvidia-smi -L` first and
  fails with a clear message if it doesn't work (`nvidia_gpu_exporter_preflight`).
* Installs the release `.deb` (checksum-verified against `checksums.txt`);
  `nvidia_gpu_exporter_version: latest` resolves via the GitHub API, so pin it
  for reproducibility.
* Flags are set via a systemd drop-in (`ExecStart=` override) because the
  exporter has no config file. Default backend is `nvml`; `exec` shells out to
  `nvidia-smi`.
* Grafana dashboard: 14574.

### Migrating hand-installed exporters
Both exporter roles take over from upstream "binary + unit file" installs:
* `role_node_exporter` stops, disables and deletes
  `/etc/systemd/system/node_exporter.{socket,service}` (they'd hold :9100) and
  runs Ubuntu's `prometheus-node-exporter` instead
  (`node_exporter_remove_manual_install`).
* `role_nvidia_gpu_exporter` disables and deletes
  `/etc/systemd/system/nvidia_gpu_exporter.service`, which would shadow the
  package's unit (`nvidia_gpu_exporter_remove_manual_unit`).

The old `/usr/bin/node_exporter` binary and `node_exporter` user are left
behind; remove them by hand once you're happy. Upgrading the GPU exporter from
1.x to 2.x also switches the default backend to `nvml`; check your dashboards
afterwards, or set `nvidia_gpu_exporter_backend: exec` for the old behaviour.

### Grafana
`grafana_admin_password` only takes effect on the very first start. Afterwards
use `sudo grafana cli admin reset-admin-password`.
# ansible-roles
