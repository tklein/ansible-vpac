# Bootc VM Delivery Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Independently deliver and update SSC600SW and Windows VM golden disks on the bootc host without losing per-instance identity or configuration.

**Architecture:** A host-side `vpac-vm@.service` reconciles OCI-packaged disks and declarative VM values. Shared libvirt templates preserve guest device contracts, persistent state lives on local `/var`, and candidate updates are prepared before stopping the currently running VM. VM/config recovery is separate from bootc OS rollback.

**Tech Stack:** Host plan's Python/Jinja/schema/test infrastructure, Podman, qemu-img, libvirt/KVM, virtio/virtiofs, swtpm/UEFI, RHEM 1.3 configuration delivery.

**Spec:** `docs/superpowers/specs/2026-10-01-bootc-edge-manager-design.md` at `c0bd480`; host prerequisite plan: `docs/superpowers/plans/2026-10-05-bootc-host-implementation.md`.

## Global Constraints

- **Workloads:** ABB SSC600SW (RT VM), Windows engineering workstation (non-RT VM).
- **Statelessness:** Immutable golden disk + small persistent data volume.
- **PTP:** Dedicated onboard NIC; enabled by default, disable-able per device.
- **Coexistence:** Package mode stays first-class side by side.
- No Ansible after image-mode deployment, no distributed storage, no live migration of SSC600.
- Virtio guest NICs; bridge/macvtap host attachment is separate from guest model. Preserve three SSC600 NICs and their vendor ordering until S3 confirms roles.
- Stable, distinct, explicitly supplied MACs per interface; no runtime-generated MAC identity.
- Raw vendor disks are supported; never infer format from extension. Backing format must be explicit for overlays.
- SSC600 configuration/license persistence is a validation gate before any overlay reset or destructive update.
- Preserve per-instance Windows UEFI NVRAM and TPM state where used, as well as data disks.
- Keep current disabled reset-watchdog policy unless vendor/lab evidence and user approval justify a change.
- RHEM rollout of OS images does not automatically schedule independent VM/config updates.
- Documentation and covering tests ship in every task commit; coverage uses the host plan's branch-enabled 95% aggregate gate plus every recovery/destructive path tested.
- Never ship vendor disks, real enrollment data, registry auth or per-unit identities in test fixtures or reusable host-image layers.
- User's untracked design `copy.md` is out of scope.

---

## Execution and dependencies

Run after the host plan, using one fresh implementer and a task-scoped spec/quality review per task, with a final whole-branch review. Host contract modules and shared sources are prerequisites, not recreated here. An implementer sees their task brief plus predecessor interfaces, never a history dump.

| Task | Prerequisites | Deliverable |
|---|---|---|
| 1 | Host Tasks 1, 2 | Vendor/guest persistence and RT acceptance evidence gates |
| 2 | Host Tasks 2, 7; VM 1 for vendor packaging approval | Validated OCI disk bundles and verified local extraction |
| 3 | Host Tasks 3, 4, 6; VM 1, 2 | Guest XML/profile contract with fixed NIC identities |
| 4 | Host Tasks 5, 7; VM 1-3 | Safe disk/state lifecycle with independently tested recovery |
| 5 | Host Tasks 8, 9; VM 4 | Host VM units, coordinated reconciliation and status |
| 6 | VM 1-5 | Combined OS/VM lifecycle and vendor workload acceptance |

Hardware-free tests and synthetic guest boot can proceed without a vendor disk. Vendor acceptance and destructive reset-on-upgrade cannot. For insufficient vendor evidence, return a blocked task with the exact missing facts; do not implement a pretend persistent data disk or guess ABB's settings format.

## File map and shared interfaces

