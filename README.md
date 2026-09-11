# b08x.devworkstation

This is the collection I use to turn a fresh Fedora or AlmaLinux install into a
working developer machine. It handles the system baseline, the desktop,
virtualisation, container runtimes, realtime audio tuning, and the AI coding
agents — the things I'd otherwise spend a weekend setting up by hand and then
forget how I did.

I run it against two machines that look nothing alike: `gir`, a Dell Precision
5560 laptop on Fedora 43, and `tinybot` on AlmaLinux 10.2. That's why the
distro-switching is real rather than aspirational — both paths get exercised
every time I touch this.

<!--start requires_ansible-->
<!--end requires_ansible-->

## Here's what's here

Nine roles, seven of which do real work:

| Role | What it handles |
| :--- | :--- |
| `base` | Packages, repos, zsh, Go, fzf, yadm, Homebrew, gitflow, Intel oneAPI |
| `user` | My per-user toolchains — cargo, pip, rbenv, uv, ranger |
| `desktop` | Desktop environment, GNOME, Qt/GTK theming, Flatpaks, input-remapper |
| `tuning` | Realtime audio — sysctl, limits, udev timers, rtirq, rtkit, cpupower |
| `libvirt` | libvirt, KVM, virtual networking, LVM storage |
| `containerd` | Podman, Docker, distrobox |
| `coding_agents` | Antigravity, Claude, Crush, OpenCode, Vibe |
| `networking` | Nothing yet — scaffold stub |
| `run` | Nothing yet — scaffold stub |

## Before you start

You'll need `ansible-core` 2.15 or newer, and a target running **Fedora** or
**AlmaLinux**. Those are the only two distributions I've written `vars/` and
`tasks/distro/` coverage for; anything else fails when a role tries to load a
vars file that doesn't exist.

You'll also want three collections installed:

```bash
ansible-galaxy collection install ansible.utils community.general ansible.posix
```

I've only declared `ansible.utils` in `galaxy.yml`. The other two get used for
real — `community.general` for flatpak, ini_file, cargo, gem and uv, and
`ansible.posix` for sysctl — so install them yourself until I fix the manifest.

## Getting it

I usually pull it in as a git dependency:

```yaml
# requirements.yml
collections:
  - name: https://github.com/b08x/devworkstation.git
    type: git
    version: development
```

```bash
ansible-galaxy collection install -r requirements.yml
```

If you're vendoring it into a control repo, a submodule keeps this collection's
history separate from your inventory:

```bash
git submodule add https://github.com/b08x/devworkstation.git \
  collections/ansible_collections/b08x/devworkstation
git submodule update --init --recursive
```

The Galaxy path (`ansible-galaxy collection install b08x.devworkstation`) isn't
live — `galaxy.yml` still carries scaffold metadata, so there's nothing published
to install yet.

## Included content

<!--start collection content-->
<!--end collection content-->

## Running it

I split the play in two. The system roles need `become`; the `user` role
specifically does not.

```yaml
- name: System provisioning
  hosts: workstations
  become: true
  roles:
    - role: b08x.devworkstation.base
    - role: b08x.devworkstation.tuning
    - role: b08x.devworkstation.containerd

- name: User setup
  hosts: workstations
  become: false
  roles:
    - role: b08x.devworkstation.user
```

That split matters more than it looks. The `user` role writes into `$HOME`, and
running it under `become` hands you a home directory owned by root — recoverable,
but not how I'd choose to spend an afternoon.

### Picking what runs

Tags nest. A role-level tag pulls in everything underneath it, and the subsystem
tags narrow from there.

```bash
# Everything, one host
ansible-playbook site.yml --limit tinybot

# Just the audio tuning
ansible-playbook site.yml --tags tuning

# Just one piece of it
ansible-playbook site.yml --tags rtirq

# Container runtimes, but skip the Docker-specific work
ansible-playbook site.yml --tags containerd --skip-tags docker

# Pull every coding agent back off
ansible-playbook site.yml --tags coding_agents -e coding_agents_uninstall=true

# See what would change, without changing it
ansible-playbook site.yml --tags base --check --diff
```

The subsystem tags available to you:

- **base** — `dnf`, `repos`, `packages`, `zsh`, `go`, `fzf`, `yadm`, `gitflow`, `homebrew`, `intel`, `inxi`
- **user** — `cargo`, `pip`, `rbenv`, `ranger`, `uv`
- **desktop** — `gnome`, `flatpaks`, `flathub`, `input-remapper`, `vscode`
- **tuning** — `sysctl`, `limits`, `udev`, `rtirq`, `tuned`, `cpupower`, `groups`
- **libvirt** — `install`, `config`, `service`, `users`, `lvm`, `network`
- **containerd** — `podman`, `docker`, `distrobox`
- **coding_agents** — `antigravity`, `claude`, `crush`, `opencode`, `vibe`, each pairing with `install`, `config` or `uninstall`

## Configuring it

