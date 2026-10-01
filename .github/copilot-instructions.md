# Copilot instructions — ansible-vpac

Ansible + RHEL image mode (bootc) for a Red Hat Edge **Virtual Protection Architecture Cluster (vPAC)**: RT-tuned KVM hosts running utility protection VMs (reference workload: ABB SSC600SW). Users are protection/substation engineers, not Ansible experts — docs are written for them (`docs/QUICKSTART.md`).

## Commands

```bash
ansible-galaxy collection install -r requirements.yml   # controller collections
pip install --user -r requirements.txt                  # passlib (password_hash with custom salt)

python3 tools/site-form.py        # web form at http://127.0.0.1:8765, writes inventory/<site>/ incl. encrypted vault

ansible-playbook -i inventory/<site> site.yml --tags preflight --ask-vault-pass
ansible-playbook -i inventory/<site> site.yml --ask-vault-pass
ansible-playbook -i inventory/<site> site.yml --tags validate --ask-vault-pass
ansible-playbook -i inventory/<site> site.yml --tags preflight-network   # one preflight sub-check

ansible-playbook tests/regression/test_linkdown_scope.yml   # run a single regression test
```

Regression tests need no infrastructure. A test passes when the final recap shows `failed=0`; mid-run `fatal: FAILED!` lines are expected for cases that assert a check *must* fire (caught by `rescue`). Only the final summary is authoritative.

Image mode: `image-mode/build.sh connected|airgapped <image-ref>` builds/pushes the bootc image and prints the `bootc-image-builder` recipe.

## Architecture

Two deployment paths share this repo and both are first-class:

- **Package mode** (`site.yml`): stock RHEL 9.7+/10.2+ converged by roles. `site.yml` imports `playbooks/NN-*.yml` in strict dependency order (00 preflight → 10 baseline → 20 networking → 30 virt → 40 PTP → 50 RT → 60 Ceph → 70 Pacemaker → 75 STONITH → 80 VMs → 90 validate). The stage number *is* the dependency contract; each stage has a tag. `op-*.yml` playbooks are day-2 operations, not part of `site.yml`.
  - `deployment_mode: airgapped | connected` switches package/image sources. Air-gapped adds `00-mint-builder-iso.yml` → `01-build-builder.yml` → `00b-mint-cluster-isos.yml` before `site.yml`; nodes then pull only from the builder's mirror/registry.
- **Image mode** (`image-mode/`): bootc image with kernel-rt (stock kernel removed — image must contain exactly one kernel), RT kargs in `kargs.d/`, static config in `files/`. Site identity is applied after boot (`image-mode/runtime/`). A redesign toward Red Hat Edge Manager is specified in `docs/superpowers/specs/`.

Inventory (`inventory/example/` is the template, default in `ansible.cfg`): everything lives in `group_vars/all/main.yml` (`vpac_nodes`, `networks`, `networking_defaults`, `rt_tuning`, `stonith`, `sources`, `vm_catalog`), secrets in `group_vars/all/vault.yml`, per-node values in flat `host_vars/<node>.yml`. Operators are meant to generate this via `tools/site-form.py`, not hand-edit it; `docs/OPERATOR-VALUES.md` documents every value.

## Invariants the code enforces (don't weaken them)

- **Heartbeat** (corosync) must be on a dedicated NIC/VLAN, never on the VM management bridge — bridge churn causes STP flaps → split-brain → same VM on two nodes.
- **PTP NIC** must not be a bridge member, bond slave, or macvtap target — otherwise guests swallow PTP frames.
- **Process bus** (GOOSE/SV) uses a dedicated NIC per relay via macvtap, no host IP.
- All nodes are identical; no inventory groups implying node roles. VM placement goes in `vm_catalog` (`target_host`, `allowed_hosts`).
- Isolated-core ranges must agree across kargs (`isolcpus`/`nohz_full`/`rcu_nocbs`), tuned `isolated_cores`, VM `vcpupin`/`emulatorpin`, and the `cyclictest` range.
- Checks must be scoped to interfaces the role manages (see `roles/networking/tasks/assert_linkdown_scope.yml` — libvirt's `virbr0` is expected after stage 30).

## Conventions

- Each role has a `README.md` documenting what it checks/changes, its variables, and sub-tags; update it with the role. `preflight` is read-only and has per-check toggles `preflight_check_*`.
- Logic worth regression-testing goes in its own `tasks/assert_*.yml` so production and `tests/regression/*.yml` include the *same* file.
- Roles refuse to overwrite files they didn't write (e.g. the qemu hook) and mask (not just disable) conflicting services like `irqbalance`.
- Commit messages: `<area>: <imperative, lower-case description>` (e.g. `preflight: accept RHEL 9.7+ or 10.2+`).