| Path | Responsibility |
|---|---|
| `image-mode/workloads/` | Bundle format, packaging tool and public descriptor examples; no disks |
| `image-mode/schema/workload-v1.json` | OCI descriptor validation |
| `image-mode/runtime/vpac/bundles.py` | Pull/resolve/stage bundles without executing their image |
| `image-mode/runtime/vpac/guest.py` | Pure guest context/rendering and resource validation |
| `image-mode/runtime/vpac/disks.py` | Overlay, persistent volume and identity ownership |
| `image-mode/runtime/vpac/vm.py` | Explicit update state machine and libvirt operations |
| `image-mode/runtime/vpac/vm_cli.py` | Thin command interface |
| `image-mode/runtime/vpac/workloads.py` | Desired/observed membership and rollout coordination |
| `tests/image_mode/` | Guest, lifecycle, failure and systemd behavior coverage |

Reuse host `Runner`, `CommandResult`, `ConfigError`, `ApplyError`, `write_status`, `load_node`, `validate_pair`, `render_host` and `desired_hash`. Shared templates remain under `shared/templates/`.

`node.yaml` VM entry contract (freeze in host Task 2):

```yaml
name: ssc600
image: registry.example/vpac-vm-ssc600@sha256:<64-hex-digest>
enabled: true
diskMode: reset-on-upgrade
macs:
  management: "52:54:00:aa:bb:01"
  process: "52:54:00:aa:bb:02"
  service: "52:54:00:aa:bb:03"
# Optional: dataDisk: {sizeGiB: 20}
# Optional: configSource: /etc/vpac/guest-config/ssc600
```

The digest syntax above describes the field, not an executable placeholder fixture. Tests use a public synthetic full 64-hex digest. `station` in the original illustrative spec is an alias only if S3 proves the corresponding vendor NIC role; normalize once and reject contradictory aliases. This is a schema clarification to capture in docs, not automatic remapping of a live IED.

`name`: `[a-z][a-z0-9-]{0,47}`; no shell/service/path metacharacters. `image`: registry-qualified OCI reference, immutable digest preferred; tags are resolved and recorded before actions. `diskMode`: `ephemeral` or `reset-on-upgrade`. `dataDisk`: optional positive integer GiB, never shrunk/recreated silently. `configSource`: optional absolute path under the managed guest-config directory, no secrets in labels, injection only when the guest supports the declared mechanism. Unknown properties fail.

**Important:** host readiness with `vms: []` is already testable. Until this runtime exists, host reconciliation rejects enabled entries. This plan removes that guard only when units and lifecycle support are installed.

## Task 1: Establish SSC600 and Windows compatibility/persistence gates

**Files:**
- Create: `image-mode/workloads/README.md`, `image-mode/workloads/COMPATIBILITY.md`
- Create: `image-mode/workloads/acceptance-contract.json`
- Create: `tests/image_mode/test_workload_acceptance_contract.py`
- Create: `image-mode/runtime/vpac/guest.py` (persistence validator, extended in Task 3)
- Modify: `image-mode/ACCEPTANCE.md`

**Interfaces:**
- Acceptance contract records vendor bundle version/hash, guest device role mapping, persistence mechanism, licensing assumption status, RHEL support status and measurable RT thresholds.
- Never add an `approved` flag that alone authorizes destructive operations; runtime enforces structural persistence requirements too.

- [ ] **Step 1: Write evidence-shape tests before defining acceptance records.**

```python
def test_reset_mode_needs_verified_settings_restore():
    from vpac.guest import validate_persistence_contract
    from vpac.errors import ConfigError
    contract = {"settings": "system-disk", "restoreMechanism": None}
    with pytest.raises(ConfigError, match="settings"):
        validate_persistence_contract(contract, disk_mode="reset-on-upgrade")
```

Task 1 creates this small pure validator in `image-mode/runtime/vpac/guest.py`; Task 3 extends the same module. Document it in the file list during execution. Validate explicit mechanism types/required artifacts, not one boolean.