Role-internal variables carry a `<role>_` prefix. Host-level things like `user.*`
stay unprefixed, because they describe the machine and the person using it rather
than any one role — which is why I skip `var-naming[no-role-prefix]` in
`.ansible-lint` deliberately rather than out of neglect.

**Container runtimes**

- `containerd_use_podman`: Install Podman (default: `true`)
- `containerd_use_docker`: Install Docker (default: `false`)
- `containerd_docker_users`: Users added to the `docker` group
- `containerd_docker_storage`: Docker data root (default: `/var/lib/docker`)
- `containerd_docker_daemon_config`: Dict rendered into `daemon.json` (default: `{}`)

**libvirt**

- `libvirt_users`: Users added to the `libvirt` group
- `libvirt_unix_sock_group`: Socket group (default: `libvirt`)
- `libvirt_unix_sock_rw_perms`: Socket permissions (default: `0770`)
- `libvirt_configure_lvm`: Provision LVM-backed storage (default: `true`)

**Audio tuning**

- `tuning_sysctl`: Dict of sysctl keys, applied as a loop
- `tuning_limits`: Realtime limits entries
- `tuning_udev_timers`: Timer device permission rules
- `tuning_rtirq_enabled` / `tuning_rtkit_enabled`: Enable each (default: `true`)
- `tuning_tuned_profile`: tuned profile (default: `throughput-performance`)

**Coding agents**

- `coding_agents_<agent>_state`: `present` or `absent`, per agent (default: `present`)
- `coding_agents_uninstall`: Take all of them off (default: `false`)
- `coding_agents_<agent>_config_dir` / `_skills_dir`: Per-agent directories

## Known issues

I track these as `TODO` comments in the code, so they turn up where you'd
actually hit them:

```bash
grep -rn "TODO (P" roles/
```

**Broken right now**

| Priority | Where | What's wrong |
| :--- | :--- | :--- |
| P0 | `desktop/vars/Fedora.yml` | I only defined `theme`, but the tasks loop `desktop_packages.qt`, `.gtk` and `.gnome`. On Fedora those raise a missing-attribute error, the surrounding rescue swallows it, and the run goes green having installed no Qt, GTK or GNOME packages at all. |
| P0 | `desktop/tasks/input-remapper.yml` | `python3 -m install` isn't a valid module invocation. Fails in check mode. |
| P1 | `containerd/templates/etc/docker/daemon.json.j2` | Renders `virt_docker_daemon_config`, which nothing defines. The real variable is `containerd_docker_daemon_config`, so `daemon.json` never gets written. |
| P1 | `tuning/templates/etc/default/cpupower.j2` | Ignores `tuning_cpupower_governor` and the min/max frequency variables entirely — I hardcoded the governor and gated the frequency limits on literal hostnames. |
| P1 | `base/tasks/distro/Fedora.yml` | Empty file, so third-party repo setup is a silent no-op on Fedora. |
| P1 | `tuning/tasks/sysctl.yml` | `failed_when: false` masks every sysctl failure, so a misspelled key reports success. |
| P2 | `coding_agents/tasks/antigravity.yml` | Fetches an install script with `validate_certs: false`, then runs it as root. |

The cpupower one is worth spelling out, because it's the kind of bug that hides
in plain sight. My `host_vars/gir.yml` asks for `tuning_cpupower_max_freq:
3100MHz`, but the template only names `soundbot` and `ninjabot`, so gir matches
neither branch and gets no limits written at all. `inxi -Fx` on that machine
reports `min/max: 800/4600` — it has been running 1.5 GHz over the cap I thought
I'd set, quietly, for as long as those variables have existed.

**Rougher edges, not tracked in code**

- `networking` and `run` are untouched ansible-creator scaffolding. They print a
  debug message and nothing else. No playbook wires them up.
- `meta/argument_specs.yml` documents `*_my_variable` rather than the real
  options, so role argument validation is decorative. `coding_agents`,
  `containerd` and `libvirt` have no spec at all.
- Everything under `plugins/` is sample content from the scaffolder.
- Several blocks in `base` and `desktop` rescue into a `debug` task, which means
  a failed install still reports green. The two P0 entries above bite hardest,
  but the pattern is wider than that.

## Testing

```bash
ansible-lint
molecule test
pytest -vvv -n 2
```

Worth knowing: this collection's own pre-commit config doesn't include
`ansible-lint`, so `pre-commit run --all-files` here isn't the same as linting.
Run it directly.

## Contributing

Roles use the standard layout — `tasks/`, `defaults/`, `vars/`, `meta/`,
`handlers/` — with distribution-specific work in
`tasks/distro/{{ ansible_distribution }}.yml`, and firewall rules living in
whichever role opens the port rather than in a central firewall role.

If you're adding a subsystem, give it its own file and a tagged `include_tasks`
entry in the role's `tasks/main.yml`. Don't inline it.

## License

GNU General Public License v3.0 or later. Full text in [LICENSE](LICENSE).
