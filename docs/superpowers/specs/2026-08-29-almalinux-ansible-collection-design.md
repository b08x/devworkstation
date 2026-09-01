# AlmaLinux 10 Ansible Collection — Design

**Date:** 2026-08-29
**Status:** Draft for review
**Author:** Robert Pannick (with Claude)

---

## Purpose

A published Ansible collection that brings an AlmaLinux 10 host to a known-good
state — baseline configuration, storage, networking, tuning, and hardening —
built and documented as an explicitly AI-assisted engineering effort.

Two goals, and they are equally weighted:

1. **A working artifact.** Something installable via `ansible-galaxy collection install`, that other people use.
2. **A checkable one.** Galaxy version history, CI status, and coverage against a published objective list are external evidence a stranger can verify without asking the author.

The second goal is the point. Per the companion spec
`2026-08-29-cv-eval-loop-design.md`, a claim audit of `cv.md` on 2026-08-29 found
that essentially every portfolio claim was vouched for only by its author. This
collection is designed to be the first artifact in the portfolio that isn't.

---

## Why this project, specifically

Measured 2026-08-29: 101 own GitHub repositories, of which **5 have a homepage and
6 have any stars.** The deficit is not output or focus — it is _completion_. Four
existing repos attack the same problem independently:

| Repo                               | Description                                                                 | Last push  |
| ---------------------------------- | --------------------------------------------------------------------------- | ---------- |
| `ansible-rhel-workstation-builder` | _"building custom Fedora/Rocky Linux workstations using Ansible"_           | 2026-05    |
| `pop_os-workstation-builder`       | _"Pop!\_OS workstation builder with 3-Layer governance architecture"_       | 2026-08-27 |
| `dots`                             | dotfiles / config management                                                | 2026-08-27 |
| `syncopated`                       | SyncopatedIaC / RaySession Linux-audio infrastructure (confirmed real work) | 2026-01    |

This collection consolidates that three-way duplication into one publishable
thing. It is mostly finishing work already done three times, not new work.

---

## Scope

### In

AlmaLinux 10 (EL10), server and workstation profiles from a shared base. Subject
matter bounded by the **RHCE (EX294) objectives** — the author holds RHCE/RHCSA
(2013), and the curriculum supplies a published, finite, externally-authored list.

**This is the definition of done.** "Various aspects of systems architecture"
cannot be finished. "Every RHCE objective implemented as a tested role on Alma 10"
can, and progress against it is legible to a reader who did not write the list.

### Out — deliberately

| Excluded                                                    | Reason                                                                                                                     |
| ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| Distro-agnostic support (Debian/Ubuntu/SUSE)                | Explicit author decision: _"it just becomes too many decisions after a while."_ Every conditional doubles the test matrix. |
| EL8 / EL9 backports                                         | Same. A "maybe EL9 too" door is how scope creep enters. Revisit only after v1.0.                                           |
| Reimplementing CIS/STIG benchmark remediation               | Occupied by mature projects — see Roles below.                                                                             |
| Cloud-provider-specific image building (AMI/qcow pipelines) | Adjacent, unbounded, and separable. The collection should _work_ on cloud hosts; it does not build images.                 |

---

## Architecture

Standard collection layout, current Ansible conventions throughout: FQCN
everywhere, `meta/argument_specs.yml` on every role, `meta/runtime.yml` declaring
the supported `ansible-core` floor.

```
b08x.el10/                       ← namespace/name pending (Open Question 1)
├── galaxy.yml
├── README.md                    ← RHCE objective coverage checklist
├── docs/
│   └── method.md                ← the AI-assisted build record (first-class)
├── roles/
│   ├── base/                    build
│   ├── storage/                 build
│   ├── network/                 build
│   ├── tuning/                  build
│   └── hardening/               integrate — see below
├── playbooks/
│   ├── workstation.yml
│   └── server.yml
└── extensions/molecule/         per-role scenarios
```

**Rebuild clean, reference the old.** Author decision. Logic and hard-won
EL-specific fixes are read out of the four existing repos; no code is copied
forward, so no structural debt is inherited.

---

## Roles

### `base`, `storage`, `network`, `tuning` — built

| Role      | Covers                                                 |
| --------- | ------------------------------------------------------ |
| `base`    | repositories, packages, users, sudo, SSH configuration |
| `storage` | LVM, filesystems, NFS                                  |
| `network` | bond interfaces, NetworkManager / nmcli                |
| `tuning`  | tuned profiles, sysctl, kernel parameters              |

Together these take a fresh Alma 10 host to a known-good state — the coherent
minimum that exercises both the workstation and server paths.

### `hardening` — integrated, not reimplemented

**Design change from the initial proposal.** `ansible-lockdown/RHEL10-CIS` and
`RHEL10-STIG-Audit` already exist and are actively maintained against the
benchmarks. Reimplementing CIS would compete with a mature project, and a partial
implementation labeled "CIS hardening" is actively misleading — a reader assumes
benchmark coverage that isn't there.

So `hardening` is an **opinionated profile layer** over the upstream project:

1. **Profiles** — workstation vs. server, with different control sets.
2. **Documented waivers** — which controls are disabled and _why_, in operational terms. This is the deliverable with the most of the author's 23 years in it, and it does not exist upstream.
3. **Post-hardening functional verification** — the genuine contribution.