- [ ] **Step 2: Run the test; confirm absence of validator.**
- [ ] **Step 3: Implement evidence validation and document S3 experiment.** On an authorized lab host, inspect the ABB bundle checksums, actual raw disk, supported guest NIC roles, PTP status script/output and supported settings/licensing export/import. Test that settings survive the intended operation using a backed-up disposable instance. Do not extract proprietary script contents into this repository.
- [ ] **Step 4: Resolve watchdog and disk I/O as explicit lab decisions.** Current package profile uses watchdog disabled/native disk I/O; Welotec uses i6300esb reset/threads. Keep package mode unchanged, retain disabled-reset policy for image mode initially, and evaluate differences with guest/vendor evidence rather than relabeling either choice as universally safe.
- [ ] **Step 5: Define Windows variant/driver/firmware requirements.** Identify installed virtio drivers, UEFI/Secure Boot/NVRAM/TPM needs, guest configuration method (sysprep/config media/manual), and which local state must survive. Golden disk licensing remains a documented lab limitation until resolved.
- [ ] **Step 6: Record numerical RT acceptance contract from vendor/project requirements before claiming acceptance.** Include cyclictest maximum latency, test duration, GOOSE/SV traffic rates, permitted loss/latency/jitter and clock accuracy. If these values are unavailable, mark workload RT acceptance blocked; do not invent a "120 microseconds" pass criterion.
- [ ] **Step 7: Add invalid/incomplete evidence tests, update acceptance docs, run coverage and commit.** Title: `image-mode: define guest compatibility and persistence gates`.

## Task 2: Package and stage immutable VM disks

**Files:**
- Create: `image-mode/schema/workload-v1.json`
- Create: `image-mode/workloads/Containerfile`, `pack-vm.py`
- Create: `image-mode/workloads/ssc600.example.yaml`, `windows.example.yaml`
- Create: `image-mode/runtime/vpac/bundles.py`
- Create: `tests/image_mode/test_bundles.py`, `test_pack_vm.py`
- Modify: `image-mode/workloads/README.md`

**Interfaces:**
- `Bundle` frozen dataclass: `digest: str`, `descriptor: dict`, `disk_path: Path`, `disk_format: str`.
- `stage_bundle(image: str, staging: Path, runner: Runner) -> Bundle`: immutable resolved content; never starts the image.
- `validate_workload(descriptor: dict) -> dict`: schema and semantic validation.
- Bundle contains descriptor `vm.yaml` and its declared disk path; `disk.format` is exactly `raw` or `qcow2`.

- [ ] **Step 1: Write format/failed-pull tests.**

```python
def test_failed_pull_does_not_touch_active_disk(tmp_path, runner):
    from vpac.bundles import stage_bundle
    from vpac.errors import ApplyError
    active = tmp_path / "active.raw"
    active.write_bytes(b"known-good")
    runner.fail_command("podman", "pull", stderr="registry unreachable")
    with pytest.raises(ApplyError, match="pull"):
        stage_bundle("registry.invalid/vm:test", tmp_path / "stage", runner)
    assert active.read_bytes() == b"known-good"
```

Extend host Runner fixture with `fail_command(*prefix, stderr)` matching argv prefixes. Test qcow2 named `.img`, raw named `.qcow2`, and descriptor versus `qemu-img info --output=json` disagreement.

- [ ] **Step 2: Run tests before bundle implementation.**
- [ ] **Step 3: Build a minimal OCI container image with disk + descriptor using `FROM scratch`.** The packaging tool validates supplied files, uses the declared format, records vendor checksum verification, and invokes Podman with argv arrays. It must not include live guest data, credentials or licensing tokens.

```python
info = json.loads(runner.run(
    ["qemu-img", "info", "--output=json", str(disk)], timeout=60
).stdout)
if info["format"] != descriptor["disk"]["format"]:
    raise ConfigError("disk format disagrees with workload descriptor")
```

Check return code before parsing stdout. Reject preexisting backing chains, encrypted/unsupported disks and path traversal. Synthetic disk generation uses qemu-img locally, not downloaded vendor media.

