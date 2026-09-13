# ansible

Some ansile for the common-to-everything stuff on my computers:

* `ansible_svc_user` - creates a dedicated `ansible` user on each host for future ansible-connection bootstrapping. A human must set a password by hand
* `apache` - currently just used to keep Apache on ipv4 on donkey, leaving the ipv6 interface for k3s
* `base-packages` - apt package list, from Debian repos
* `chezmoi` - installs chezmoi, still need to manually set it up with ssh keys and github and whatnot
* `docker_compose_ipv4_ports` - just keeps docker-compose stuff on ipv4 on donkey, leaving ipv6 for k3s
* `otelcol` - set up OpenTelemetry Collector to send hostmetrics to Coralogix; optionally (`otelcol_enable_otlp_receiver`) also accepts OTLP from the network on the standard ports and forwards that on too
* `tailscale` - installs tailscale, doesn't try to configure, but probaly will cause an connection-drop on upgrade
* `ufw` - firewall; rules are additive/always-applied but default-deny + actually enabling it are opt-in per host (`ufw_manage_defaults`/`ufw_enable`, both off by default) - enabled on `donkey` (see host_vars). Also sets `ufw_quiet_console` (on by default, everywhere) to stop kernel warning-level messages (blocked packets included) spamming every tty/console

and some common-to-PCs stuff:

* `claude_code` - claude code, and then also `nodejs` to get the skills
* `coralogix_cli`
* `flatpak` - installs flatpak, adds the flathub remote
* `k8s_tools` - k8s tools, plus krew and a fixed set of krew plugins (see `k8s_tools_krew_plugins`)
* `opentofu` - installs opentofu

and some host-specific stuff:

* `nfs_server` - exports directories over NFS (`nfs_server_exports`), applied to the `nfs_servers` inventory group
* `k3s` - installs an ipv6-only k3s server (`k3s_node_ip`/`k3s_cluster_cidr`/`k3s_service_cidr`, all required), applied to the `k3s_servers` inventory group. Also fetches+merges its kubeconfig into the control node's `~/.kube/config` under a given name (`k3s_kubeconfig_name`, no-op if unset) - tagged `kubeconfig` so it can be re-run on its own

## Run Ansible

Set up an environment with the **ansible** user's passord:

    source ./ansible-shell

Then run ansible, there's no need to use -K. First check:

    ansible-playbook site.yaml --tags <tags> --limit <hostname> --check --diff

Then run it for real:

    ansible-playbook site.yaml --tags <tags> --limit <hostname>

## Authentication

Sourcing `ansible-shell` prompts for a sudo password and then creates aliases for ansible commands that set this password.

This is the password for the `ansible` user on each of the hosts, _not_ `avi`.

## Adding a new host

Create a `host_vars/<hostname>.yaml` by copying an existing one, but add an override to use the `avi` user:
```yaml
ansible_user: avi
ansible_ssh_private_key_file: ~/.ssh/id_ed25519
```
then run ansible with `--tags ansible_svc_user --limit <newhost> -K` to create an ansible user and set the nopasswd sudo (grab it from bitwarden)

Set the account's password by hand (`sudo passwd ansible`, from bitwarden), and remove the override


## Potential Beatraps (found by claude):

- **ansible-shell**: feeds the become password via a short-lived mode-600
  temp file (`_ansible_become_password_file`), not process substitution -
  ansible-core can't open a `<(...)`'s resolved `/proc/.../fd/pipe:[N]` path
  ("password file ... was not found").
- **ansible_svc_user**: only `donkey` sets `PasswordAuthentication no`
  globally, so the role enforces key-only login for just the `ansible`
  account via a `Match User` drop-in in `/etc/ssh/sshd_config.d/` (assumes
  the stock `Include` line for that directory is still present). The
  account is created locked and never touched again after - set its
  password by hand (`sudo passwd ansible`), same value on every host.
- **Raspberry Pi OS** (`fairygodmother`) ships `/etc/sudoers.d/010_pi-nopasswd`
  (NOPASSWD for the first-boot user), baked into the image by `userconf-pi`,
  not by ansible or this repo. Commented out on the host directly; don't
  rely on it regardless - `ansible.cfg` never assumes passwordless sudo.
- **otelcol**'s OTLP receiver (`otelcol_enable_otlp_receiver`) binds
  `0.0.0.0:4317`/`4318` with no auth in front of it - anything on the
  network can feed it telemetry that gets forwarded to Coralogix under that
  host's account. No host has this on currently; add a `ufw` rule before
  ever turning it on.
- The self-referencing extend pattern (`foo: "{{ foo + [...] }}"`) throws
  "Recursive loop detected in template" on the installed ansible-core -
  assign the full value directly instead (see `host_vars/donkey.yaml`'s
  `otelcol_filelog_pipelines`).
- **tailscale**: installs/upgrades only, doesn't manage tailnets - upgrading
  while connected through it drops the connection. Reuses a host's existing
  apt source for the repo if one's already there, rather than adding a
  second, differently-keyed one (which makes apt hard-error).
- **ufw**: sets its own `update_cache`/`cache_valid_time`, independently of
  `base_packages` - needed because `--tags ufw` alone skips that role.
  `ufw_quiet_console` (on by default) drops the kernel's console log level
  so `[UFW BLOCK]` messages stop hitting every tty - a `kernel.printk`
  setting, not syslog, and doesn't affect what's logged elsewhere.
