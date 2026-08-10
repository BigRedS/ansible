# ansible

Partial, defaults-plus-overrides Ansible control for a handful of ordinary
Linux machines that are otherwise managed and fiddled with by hand. This
repo does **not** try to own everything about a host - only:

- `base_packages` - a shared apt package list (perl, vim, git, tmux, ...)
- `otelcol` - installs `otelcol-contrib` and points it at Coralogix
- `chezmoi` - installs the chezmoi binary and (optionally) bootstraps it from a dotfiles repo
- `ufw` - manages allow rules, with enabling/default-deny opt-in per host
  (currently commented out of `site.yml` - not applied to any host yet)

Each role ships sane shared defaults; each host overrides only what's
different about it. Nothing here assumes a clean-slate machine - roles are
written to be safe to run against a box that's already been configured by
hand (see "Safety notes" below).

## Layout

```
ansible.cfg          # inventory path, safe defaults
site.yml              # entrypoint, applies roles to linux_boxes (ufw commented out for now)
inventory/hosts.yml    # add your real hosts here
group_vars/all/        # shared defaults (main.yml) + gitignored secrets (vault.yml)
host_vars/              # per-host overrides, one file per hostname
roles/
  base_packages/
  otelcol/
  chezmoi/
  ufw/
```

## First-time setup

```bash
ansible-galaxy collection install -r requirements.yml

# add real hosts
$EDITOR inventory/hosts.yml

# create the secrets file - group_vars/all/vault.yml is gitignored, so it's
# kept in cleartext rather than ansible-vault encrypted. Fine for a
# single-user, not-pushed-anywhere repo like this one; if that stops being
# true, `ansible-vault encrypt` it instead and drop it from .gitignore.
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
$EDITOR group_vars/all/vault.yml   # set otelcol_coralogix_private_key

# per host, copy and edit an overrides file
cp host_vars/example-host.yml.example host_vars/<real-hostname>.yml
$EDITOR host_vars/<real-hostname>.yml
```

## Running

```bash
# everything
ansible-playbook site.yml

# just one aspect, on one host
ansible-playbook site.yml --tags otelcol --limit <hostname>

# see what would change without applying it
ansible-playbook site.yml --check --diff
```

Use `--check --diff` liberally on machines that are also managed by hand -
it's the cheapest way to confirm ansible isn't about to fight someone's
manual changes before you actually run it.

## The override pattern

Role defaults live in `roles/<role>/defaults/main.yml` - the lowest-precedence
variable layer. A `host_vars/<hostname>.yml` file can reference the same
variable name to *extend* rather than replace it, because the default is
still what resolves inside the template:

```yaml
# host_vars/my-desktop.yml
base_packages: "{{ base_packages + ['htop', 'ncdu'] }}"
ufw_rules: "{{ ufw_rules + [{'rule': 'allow', 'port': '8080', 'proto': 'tcp'}] }}"
```

Or override outright by just assigning a plain value instead of extending.

**Caveat:** this self-referencing extend pattern (`foo: "{{ foo + [...] }}"`)
throws `Recursive loop detected in template` on ansible-core 2.19.4 (the
version currently installed here) - confirmed with a minimal repro outside
this repo's roles, so it's not specific to `base_packages`/`ufw_rules`.
Until that's resolved (older ansible-core, or a different mechanism),
override outright with the full value instead - see `host_vars/donkey.yaml`
for an example that assigns `otelcol_filelog_pipelines` directly rather
than extending it.

## Safety notes (partial control, on purpose)

- **ufw**: allow rules are always applied (additive, harmless), but setting
  default-deny (`ufw_manage_defaults`) and actually enabling ufw
  (`ufw_enable`) are both off by default. Flip them per-host only once
  you're ready to hand that host's firewall fully to ansible - otherwise
  this role is a no-op beyond making sure your allow-list is present.
- **chezmoi**: only installs the binary by default. It will run
  `chezmoi init --apply` exactly once (guarded on the source directory not
  yet existing) if you set `chezmoi_source_repo` - it will not keep re-
  applying and clobbering changes made by hand afterwards.
- **otelcol**: installs a specific pinned version
  (`otelcol_version`) from the upstream GitHub releases and fully owns
  `/etc/otelcol-contrib/config.yaml`. This one *is* meant to be fully
  ansible-managed; there's no natural "partial" state for a collector config.

## Things to double check before relying on this

- `otelcol_coralogix_domain` in `group_vars/all/main.yml` - this needs to
  match your Coralogix account's actual ingestion region/domain; it wasn't
  verified against live Coralogix docs when this repo was scaffolded.
- The otelcol config template ships with a `hostmetrics` metrics pipeline
  by default, plus opt-in logs sources: set `otelcol_filelog_pipelines`
  (empty by default) in a host's `host_vars/<hostname>.yml` to tail specific
  log files, and/or `otelcol_enable_journald: true` to ship the systemd
  journal (syslog/auth/etc.) directly - useful on Debian boxes that don't
  keep flat files like `/var/log/syslog`. See
  `host_vars/example-host.yml.example` for the shape of both, including
  routing individual sources to their own Coralogix application/subsystem.