- [ ] **Step 4: Stage by resolved digest with registry auth/TLS inherited from host delivery.** Use `podman pull`/inspect, `podman create` and controlled `podman cp` (no container start). Reject symlink escapes and size/resource overrun, validate descriptor and qemu image, then atomically promote the complete staged directory. Always remove only the created temporary container on success/failure; never prune unrelated containers/images.
- [ ] **Step 5: Add tests for digest cache hit, changed tag resolution, corrupt JSON, truncated disk, credential redaction, failed copy/validation, symlinks, path traversal and interrupted staging cleanup.**
- [ ] **Step 6: Document package/push/verify commands and redistribution limitations together; run tests/coverage and commit.** Title: `image-mode: package and verify OCI virtual machine disks`.

## Task 3: Render compatible guest domains with resource isolation

**Files:**
- Modify: `image-mode/runtime/vpac/guest.py`
- Modify only as needed: `shared/templates/vm/domain.xml.j2`
- Modify: `roles/vm_templates/README.md` if shared inputs are extended
- Create: `tests/image_mode/test_guest.py`, `fixtures/guests/`
- Modify: `image-mode/workloads/README.md`, `image-mode/runtime/README.md`

**Interfaces:**
- `guest_context(entry: dict, bundle: Bundle, hardware: dict, disk_paths: dict[str, Path]) -> dict`: maps into `_vm`/`_profile` for the shared source.
- `render_guest(context: dict, templates: Path) -> bytes`.
- `validate_resources(entries: list[dict], bundles: dict[str, Bundle], hardware: dict) -> None`: validates total hugepages, distinct RT CPU assignments and unique MACs across enabled guests.
- `validate_persistence_contract()` from Task 1 remains the gate.

- [ ] **Step 1: Write structural guest XML tests.**

```python
def test_ssc600_nics_are_virtio_with_fixed_identity(ssc_context, templates):
    from vpac.guest import render_guest
    root = ElementTree.fromstring(render_guest(ssc_context, templates))
    nics = root.findall("./devices/interface")
    assert len(nics) == 3
    assert [nic.find("model").get("type") for nic in nics] == ["virtio"] * 3
    assert [nic.find("address").get("slot") for nic in nics] == ["0x02", "0x03", "0x07"]
    assert nics[1].get("type") == "direct"
    assert nics[1].find("source").get("mode") == "bridge"
    assert nics[1].get("trustGuestRxFilters") == "yes"
    assert len({nic.find("mac").get("address") for nic in nics}) == 3
```

Define `ssc_context` from a synthetic bundle descriptor + hardware fixture; import `ElementTree`. It uses no ABB disk or XML copy.

- [ ] **Step 2: Run this test to expose absent renderer and unsupported per-NIC MAC/PCI inputs.**
- [ ] **Step 3: Implement descriptor-to-shared-template mapping.** SSC600: 8 GiB, four dedicated single-thread vCPUs/FIFO 50, validated I/O pool, no balloon/reset watchdog, host CPU passthrough, q35, vendor-compatible NIC slots/roles and PTP filesystem tag. Do not inherit cluster-specific lease/storage/migration options. Windows: housekeeping CPU pinning, no FIFO, supported virtio drivers, per-instance NVRAM/TPM paths.
- [ ] **Step 4: Extend shared XML only with optional per-NIC MAC/address and memory source inputs.** Existing package entries without the new fields must render exactly the frozen Task 3 host outputs. Do not remove Ansible filters or change default devices to ease plain Jinja usage. New code must XML-escape values.
- [ ] **Step 5: Validate libvirt schema and resource budget.** Use `virt-xml-validate` on generated XML on Linux; add tests for insufficient RAM/hugepages, CPUs overlapping Windows/housekeeping, offline siblings, missing guest config support and injection requesting an unsupported vendor mechanism. Do not imply a vendor appliance supports cloud-init just because an ISO can be attached.
- [ ] **Step 6: Resolve PTP status integration from S3.** Use verified vendor scripts delivered with the workload or a validated compatible writer; SELinux labeling/read-only policy must allow the actual guest contract. Never replace the vendor output with unrelated JSON.
- [ ] **Step 7: Run guest tests, frozen package parity, original Ansible regression, coverage; update docs and commit.** Title: `image-mode: render isolated guest domains with stable virtio identities`.