- **k8s_tools/krew**: installed per-user (`~/.krew`), unlike the rest of the
  role. Ansible doesn't add `~/.krew/bin` to `PATH` - do that by hand or via
  chezmoi.
- **chezmoi**: `chezmoi_user` has no default and must be set explicitly
  alongside `chezmoi_source_repo` - it used to default to `ansible_user`,
  which broke the moment that stopped meaning "the human operator".
- otelcol ships a `hostmetrics` pipeline by default; logs are opt-in via
  `otelcol_filelog_pipelines` (tail specific files) and/or
  `otelcol_enable_journald: true` (ship the systemd journal).
- `k8s_tools_k9s_version`/`k8s_tools_kubectx_version` - pinned GitHub
  releases, no apt repo to track: bump by hand
  ([k9s](https://github.com/derailed/k9s/releases),
  [kubectx](https://github.com/ahmetb/kubectx/releases)).
- `k8s_tools_helm_version` - installed from `get.helm.sh`, not helm's own
  apt repo: that repo's TLS chain roots at a CA Debian trixie's
  `ca-certificates` doesn't carry yet (as of 2026-08). Revisit once fixed;
  bump by checking [helm releases](https://github.com/helm/helm/releases).
- `nodejs_major_version` - pinned Node major; bump by hand for a new one.
- `coralogix_cli_version` - pinned GitHub release, bump by hand. `cx
  profiles add` still needs running by hand per client - not
  ansible-managed.
- `nfs_server`: only applied to the `nfs_servers` group, not all of
  `linux_boxes`. No firewall rule opens NFS's ports - add one before
  relying on this beyond a trusted network.
- `nfs_server` on `fairygodmother`: a leftover `cockpit-file-sharing` export
  file used to break `exportfs -ra` (an invalid byte in its output tripped
  ansible's UTF-8 check) - fixed by deleting it and folding that export
  into `nfs_server_exports` directly.
- `apache`/`docker_compose_ipv4_ports` on `donkey`: donkey's Apache wasn't
  installed by ansible (`nginx-common`/`/etc/nginx` is an unrelated orphaned
  remnant) - these roles just pin Apache's `Listen` address and gogs'
  compose ports to IPv4, freeing the IPv6 address for k3s. Applying either
  restarts Apache/recreates the gogs container, briefly taking down every
  vhost on the box.
- `k3s`: IPv6-primary - cluster/service CIDRs are a ULA range NAT'd out via
  `flannel-ipv6-masq`, `k3s_node_ip` is the host's real public IPv6.
  Optionally dual-stack for pod *egress* only (`k3s_node_ip_v4`, e.g. to
  reach IPv4-only external APIs). CoreDNS forwards via a k3s-specific
  `resolv.conf` of IPv6-only public resolvers, since the pod network has no
  IPv4 route out. A sysctl (`net.ipv6.conf.all.accept_ra = 2`) stops k3s
  losing its IPv6 default route after install.
- `k3s_disable`/`k3s_allow_unprivileged_port_bind` (`donkey`): k3s's bundled
  Traefik+ServiceLB claims a Service's `hostPort` with no `hostIP` set,
  which binds *both* address families on the node regardless of the
  Service's own `ipFamilies` - no annotation/flag scopes it to one family
  (checked against k3s's `servicelb.go` source). Pinning the Service to
  IPv6-only at the Helm-values level (an earlier approach here, since
  removed) only constrains the ClusterIP - it doesn't stop ServiceLB's
  `hostPort` claim from grabbing IPv4 too, conflicting with apache. Fix:
  disable both bundled components outright (`k3s_disable`) and run your own
  Traefik instead - see the `farfaraway` cluster repo's `traefik/` for the
  replacement, which needs `hostNetwork: true` to bind a specific host IP
  (a plain `hostPort`+`hostIP` on the container port isn't enough - the
  chart also feeds `hostIP` into Traefik's own entrypoint bind address,
  which only resolves inside the pod's own netns if that's actually the
  host's). Non-root + `hostNetwork` then can't bind ports <1024 at all -
  the obvious fix, `securityContext.capabilities.add: [NET_BIND_SERVICE]`,
  is a silent no-op due to a long-standing Kubernetes bug
  ([kubernetes/kubernetes#56374](https://github.com/kubernetes/kubernetes/issues/56374)),
  and a pod-level `sysctls` override for
  `net.ipv4.ip_unprivileged_port_start` is flatly rejected by the API
  server for `hostNetwork` pods (no separate netns to scope it to) - the
  only remaining fix is host-wide, hence this ansible var.
- `k3s` kubeconfig fetch/merge: k3s bakes a loopback address (`127.0.0.1` or
  `[::1]`, depending on IP family) into its generated kubeconfig, and its
  cert is only valid for `k3s_node_ip`/`k3s_tls_san` - `k3s_kubeconfig_server_host`
  plus an entry in `k3s_tls_san` point both at the host's Tailscale IP
  instead. Existing kubeconfig entries (and whichever is `current-context`)
  are left alone; only a same-named entry gets overwritten.