**On (3):** upstream asserts that controls were _applied_. Nobody verifies the
host still _works_ afterward. The unowned territory is exactly the tension
between hardening and optimization:

- Does the bond interface still come up?
- Do NFS mounts still mount?
- Did the tuned profile survive?
- Does the workstation still boot to a usable session?

Molecule scenarios assert these _after_ hardening runs. That is the thing this
collection offers that no other EL10 collection does, and it falls directly out
of the author's stated interest in "workstation hardening **and** optimization."

---

## Testing

Molecule per role, plus an integration scenario running the full `server.yml` and
`workstation.yml` playbooks. Containers for fast per-role feedback; at least one
VM-backed scenario, because bonds, LVM, and kernel tuning cannot be honestly
tested in a container.

CI on GitHub Actions: lint (`ansible-lint`), syntax, molecule, and idempotence
(every role converges twice with zero changes on the second pass). Idempotence is
non-negotiable — it is the single clearest signal of a serious collection.

**The post-hardening functional suite is the highest-value test asset** and should
be built early, not last. It is the differentiator; if it slips, the collection is
ordinary.

---

## The AI-assisted method (`docs/method.md`)

A first-class deliverable, not a footnote. It records:

- Which parts were model-generated, which hand-written, which generated then rewritten.
- **Where models produced plausible-but-wrong Ansible.** EL10 is where this will happen most: training data skews heavily EL8/EL9, so models confidently emit deprecated module names, removed defaults, and pre-Wayland assumptions. Every such instance is a data point worth recording.
- The review gates that caught them — the verification instinct the author already applies via SIFT-style source checking, turned on his own workflow.
- An honest accounting of where AI assistance helped and where it cost more than it saved.

**Why this matters beyond the collection:** it converts "I use AI tools" — an
unfalsifiable claim of the exact kind the CV audit stripped out — into a
documented method with specific, checkable examples. It is the artifact that makes
the author's target archetype (AI-augmented systems engineering / support) real
rather than asserted.

It also has standalone value. A catalogue of EL10 migration breakage and where
LLMs get EL10 wrong is publishable on its own and cannot be written from a study
guide.

---

## Release Plan

**Ship v0.1.0 early and deliberately incomplete.** This is the most important
process decision in the document.

The failure mode is not building the wrong thing — it is never publishing. The
evidence is 101 repos and 5 homepages. Galaxy version history is the checkable
artifact, and it accrues nothing until something is tagged.

| Version    | Contents                                                                                                                      |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **v0.1.0** | `base` + `storage`, molecule + CI green, Galaxy published, README with the full RHCE checklist showing what is _not_ yet done |
| v0.2.0     | `network` + `tuning`                                                                                                          |
| v0.3.0     | `hardening` with profiles, waivers, and the functional verification suite                                                     |
| v0.4.0     | `docs/method.md` published                                                                                                    |
| v1.0.0     | RHCE objective coverage complete; API stable                                                                                  |

v0.1.0 ships **two** roles rather than five. The purpose of the first tag is to
prove the pipeline end to end — collection build, molecule, CI, Galaxy publish —
against the smallest possible payload. Adding roles to a working release process
is easy; retrofitting a release process onto five finished roles is where projects
die.

An honest README that says "3 of 20 objectives covered" is more credible than
silence, and it makes the next release visibly incremental rather than a rewrite.

---

## Risks

| Risk                                          | Mitigation                                                                                                |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| Never published (the historical pattern)      | v0.1.0 is deliberately two roles; publishing is the release gate, not completeness                        |
| Scope creep into distro-agnosticism           | Named as an explicit non-goal; any conditional on `ansible_os_family` outside EL is a review failure      |
| `hardening` drifts toward reimplementing CIS  | Role depends on upstream; if it starts writing its own controls, that is the signal it has gone wrong     |
| LLM-generated EL10 code is subtly wrong       | The reason molecule and idempotence checks are non-negotiable; also the raw material for `docs/method.md` |
| Container-only testing gives false confidence | At least one VM-backed scenario for bonds, LVM, and kernel tuning                                         |
| The four legacy repos linger and re-fragment  | On v0.1.0, archive them on GitHub with a README pointer to the collection                                 |

---

## Open Questions

1. **Namespace and name.** `b08x.el10`? `b08x.almalinux`? Galaxy namespace must match the author's Galaxy account. Renaming after publish is disruptive, so decide before v0.1.0.
2. **`syncopated` audio work.** Fold the Linux-audio provisioning in as a `workstation` profile option, keep it a separate downstream collection, or leave it out of scope for v1.0?
3. **VM backend for molecule** — Vagrant/libvirt, or containers plus a single CI VM job? Affects how much of the suite runs locally.
4. **License.** Ansible collections are conventionally GPL-3.0 or MIT; upstream `ansible-lockdown` is MIT. A dependency relationship makes this worth deciding deliberately.
5. **Does `dots` belong here at all?** Dotfile management is arguably a separate concern from system provisioning and may not fit the collection's boundary.

---

## Success Criteria

1. `ansible-galaxy collection install` works, from Galaxy, for a stranger.
2. CI is green and idempotence passes on every role.
3. The README shows coverage against RHCE objectives, including what is missing.
4. `docs/method.md` contains at least ten specific, cited instances of AI-generated Ansible being wrong on EL10 and how each was caught.
5. Post-hardening functional verification passes — the host still works after `hardening` runs.
6. The four legacy repos are archived and point here.