## Task 4: Implement recoverable disk and VM update lifecycle

**Files:**
- Create: `image-mode/runtime/vpac/disks.py`, `vm.py`
- Create: `tests/image_mode/test_disks.py`, `test_vm_lifecycle.py`
- Modify: `image-mode/workloads/README.md`, `image-mode/ACCEPTANCE.md`

**Interfaces:**
- `VMPaths` frozen dataclass: `root: Path`, `golden: Path`, `overlay: Path`, `data: Path | None`, `nvram: Path | None`, `tpm: Path | None`, `metadata: Path`.
- `prepare_disks(entry: dict, bundle: Bundle, current: dict | None, root: Path, runner: Runner) -> VMPaths`: prepares a **candidate** generation; never deletes current state.
- `reconcile_vm(entry: dict, bundle: Bundle, hardware: dict, root: Path, runner: Runner) -> dict`: explicit phase/status with desired/running digest.
- On-disk metadata tracks active/candidate/previous digest and paths, format, identity paths, last phase and configuration fingerprint. Atomic writes after each irreversible boundary support crash recovery.

- [ ] **Step 1: Write update-failure and persistent-identity tests.**

```python
def test_candidate_start_failure_restores_previous_vm(vm_fixture, runner):
    from vpac.vm import reconcile_vm
    previous = vm_fixture.active_digest
    runner.fail_command("virsh", "start", "ssc600", stderr="candidate boot failed",
                        once=True)
    status = reconcile_vm(**vm_fixture.arguments)
    assert status["phase"] == "failed"
    assert status["runningDigest"] == previous
    assert vm_fixture.previous_overlay.exists()
    assert vm_fixture.data_disk.read_bytes() == b"persistent-settings"
```

Extend the fake with `once=True` to fail candidate start but permit restoration. Define `vm_fixture` in this task with candidate/current synthetic files, metadata and arguments; test no real libvirt side effects.

- [ ] **Step 2: Run the red test.**
- [ ] **Step 3: Implement disk creation using explicit format.**

```python
argv = ["qemu-img", "create", "-f", "qcow2", "-F", bundle.disk_format,
        "-b", str(bundle.disk_path), str(candidate_overlay)]
result = runner.run(argv, timeout=120)
if result.returncode:
    raise ApplyError("overlay", result.stderr)
```

Data disks are created once; size/format is checked thereafter. Reject requested shrink/replacement. Ephemeral recreates the system overlay on **managed cold starts**, not every `virsh` guest reboot; document this exact boundary. Reset-on-upgrade preserves overlay until digest changes and is refused for SSC600 without validated persistence/restore evidence.

- [ ] **Step 4: Implement candidate state machine:** resolve/pull/validate => resource/domain validation => prepare candidate disks/config => save previous definition/metadata => graceful shutdown with bounded wait => define/start candidate => verify libvirt running state and S3-approved readiness => atomically commit active generation. Running alone is not proof the relay application is healthy; expose that distinction.
- [ ] **Step 5: Handle failure/crash at every boundary.** Failed pull/render/prepare leaves the old VM running. Failed candidate start restores the prior definition/overlay, preserves identity/data, reports update failure even if recovery succeeds, and records recovery failures separately. Keep previous generation until explicit verified retirement; no automatic broad directory deletion.
- [ ] **Step 6: Preserve UEFI NVRAM/TPM across overlay recreation, disable/enable and retries.** Undefine must retain them using verified libvirt flags; do not use undefine flags that remove storage or TPM state. Persistent disks cannot be treated as automatically rolled back; candidate applications may change data schema, so S3 must define any required backup/recovery.
- [ ] **Step 7: Add tests:** same digest/no restart, changed config/scheduled restart, cache hit with drift repair, disabled entry/shutdown with disks retained, restart after host crash in each phase, failed graceful stop/no forced kill by default, recovery failure, old generation retention and malicious file paths.
- [ ] **Step 8: Document each disk mode/recovery procedure in this commit, run coverage and commit.** Title: `image-mode: reconcile virtual machine updates with preserved identity`.

