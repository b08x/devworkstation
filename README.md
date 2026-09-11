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
| `desktop` | Desktop environment, GNOME, Qt/GTK theming, Flatpaks, VS Code, Antigravity Hub/IDE, input-remapper |
| `tuning` | Realtime audio — sysctl, limits, udev timers, rtirq, rtkit, cpupower |
| `libvirt` | libvirt, KVM, virtual networking, LVM storage |
| `containerd` | Podman, Docker, distrobox |
| `coding_agents` | Antigravity CLI, Claude, Crush, OpenCode, Vibe |
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
- **desktop** — `gnome`, `flatpaks`, `flathub`, `input-remapper`, `vscode`, `antigravity`
- **tuning** — `sysctl`, `limits`, `udev`, `rtirq`, `tuned`, `cpupower`, `groups`
- **libvirt** — `install`, `config`, `service`, `users`, `lvm`, `network`
- **containerd** — `podman`, `docker`, `distrobox`
- **coding_agents** — `antigravity` (CLI only; the Hub and IDE are tagged `antigravity` under **desktop**), `claude`, `crush`, `opencode`, `vibe`, each pairing with `install`, `config` or `uninstall`

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

**Base — rescue strictness**

Three blocks in `base` are best-effort: they warn and continue when they fail,
because the role still delivers what it promises without them. Set one to `true`
to turn its failure into a play failure.

- `base_intel_graphics_required`: Intel graphics drivers (default: `false` — the
  packages only apply to Intel hardware)
- `base_zoxide_required`: zoxide (default: `false` — a shell convenience)
- `base_dnf_priorities_required`: DNF repository priorities (default: `false` —
  DNF still resolves packages with its default ordering)

Every other rescue in the collection either re-raises or performs a real
fallback. See the rescue convention under [Contributing](#contributing).

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

The P0 and P1 defects that used to be listed here are fixed, and no
`TODO (P...)` markers remain in `roles/`.

**Rougher edges, not tracked in code**

- `networking` and `run` are untouched ansible-creator scaffolding. They print a
  debug message and nothing else. No playbook wires them up.
- `meta/argument_specs.yml` documents `*_my_variable` rather than the real
  options, so role argument validation is decorative. `coding_agents`,
  `containerd` and `libvirt` have no spec at all.
- Everything under `plugins/` is sample content from the scaffolder.

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

**Rescue blocks must recover or re-raise, never just log.** A `rescue:` whose
only action is `debug` converts a failure into a green play, which is how the old
package defects went unnoticed for so long. Every rescue here carries a comment
naming which of three shapes it is:

1. **Add context, then re-raise** — a `debug` explaining the likely cause,
   followed by `ansible.builtin.fail`. This is the default for anything the role
   promises to deliver.
2. **Real fallback** — the rescue satisfies the same contract another way, as in
   `base/tasks/inxi.yml` (dnf → standalone script) and `user/tasks/ranger.yml`
   (dnf → pipx).
3. **Documented optional dependency** — the block is genuinely best-effort and
   the contract holds without it. These must be gated behind a
   `<role>_<feature>_required` default so an operator can opt into strictness.

Don't reach for a blanket `ignore_errors` instead, and don't `set_fact` a
`*_failed` flag that nothing reads.

## License

GNU General Public License v3.0 or later. Full text in [LICENSE](LICENSE).