## Task 5: Wire VM services, rollout coordination and truthful status

**Files:**
- Create: `image-mode/runtime/vpac/vm_cli.py`, `workloads.py`
- Create: `image-mode/files/usr/libexec/vpac/vpac-vm`
- Create: `image-mode/files/usr/lib/systemd/system/vpac-vm@.service`
- Modify: `image-mode/runtime/vpac/apply.py`, `health.py`, `cli.py`
- Modify: `image-mode/Containerfile`, `image-mode/edge-manager/fleet.yaml`
- Create: `tests/image_mode/test_vm_units.py`, `test_workloads.py`
- Modify: `image-mode/runtime/README.md`, `image-mode/edge-manager/README.md`, `image-mode/ARCHITECTURE.md`

**Interfaces:**
- `reconcile_workloads(desired: list[dict], observed: dict, runner: Runner) -> dict`: reconciles membership through managed systemd units.
- CLI: `vpac-vm NAME reconcile|stop|status`; validates name against node schema and takes an exclusive per-VM lock.
- Workload status exposes `desiredDigest`, `runningDigest`, `phase`, `guestReady`, `error`; host status is aggregated under one status writer/lock so two VM workers cannot overwrite each other.
- Health requires enabled RT domains actually running and healthy host configuration; inability to verify guest readiness is an explicit unknown, not `true`.

- [ ] **Step 1: Write orchestration tests.**

```python
def test_removed_vm_stops_but_keeps_disks(runner):
    from vpac.workloads import reconcile_workloads
    result = reconcile_workloads([], {"ssc600": {"managed": True, "running": True}}, runner)
    assert ("systemctl", "stop", "vpac-vm@ssc600.service") in runner.calls
    assert not any(call[0] in {"rm", "qemu-img"} for call in runner.calls)
    assert result["ssc600"]["phase"] == "disabled"
```

Also test disabled entries, invalid unit names, unrelated libvirt guests, config changed while a pull is active and two writers updating different VM statuses.

- [ ] **Step 2: Run tests before orchestration implementation.**
- [ ] **Step 3: Add host units with explicit readiness ordering.** Require successful host configure and RT prepare, then their verified desired-hash readiness; `After=` alone is insufficient. Libvirt autostart must not bypass readiness after reboot. On stop, use graceful bounded shutdown; report failure rather than silently force-destroying a protection VM.

Use `Type=oneshot`/`RemainAfterExit` only with explicit live-domain observation in health/status; systemd "active" alone does not mean the guest is alive. External Cockpit changes are observed and reconciled under the documented ownership/maintenance model.

- [ ] **Step 4: Remove the enabled-VM guard from the host once units are installed, wire desired workload reconciliation and PTP status dependency.** Domain-affecting network changes must stop/schedule dependent guests through the VM lifecycle before host network replacement. No restart of unrelated guests. Removing the last entry stops only vPAC-owned guests and retains their data.
- [ ] **Step 5: Wire verified RHEM configuration rollout controls.** Require a staged maintenance workflow before updating desired VM digest. If RHEM 1.3 offers only OS rollout controls, use the operator-controlled config procedure documented in host Task 8; do not invent an API or claim native VM application support.
- [ ] **Step 6: Extend health while separating failures.** OS boot regression can trigger OS rollback through the verified integration; independent VM/config update failure remains a VM/config failure with its own recovery. Persistent broken desired values must not produce an endless bootc rollback loop; test the verified bootstrap/recovery policy.
- [ ] **Step 7: Update operator docs/examples in this commit; run tests/coverage and commit.** Title: `image-mode: manage declarative VM services and workload status`.

## Task 6: Validate combined host/workload lifecycle on lab hardware

**Files:**
- Create: `tests/image_mode/integration/test_vm_boot.py`, `test_vm_update.py`
- Modify: `image-mode/ACCEPTANCE.md`, `image-mode/workloads/COMPATIBILITY.md`
- Modify: `README.md`, `image-mode/README.md`, `tests/README.md`, `.github/copilot-instructions.md`

**Interfaces:**
- Integration tests are opt-in and require an explicitly selected disposable lab device and operator-supplied media.
- Acceptance results include versions/digests, profile, numerical contract, measured outcomes, settings/identity persistence and exact command logs without credentials.

- [ ] **Step 1: Write integration tests for synthetic golden A/B and non-destructive recovery before testing proprietary guests.**

```python
@pytest.mark.integration
def test_failed_vm_update_keeps_identity_and_previous_guest(lab_vm):
    before = lab_vm.snapshot_identity()
    old_digest = lab_vm.running_digest()
    lab_vm.deliver_unbootable_candidate()
    lab_vm.wait_for_update_failure()
    assert lab_vm.running_digest() == old_digest
    assert lab_vm.snapshot_identity() == before
```

`lab_vm` fixture is implemented here with operator-selected SSH transport and verified commands, no default production host. A selection/authorization check runs before any mutation; pytest marker alone is not safety.

- [ ] **Step 2: Run synthetic tests on a disposable qcow2 host:** raw/qcow2 backing, candidate boot, pull/start failure, reboot recovery, disabled/re-enabled guest, removed desired entry, fixed MAC/UUID, NVRAM/TPM/data persistence.
- [ ] **Step 3: Execute vendor lab acceptance only after S3 gates pass.** Configure SSC600 with vendor tools; separately test the supported injected-config mechanism where available. Verify golden replacement/settings recovery/licensing behavior rather than inferring persistence from host disk files.
- [ ] **Step 4: Measure RT behavior on MK2:** cyclictest plus ABB guest and GOOSE/SV load, PRP traffic, dedicated PTP, background Windows work, image/disk transfer and monitoring load. Apply Task 1's numerical acceptance contract and record actual maximums/loss/jitter. PTP disabled is a supported configuration state, not equivalent protection timing quality.
- [ ] **Step 5: Exercise OS update independent of VM version, VM update independent of OS, and deliberately bad OS rollback with unchanged persistent VM state.** Confirm host RT policy, virtio NIC role/order, virtiofs status contract and SELinux remain functional; report any data-schema limit on downgrade.
- [ ] **Step 6: Run full image-mode tests/coverage, package parity and original Ansible regression, image lint and relevant boot integration.** Record unavailable mandatory tests as blocked acceptance, not an overall pass.
- [ ] **Step 7: Update all affected docs and Copilot commands/architecture with the shipped behavior in this same commit.** Keep operator quickstart concise; link deep recovery/compatibility procedures. Commit title: `image-mode: validate protection VM delivery and lifecycle`.

## Coverage and review checklist

| Behavior | Required evidence |
|---|---|
| Invalid inputs, unsupported formats, corrupt bundles | No active-state mutation, explicit failure tests |
| Overlay reset/create/update boundaries | Tests for every disk mode and crash phase |
| Candidate failure and recovery failure | Prior guest preservation, truthful failed desired update |
| MAC/UUID, UEFI, TPM, data disk identity | Stable across stop/start/update/reboot/OS rollback |
| Domain XML and shared sources | Structured assertions, libvirt validation, package-mode frozen parity |
| Unit/config coordination | Readiness, removal/disable, network dependencies and writer locking |
| PTP optional mode/status | Real disabled/unsynchronized output, no stale sync claims |
| Vendor timing | Numerical S3 acceptance contract plus actual MK2 measurements |
| Docs and examples | Same task commits, schema validation, recovery commands exercised |

Plan scope is complete only when both plans' acceptance gates are satisfied. Hardware-free tests establish implementation behavior, not vendor approval, RHEL certification or physical timing. Execution choice is already **subagent-driven development**; do not start it until the plans are accepted.
