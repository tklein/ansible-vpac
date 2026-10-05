# Bootc Host and Edge Manager Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a RHEL 10 RT bootc host that receives per-device values from Red Hat Edge Manager 1.3 standalone and configures itself without post-deployment Ansible.

**Architecture:** Evolve `image-mode/` into a hardware-profiled image with a small Python renderer/reconciler. Keep templates shared with package mode, protect management connectivity with an explicit network transaction, and publish truthful local health. VM delivery has its own dependent plan; this plan first delivers a usable, validated host with an empty VM list.

**Tech Stack:** RHEL 10 bootc, Podman, bootc-image-builder, NetworkManager/nmstate, systemd, tuned, linuxptp/chrony, libvirt modular daemons, RHEM 1.3/flightctl-agent, Python 3, Jinja2, PyYAML, jsonschema, pytest/pytest-cov.

**Spec:** `docs/superpowers/specs/2026-10-01-bootc-edge-manager-design.md`, committed at `c0bd480`.

## Global Constraints

- **Edge manager:** RHEM 1.3, standalone (RHEL-hosted).
- **Base OS:** `rhel10/rhel-bootc` (RHEL 10.x); choose an available supported minor/digest only after Task 1 evidence.
- **Hardware:** One model: Welotec Rugged Substation Automation Computer MK2.
- **PTP:** Dedicated onboard NIC; enabled by default, disable-able per device.
- **Coexistence:** Package mode stays first-class side by side.
- **Statelessness:** Immutable golden disk + small persistent data volume.
- No Ansible after image-mode deployment; controller Ansible is permitted for package-mode regression tests.
- No Ceph, Pacemaker, corosync, STONITH, or distributed-storage packages in the single-node image.
- No per-unit MACs, site identity, enrollment secrets, registry credentials, or vendor disks in the reusable host image.
- Exactly one installed kernel, and it must be the RT kernel. Kernel arguments are owned by `kargs.d`, not a second tuned bootloader writer.
- Derive CPU isolation, hugepages, tuned settings, IRQ policy and VM allocation from the same validated hardware profile.
- Guest virtio, host bridge/macvtap and physical `igc`/`igb` drivers are distinct concepts. Keep dedicated PTP separate from macvtap.
- Reference: Welotec revision `6cb52665b71cd656f44ee686a18bbe5333d0c604`; its active Debian path is evidence, not proof of RHEL 10 support.
- Every task includes its tests and operator-facing documentation in the same commit. A task is not complete with docs or covering tests deferred.
- Existing untracked `docs/superpowers/specs/2026-10-01-bootc-edge-manager-design copy.md` belongs to the user; do not read, stage, edit, or delete it.

---

## Execution, dependencies and scope

This is an implementation **plan**, not an instruction to start implementation while writing/reviewing it.

Use an isolated git worktree at execution time. Dispatch one fresh implementer per task, then a task-scoped review with separate **spec compliance** and **code quality** verdicts. Include the task brief, binding constraints, predecessor interfaces and exact test commands; implementers do not launch their own agents. Run a whole-branch review after the final task. Do not run concurrent writers, even for tasks with independent prerequisites.

| Task | Prerequisites | Independently reviewable result |
|---|---|---|
| 1 | None | Evidence gates and reproducible hardware/platform probes |
| 2 | None; real hardware fixture needs Task 1 | Validated node/profile/status contracts and test harness |
| 3 | 2 | Shared templates with unchanged package-mode renders |
| 4 | 2, 3 | Deterministic offline render and drift plan |
| 5 | 1, 4 | Transactional host reconciliation and truthful status |
| 6 | 1, 2 | Validated hardware RT policy and startup readiness |
| 7 | 1, 5, 6 | Buildable, linted RHEL 10 image with baked runtime |
| 8 | 1, 5, 7 | RHEM delivery, enrollment and install procedures |
| 9 | 5, 6, 7, 8 | Boot health, upgrade/rollback and acceptance evidence |

Tasks 2-4 can proceed with explicitly synthetic profiles when hardware is unavailable. Tasks 5-9 cannot claim lab acceptance until their real evidence gates are met. Missing subscriptions, RHEM access, physical hardware or vendor contracts are blockers, not permission to guess. Write the exact missing evidence in the execution ledger. Never silently substitute RHEL 9, invent RHEM API fields, or mark skipped hardware checks as passing.

The companion `2026-10-05-bootc-vm-delivery-implementation.md` covers VM packaging, persistent identities, updates and the RT-VM extension to health checks. Containers, HA and storage remain future sub-projects. `cluster:` is rejected with a clear unsupported-mode error in schema version 1, not accepted and ignored.

## File responsibilities

| Path | Responsibility |
|---|---|
| `image-mode/runtime/vpac/` | Small Python package: contracts, adapters, rendering, plans, command runner, apply and status |
| `image-mode/schema/` | Versioned node, hardware, status and (later) workload JSON schemas |
| `image-mode/profiles/` | Validated hardware class policy; no per-unit identity |
| `shared/templates/` | Single template sources, consumed by package roles and baked runtime |
| `image-mode/files/usr/libexec/vpac/` | Thin installed entry points; substantive Python stays in the package |
| `image-mode/files/usr/lib/systemd/system/` | Boot/path units, readiness and service ordering |
| `image-mode/files/usr/lib/greenboot/check/required.d/` | Thin health entry point, only if Task 1 validates this supported integration |
| `image-mode/edge-manager/` | Version-verified fleet/config examples and operator procedures |
| `image-mode/tools/` | Probes, profile generation, packaging/install helpers |
| `tests/image_mode/` | Unit, failure-path, render parity and integration tests |
| `tests/image_mode/fixtures/` | Synthetic public inputs and frozen pre-refactor expected outputs |

Keep each production module focused on one responsibility. Do not make one `vpac-configure` script own schema validation, rendering, command execution and VM disk management.

## Test and documentation policy

The repo currently has Ansible regression tests, not a measured Python coverage baseline. Establish a new baseline, then retain it. Do not claim pre-existing coverage percentages.

Task 2 creates:

```text
# requirements-test.txt: controller/test workstation only
pytest>=8,<9
pytest-cov>=5,<8
PyYAML>=6,<7
Jinja2>=3.1,<4
jsonschema>=4.22,<5
```

Use Python 3.11+ for the host test environment. Install Ansible from the existing controller setup for parity tests; keep these packages out of `requirements.txt`, which currently serves the Ansible controller's `passlib` dependency. Resolve runtime RPM names in Task 1 rather than deploying pip onto the node.

```bash
python -m pip install -r requirements-test.txt
python -m pytest tests/image_mode/test_contracts.py::test_rejects_duplicate_keys -q
python -m pytest tests/image_mode -m "not integration" \
  --cov=image-mode/runtime/vpac --cov-branch \
  --cov-report=term-missing --cov-report=xml --cov-fail-under=95
ansible-playbook tests/regression/test_linkdown_scope.yml
```

Gate new runtime Python at **at least 95% aggregate coverage with branch measurement enabled**, and require explicit tests for **every** destructive, rollback, invalid-input and partial-failure path regardless of the aggregate percentage. Use `coverage report --show-missing` for per-file gaps; no broad `pragma: no cover`, exclusions or percentage-lowering to satisfy the gate. Probe/build scripts and systemd units also need behavioral tests; Python coverage cannot establish that they work.

After each production task: targeted tests first, then the full non-integration image-mode suite and its coverage gate. Task 3 and any later package-role modification also run the Ansible regression and parity suite. Integration tests must have an explicit opt-in marker and clear environment preconditions; release acceptance fails when required hardware tests are unavailable.

Docs are deliverables within each task: examples validate against the schema, generated configuration is explained beside its owner, and every public command includes single-test usage. Root README remains protection-engineer-oriented; detailed implementation contracts stay under `image-mode/`.

## Task 1: Establish hardware and platform evidence gates

**Files:**
- Create: `image-mode/tools/probe-hardware.sh`
- Create: `image-mode/tools/check-platform.sh`
- Create: `image-mode/profiles/README.md`
- Create: `image-mode/edge-manager/README.md`
- Create: `tests/image_mode/test_probes.py`
- Create: `tests/image_mode/conftest.py` (command-shim fixture, extended in Task 2)
- Create: `requirements-test.txt` (initial test dependencies, finalized in Task 2)
- Modify: `image-mode/README.md` (status/evidence section)

**Interfaces:**
- Consumes: authorized RHEL build host, MK2 read-only access, RHEM 1.3 lab endpoint; not credentials in git.
- Produces: `probe-hardware.sh` emits public topology/capability JSON to stdout, exits nonzero on incomplete collection.
- Produces: dated evidence in the docs, sanitized identifiers, immutable base-image digest and RPM/source versions. Probe output containing per-unit MACs stays outside git.

- [ ] **Step 1: Write probe behavior tests before scripts.**

```python
def test_hardware_probe_propagates_lscpu_failure(tmp_path, fake_path):
    fake_path.command("lscpu", exit_code=17, stderr="probe unavailable")
    result = fake_path.run("image-mode/tools/probe-hardware.sh")
    assert result.returncode != 0
    assert "probe unavailable" in result.stderr
    assert not result.stdout.strip()
```

Implement the `fake_path` fixture in `conftest.py`: create executable shell shims in `tmp_path`, prepend that directory to the child process PATH, run with `subprocess.run(..., capture_output=True, text=True, check=False)`. Its public API is `.command(name, exit_code=0, stdout="", stderr="")`, `.run(script, *args, env=None)` and `.calls(name)`. Command scripts record invocations into a temporary JSON-lines file. Restrict these script tests to Linux; pure Python runtime tests remain hardware-free. Also test success JSON, no NIC timestamp capability, and unavailable CAT tooling as explicit reported capabilities, not invented support.

- [ ] **Step 2: Run `python -m pytest tests/image_mode/test_probes.py -q`; confirm failure because the script is missing.** Add the test dependencies from the policy block if the chosen runner reports missing packages; this task owns the initial manifest until Task 2 finalizes it.
- [ ] **Step 3: Implement read-only collection.** Use strict shell error handling, `lscpu -J`, `lscpu -e`, sysfs CPU sibling/online data, `lspci -nnk`, `ethtool -i`/`-T`, and sysfs port/device paths. Do not offline CPUs, write MSRs, rename NICs, alter networking or run RT-load probes during discovery.

```bash
set -euo pipefail
topology="$(lscpu -J)"
printf '%s\n' "$topology" | python3 -m json.tool >/dev/null
# Collect all required outputs before emitting the final JSON document.
```

- [ ] **Step 4: On the authorized builder, establish S2.** Query actual RHEL 10 BaseOS/AppStream/RT/NFV content, kernel package layout, tuned, linuxptp, nmstate, runtime Python RPMs, greenboot and flightctl-agent sources. Build a minimal disposable RT bootc image, run `bootc container lint`, then boot it and confirm `/sys/kernel/realtime` plus the running kernel. Record commands, digest and output; a name containing `rt` alone is insufficient. Do not edit the production Containerfile yet.
- [ ] **Step 5: Establish S4 on RHEM 1.3.** Verify the exact enrollment artifact, supported installer file-injection method, inline configuration and fleet templating syntax, config rollout policy, registry CA/auth delivery, hook ordering, and device status reporting API. Checkpoint/check-confirm capabilities for the installed nmstate version and the supported OS-health/rollback integration must be recorded. Use versioned authoritative docs and real lab round trips, not upstream-latest examples.
- [ ] **Step 6: Record findings with sources and limitations in the three docs.** Include the hardware model/CPU variant and dedicated PTP port mapping. If access is absent, record the blocked gate rather than fabricating a usable MK2 profile or claimed build.
- [ ] **Step 7: Run probe tests and commit only task files.** Commit title: `image-mode: add hardware and platform evidence probes`, with the standard Copilot co-author trailer.

## Task 2: Define validated contracts and test scaffolding

**Files:**
- Create: `image-mode/schema/node-v1.json`, `hardware-v1.json`, `status-v1.json`
- Create: `image-mode/runtime/vpac/__init__.py`, `errors.py`, `contracts.py`
- Modify: `tests/image_mode/conftest.py`
- Create: `tests/image_mode/test_contracts.py`
- Create: `tests/image_mode/fixtures/node-minimal.yaml`, `hardware-synthetic.json`
- Create/finalize: `requirements-test.txt`, `pytest.ini`
- Modify: `tests/README.md`, `image-mode/runtime/README.md`

**Interfaces:**
- `ConfigError(ValueError)` carries a field path and safe diagnostic; no secrets.
- `load_node(path: Path) -> dict[str, object]`: rejects duplicate keys, unsupported schema versions and unknown properties.
- `load_hardware(path: Path) -> dict[str, object]`: validates baked class policy.
- `validate_pair(node: dict, hardware: dict) -> None`: validates semantic relationships before mutation.
- `load_node()` accepts the fully specified `vms` contract in the companion plan, but this host phase rejects **enabled** VMs until VM runtime support is installed. Do not ignore requested workloads.

- [ ] **Step 1: Write failing tests for parsing and semantic validation.**

```python
def test_rejects_duplicate_keys(tmp_path):
    from vpac.contracts import load_node
    from vpac.errors import ConfigError
    path = tmp_path / "node.yaml"
    path.write_text("schemaVersion: 1\nhostname: a\nhostname: b\n")
    with pytest.raises(ConfigError, match="duplicate"):
        load_node(path)

def test_rejects_ptp_port_reuse(node, hardware):
    from vpac.contracts import validate_pair
    from vpac.errors import ConfigError
    hardware["ports"]["ptp"]["path"] = hardware["ports"]["process"]["path"]
    with pytest.raises(ConfigError, match="PTP"):
        validate_pair(node, hardware)
```

Define imports (`pytest`, `Path`) in each test module. `conftest.py` prepends the resolved `image-mode/runtime` directory to `sys.path` and exposes independent deep copies via fixtures `node` and `hardware`.

The synthetic fixture defines 8 physical/16 logical CPUs, housekeeping `[0,1,2]`, I/O `[3]`, vCPUs `[4,5,6,7]`, offline siblings `[8,9,10,11,12,13,14,15]`, and **explicitly synthetic** PCI path selectors for `mgmt`, `station`, `process`, `service`, `ptp`. It must never be installed as a production profile.

- [ ] **Step 2: Run the two single tests; expect missing-module/import failures before implementation.**
- [ ] **Step 3: Implement safe YAML loading and JSON Schema validation.**

```python
class UniqueKeyLoader(yaml.SafeLoader):
    pass

def construct_mapping(loader, node, deep=False):
    result = {}
    for key_node, value_node in node.value:
        key = loader.construct_object(key_node, deep=deep)
        if not isinstance(key, str) or key in result:
            raise ConfigError("duplicate or non-string mapping key")
        result[key] = loader.construct_object(value_node, deep=deep)
    return result
```

Register the constructor on this loader only, not globally. Enable jsonschema format checking and reject non-finite numbers, excessive file size (>1 MiB), ambiguous MACs, invalid CIDRs and unexpected YAML objects.

Freeze the following fields:

```yaml
schemaVersion: 1
hostname: lab-node
networks:
  mgmt: {address: 10.1.1.11/24, gateway: 10.1.1.1, dns: [10.1.1.2]}
  station: {address: 10.1.2.11/24, vlan: 20}
ptp: {enabled: true, domain: 0, profile: iec-61850-9-3}
time: {ntpServers: [10.1.1.2]}
vms: []
```

`hostname`, both network objects, `time` and `vms` are required. Missing `ptp` means enabled, domain 0, IEC 61850-9-3; an explicitly selected C37.238 profile is allowed only after Task 1 validates its parameter mapping. Use standard schema defaults only via explicit normalization; jsonschema validation does not apply defaults itself.

Hardware fields: `schemaVersion`, `id`, `cpu` (three allocation pools, offline siblings, hugepage size/count, FIFO priority, supported NUMA node), `ports` (role => class-level path selector/driver/stable name), and `capabilities` (validated frequency/CAT policy). No addresses, hostnames or permanent-MAC constants.

- [ ] **Step 4: Add tests for invalid versions, unknown keys/cluster config, MAC uniqueness, invalid VM names/path traversal, IPv4/CIDR/gateway mismatch, conflicting CPU pools/siblings, insufficient hugepages, reserved physical ports, and absence of NTP servers when PTP is disabled.** VM CPU/memory constraints are checked when workload descriptors are available in the companion plan.
- [ ] **Step 5: Define status schema:** `schemaVersion`, `desiredHash`, `appliedHash`, `phase`, `steps`, `workloads`, `error`, `updatedAt`. `desiredHash` may differ from `appliedHash` after failure; `phase="failed"` must never appear as healthy. `workloads` begins as `{}`.
- [ ] **Step 6: Validate the committed examples in tests and document every field/default and unsupported feature.** Explain host IP versus guest IP and omit factory-default secondary addresses unless explicitly requested.
- [ ] **Step 7: Run contract tests, coverage gate and commit.** Title: `image-mode: define node and hardware configuration contracts`.

## Task 3: Share template sources without package-mode drift

**Files:**
- Create: `shared/templates/networking/nmstate.yml.j2`
- Create: `shared/templates/networking/singlenode.yml.j2`
- Create: `shared/templates/virtualization/libvirt-network.xml.j2`
- Create: `shared/templates/time/timemaster.conf.j2`, `chrony-ptp-followers.conf.j2`
- Create: `shared/templates/vm/domain.xml.j2`
- Modify source-selection tasks in: `roles/networking/tasks/apply.yml`, `roles/virtualization/tasks/libvirt_networks.yml`, `roles/ptp_timesync/tasks/timemaster.yml`, `roles/ptp_timesync/tasks/ntp_follower.yml`, `roles/vm_templates/tasks/render_one.yml`, `image-mode/runtime/networking/apply-networking.yml`
- Remove old template copies only after their consumers use shared sources.
- Create: `tests/image_mode/test_template_parity.py`, `fixtures/package-mode/` inputs and expected renders
- Modify: affected role READMEs, `tests/README.md`

**Interfaces:**
- Package consumers retain their existing inventory variable interface and profile YAML locations.
- Shared templates initially retain their existing inputs (`_vm`, `_profile`, `item`, `vpac_net`, networking/time variables). Task 4 supplies adapters and compatible filters outside templates.
- Source selection uses `{{ role_path }}/../../shared/templates/...` for roles, and a path resolved from the runtime playbook's `playbook_dir` for the legacy image-mode play. Never depend on caller cwd or `site.yml` being the entry point.

- [ ] **Step 1: Before moving any template, capture real Ansible renders into frozen fixtures.** Cover untagged/tagged mgmt/station, shared/dedicated heartbeat, PTP/no PTP, file/raw/qcow2/RBD/lease disks, SSC600/VPR and both Windows profiles. Capture success output with controlled inventory inputs and record their commit provenance. Never regenerate expected output from the post-refactor source in the same test.
- [ ] **Step 2: Write and run a failing shared-source existence/parity test.**

```python
def test_shared_source_is_used(package_renderer, repo_root):
    expected = repo_root / "shared/templates/vm/domain.xml.j2"
    assert expected.is_file()
    assert package_renderer.template_path("vm_templates", "domain") == expected
    assert package_renderer.render_case("ssc600-file") == \
        package_renderer.expected_case("ssc600-file")
```

`package_renderer` is implemented in this task: a controller-local Ansible play harness using the actual template module, tracked fixture variables, temp output directories, `gather_facts: false`, `become: false`, and failure on nonzero Ansible exit. It inspects source-selection paths from the changed tasks. Do not replace real Ansible rendering with a mock.

- [ ] **Step 3: Move sources unchanged, wire all consumers and remove redundant copies.** Preserve attribution, comments, defaults and rendered bytes. Do not fix unrelated template issues, change watchdog defaults, replace disk-I/O defaults or add new VM device defaults here.
- [ ] **Step 4: Run parity against every frozen case plus `ansible-playbook tests/regression/test_linkdown_scope.yml`.** Also run an invocation from a nested playbook directory and a non-root cwd to exercise path resolution.
- [ ] **Step 5: Update affected role docs with shared ownership and exact test commands; commit.** Title: `templates: share configuration sources without changing package mode`.

## Task 4: Render deterministic configuration and compute host drift

**Files:**
- Create: `image-mode/runtime/vpac/adapters.py`, `render.py`, `plan.py`
- Create: `tests/image_mode/test_render.py`, `test_plan.py`, `fixtures/rendered/`
- Modify: `image-mode/runtime/README.md`, `tests/README.md`

**Interfaces:**
- `build_context(node: dict, hardware: dict) -> dict`: maps values into existing shared-template variables; no I/O.
- `render_host(node: dict, hardware: dict, templates: Path) -> dict[str, bytes]`: keyed by allowlisted installed absolute destination paths.
- `desired_hash(node: dict, hardware: dict, template_version: str) -> str`: SHA-256 of canonical normalized inputs plus image-owned version.
- `HostPlan` frozen dataclass with `desired_hash: str`, `writes: dict[str, bytes]`, `network_change: bool`, `time_mode: str`.
- `plan_host(rendered: dict[str, bytes], live: dict[str, bytes], hash_: str, time_mode: str) -> HostPlan`.

- [ ] **Step 1: Write failing deterministic/strict-render tests.**

```python
def test_station_vlan_and_dedicated_ptp(node, hardware, templates):
    from vpac.render import render_host
    rendered = render_host(node, hardware, templates)
    state = yaml.safe_load(rendered["/etc/vpac/generated/nmstate.yaml"])
    by_name = {item["name"]: item for item in state["interfaces"]}
    assert by_name["vpac-process"]["ipv4"]["enabled"] is False
    assert by_name["vpac-ptp"]["ipv4"]["enabled"] is False
    assert by_name["br-station"]["bridge"]["port"][0]["name"] == "vpac-station.20"
    assert rendered == render_host(node, hardware, templates)
```

`templates` fixture resolves `shared/templates`; stable names here are synthetic class names, not assumed Welotec numbering.

- [ ] **Step 2: Run test and confirm missing implementation.**
- [ ] **Step 3: Implement pure adapters and Jinja rendering with `StrictUndefined`.** Register only the compatible filters actually used (`bool`, `to_uuid`, `regex_replace`, `split`, `undef`); add tests against Ansible behavior for each. Preserve UUID namespace and numeric/boolean conversion semantics. Do not expose shell execution as a template helper.

```python
def desired_hash(node, hardware, template_version):
    payload = {"node": node, "hardware": hardware, "templates": template_version}
    encoded = json.dumps(payload, sort_keys=True, separators=(",", ":"),
                         allow_nan=False).encode("utf-8")
    return hashlib.sha256(encoded).hexdigest()
```

Create libvirt network descriptions for managed bridges; direct guest NICs remain direct macvtap, not a second Linux bridge. Generate PTP disabled state as no clock ownership, not reuse of the dedicated port.

- [ ] **Step 4: Add tests for undefined variables, XML/YAML escaping, invalid profile, no secret rendering, VLAN removal and disabled PTP.** Parse generated XML/YAML structurally, not just string snippets. Time rendering must have one clock owner: timemaster-managed children when PTP is enabled; system chronyd when disabled.
- [ ] **Step 5: Implement drift planning from file bytes and observed service/network state.** A matching stored hash is not sufficient for idempotency: edits to managed files must be repaired. Unchanged outputs produce no writes or restarts; render failure produces no plan.
- [ ] **Step 6: Document output ownership, dry-run usage and default mappings alongside the implementation; run tests/coverage and commit.** Title: `image-mode: add deterministic host configuration rendering`.

## Task 5: Apply host configuration with connectivity protection

**Files:**
- Create: `image-mode/runtime/vpac/runner.py`, `apply.py`, `status.py`, `cli.py`
- Create: `image-mode/files/usr/libexec/vpac/vpac-configure`
- Create: `image-mode/files/usr/lib/systemd/system/vpac-configure.service`, `vpac-configure.path`
- Create: `tests/image_mode/test_apply.py`, `test_status.py`, `test_cli.py`
- Modify: `image-mode/runtime/README.md`, `image-mode/edge-manager/README.md`

**Interfaces:**
- `CommandResult` frozen dataclass: `returncode: int`, `stdout: str`, `stderr: str`.
- `Runner.run(argv: Sequence[str], *, timeout: int, input_text: str | None = None) -> CommandResult`; subprocess argument arrays only.
- `ApplyError(RuntimeError)` carries the failed step and redacted stderr.
- `NetworkTransaction.begin(path: Path) -> str`, `.confirm(token: str) -> None`, `.rollback(token: str) -> None`, `.reachable() -> bool`: backend uses Task 1's verified nmstate checkpoint API and management endpoint.
- `apply_host(plan: HostPlan, root: Path, runner: Runner, network: NetworkTransaction) -> dict`: returns schema-valid status.
- `write_status(path: Path, status: dict) -> None`: atomic replacement; non-success remains observable.
- CLI: `vpac-configure validate|render|apply|status --node PATH --hardware PATH`; render is read-only except an explicitly requested output directory.

- [ ] **Step 1: Add command-recording fakes and write the connectivity failure test.**

```python
def test_unreachable_management_rolls_back(plan_with_network, tmp_path, runner, network):
    from vpac.apply import apply_host
    network.reachable_result = False
    status = apply_host(plan_with_network, tmp_path, runner, network)
    assert network.calls == ["begin", "reachable", "rollback"]
    assert status["phase"] == "failed"
    assert status["appliedHash"] is None
    assert not any(call[0] == "hostnamectl" for call in runner.calls)
```

`runner` fixture stores argv tuples and returns configured `CommandResult`s. `network` fixture implements the declared protocol, returns a token, records calls, and can fail each operation. `plan_with_network` renders Task 2 fixtures and constructs Task 4's HostPlan. All writes resolve under `tmp_path`; forbid writes to the real test host.

- [ ] **Step 2: Run failure test before implementation.**
- [ ] **Step 3: Implement staged render, lock, file ownership checks and transaction orchestration.** Use an exclusive process lock; read a complete input snapshot, validate all inputs/outputs before mutation, stage on the same filesystem, and commit files using `os.replace`.

```python
token = network.begin(staged_nmstate)
try:
    if not network.reachable():
        raise ApplyError("network", "management endpoint unreachable")
    network.confirm(token)
except ApplyError:
    network.rollback(token)
    raise
```

Use an HTTP/TLS management endpoint and certificate validation established in Task 1. A server's enrollment/authorization response may prove reachability even when unauthenticated; define allowed responses explicitly. Do not rely on `nmstatectl apply` returning zero or on a local route existing as proof of remote connectivity. Capture the original network state and confirm the real checkpoint backend supports explicit confirm/rollback. No broad catch or silent success when rollback fails.

- [ ] **Step 4: Apply hostname, managed files, firewall zones, time services and libvirt networks only after network confirmation.** Check every command exit. Compare libvirt inactive/active definitions; never redefine an unrelated network or overwrite unmanaged config. Changing a network used by running guests requires a coordinated maintenance action, not silent destroy/recreate.
- [ ] **Step 5: Implement PTP transitions.** Enabled: stop/mask system chronyd, then enable timemaster (which owns ptp4l/phc2sys/chronyd children). Disabled: stop/disable timemaster, then unmask/enable system chronyd. Do not start standalone ptp4l/phc2sys concurrently with timemaster. Preserve an explicit unsynchronized/disabled status for guests; never leave stale "synchronized" data.
- [ ] **Step 6: Cover invalid input => no commands, rollback failure, command timeout/nonzero, partial time/libvirt failures, stale hashes, file drift, concurrent invocations, service changes, repeated boot reconciliation and atomic RHEM file replacement.** After partial failure, `appliedHash` remains last fully successful hash; report failing step, and a retry must converge actual state. Only managed files are restorable; do not claim whole-host atomicity.
- [ ] **Step 7: Wire boot + path units with ordering and coalescing.** Watch the parent directory as well as path replacement; the entry point reconciles current content under its lock. Missing first-boot node config is "awaiting-config", never successful configuration. VM startup later requires a readiness result for the currently desired hash.
- [ ] **Step 8: Document recovery, maintenance changes and status semantics; run tests/coverage and commit.** Title: `image-mode: reconcile host state with explicit connectivity rollback`.

## Task 6: Generate and verify the baked RT hardware policy

**Files:**
- Create: `image-mode/profiles/mk2.json` (only from S1 evidence)
- Create: `image-mode/runtime/vpac/hardware.py`, `rt.py`
- Create: `image-mode/tools/render-hardware-profile.py`
- Create: `image-mode/files/usr/lib/systemd/system/vpac-rt-prepare.service`
- Create: `image-mode/files/usr/libexec/vpac/vpac-rt-prepare`
- Modify: `image-mode/kargs.d/10-vpac-rt.toml`, `image-mode/files/etc/tuned/realtime-virtual-host-variables.conf`, `image-mode/files/etc/sysconfig/cpupower`
- Create: `tests/image_mode/test_hardware.py`, `test_rt.py`
- Modify: `image-mode/profiles/README.md`, `image-mode/README.md`

**Interfaces:**
- `generate_profile_files(profile: dict) -> dict[str, bytes]`: derives kargs, tuned variables and udev/systemd `.link` naming.
- `verify_hardware(profile: dict, observed: dict) -> None`: rejects wrong CPU/port capabilities before RT startup.
- `prepare_rt(profile: dict, runner: Runner, sysfs: Path) -> dict`: applies validated runtime-only policy and reports readiness.
- Hardware readiness is not implied by `network-online.target`; VM units consume the successful `vpac-rt-prepare.service` and observed state.

- [ ] **Step 1: Write a failing profile consistency test.**

```python
def test_isolation_sources_agree(hardware):
    from vpac.hardware import generate_profile_files
    files = generate_profile_files(hardware)
    kargs = files["/usr/lib/bootc/kargs.d/10-vpac-rt.toml"].decode()
    tuned = files["/etc/tuned/realtime-virtual-host-variables.conf"].decode()
    assert '"isolcpus=3-7"' in kargs
    assert '"nohz_full=3-7"' in kargs
    assert '"rcu_nocbs=3-7"' in kargs
    assert "isolated_cores=3-7" in tuned
```

- [ ] **Step 2: Run this test; confirm missing generator.**
- [ ] **Step 3: Implement derivation and capability-gated runtime policy.** Generate one isolation set from I/O plus vCPU pools and validate that housekeeping is nonempty. Reserve the exact hugepage budget. Never hardcode synthetic CPU indices or unit MACs into production outputs. Boot readiness checks actual online sibling/isolation sets, not just intended config.

For frequency/C-state/CAT use the validated backend from S1/S2. Prefer supported resctrl/pqos interfaces where sufficient; raw MSR writes need documented, model-specific approval. Do not combine resctrl allocation and uncoordinated MSR writers.

- [ ] **Step 4: Implement persistent port naming by validated class PCI/physical path; test ambiguous/missing matching as failures.** Do not assume all MK2 CPU/NIC variants share one path layout. PRP firmware configuration remains manual and documented.
- [ ] **Step 5: Apply NIC IRQ and IRQ-thread policy after interfaces/tuned settle, and recheck after queue recreation.** Use sysfs MSI IRQ enumeration and recorded physical-device identity rather than grepping aliases only. CPU 3 in the reference is a validated role to reproduce, not a universal constant. No fixed one-minute delay; setup must report readiness before starting guests.
- [ ] **Step 6: Add tests for offline siblings, unsupported CAT/frequency controls, failed sysfs write, missing device, recreated IRQ list, policy applied before VM readiness and no swap.** Windows/host background processes must remain on housekeeping CPUs; the image must not enable swap.
- [ ] **Step 7: Record generated profile provenance and validation commands in docs.** Include read-only checks and explain which settings require an image rebuild/reboot. Run tests/coverage and commit. Title: `image-mode: bake validated MK2 real-time hardware policy`.

## Task 7: Assemble and validate the RHEL 10 image

**Files:**
- Modify: `image-mode/Containerfile`, `image-mode/build.sh`
- Create: `image-mode/tools/install-runtime.sh`
- Create: `tests/image_mode/test_build_contract.py`
- Modify: `image-mode/README.md`, `image-mode/ARCHITECTURE.md`, `README.md`

**Interfaces:**
- Build context is repository root so shared sources can be copied; `build.sh` resolves it independently of caller cwd.
- Image installs `vpac` package under a documented `/usr/lib/vpac-python` path, templates under `/usr/share/vpac/templates`, schemas under `/usr/share/vpac/schema`, and hardware profile under `/usr/share/vpac/profiles/mk2.json`. Entry points set their own import path.
- `build.sh connected|airgapped IMAGE_REF` retains the CLI but uses version-verified RHEL 10 sources and tools.

- [ ] **Step 1: Write command-shim build tests before modifying the build.**

```python
def test_airgapped_requires_mirror_without_build(fake_path):
    result = fake_path.run("image-mode/build.sh", "airgapped", "registry.invalid/node:test",
                           env={"MIRROR_URL": ""})
    assert result.returncode == 2
    assert fake_path.calls("podman") == []
```

Cover root build context, spaces in paths, connected/air-gapped args, nonzero build/lint/push propagation, and no push after lint failure. `fake_path` extends Task 1's fixture, retaining its command/result API.

- [ ] **Step 2: Run tests against the old script; demonstrate failures for the new context and mandatory lint contract.**
- [ ] **Step 3: Rebase onto the S2-verified RHEL 10 base/digest and package sources.** Install Python runtime RPMs, profiling/validation tools, flightctl-agent and supported health components from authorized sources. Keep no site secrets in layers. Copy the generated RT files and runtime, enable services, and mask conflicting jitter sources while retaining the existing package path unchanged.

```bash
podman build -f "${HERE}/Containerfile" -t "${IMAGE_REF}" "${REPO_ROOT}"
podman run --rm "${IMAGE_REF}" bootc container lint
podman push "${IMAGE_REF}"
```

Add kernel-count/RT assertions in the build and retain lint failure as fatal. Clean only build-generated cache/state paths; runtime `/var` directories are created by tmpfiles or services, not shipped with mutable data.

- [ ] **Step 4: Update emitted bootc-image-builder recipe to the S2-verified image/version and install config.** Do not assume the tool's registry path or package naming matches RHEL 9.
- [ ] **Step 5: Actually build and lint connected and mirror-only images.** Capture base/builder/output digests, RPM manifest and command output. Missing RT/agent packages or subscription access fails the gate; do not fallback or drop kernel-rt.
- [ ] **Step 6: Update docs in this same commit.** Replace "all site/hardware settings require post-boot Ansible" with the new values-file path, distinguish legacy lab helpers, document the root build context, rebuild procedure and current verification status. Run tests/coverage and commit. Title: `image-mode: build RHEL 10 host with baked configuration runtime`.

## Task 8: Deliver device values and enrollment through RHEM/installers

**Files:**
- Create: `image-mode/edge-manager/fleet.yaml`, `device-config.example.yaml`
- Create: `image-mode/tools/prepare-install.py`
- Modify: `image-mode/bib/config.toml`
- Create: `tests/image_mode/test_edge_manager_examples.py`, `test_install_config.py`
- Modify: `image-mode/edge-manager/README.md`, `image-mode/runtime/README.md`, `image-mode/README.md`

**Interfaces:**
- Enrollment and registry auth input paths are supplied by the operator and never committed.
- `prepare_install(node_path: Path, enrollment_path: Path, output_dir: Path) -> list[Path]` creates restricted-permission installer payloads using S4's verified mechanism, fails on missing inputs/unsupported delivery targets.
- Generated payloads contain `/etc/vpac/node.yaml`, actual enrollment config path verified by S4, CA trust and supported pull-secret config.
- Node config can be supplied by fleet template or installer; only one ongoing owner is documented. RHEM becomes authoritative after enrollment.

- [ ] **Step 1: Write failing tests that validate examples and prevent credential logging.**

```python
def test_install_payload_permissions(tmp_path, node_path, enrollment_path):
    from importlib import util
    spec = util.spec_from_file_location("prepare_install", "image-mode/tools/prepare-install.py")
    module = util.module_from_spec(spec)
    spec.loader.exec_module(module)
    paths = module.prepare_install(node_path, enrollment_path, tmp_path / "out")
    assert paths
    assert all((path.stat().st_mode & 0o077) == 0 for path in paths)
```

Use POSIX permission assertions on Linux. Fixtures create public synthetic enrollment input in temp files, not real credentials.

- [ ] **Step 2: Confirm failure before implementation.**
- [ ] **Step 3: Implement the artifact generator and fleet examples using only S4-verified schema.** Never treat device labels as secrets. Validate structured label values before writing YAML. Use inline/file config for lists where label templates cannot safely represent them. All generated `node.yaml` examples must pass Task 2 schemas.
- [ ] **Step 4: Test real RHEM delivery on a disposable device:** enrollment, first config, change PTP enabled => disabled => enabled, invalid config, atomic file replacement, registry CA/auth, config reconciliation after reboot. Capture device desired/rendered configuration and local status without credentials.
- [ ] **Step 5: Document ISO/USB, PXE/HTTP and factory workflows.** Provide exact tool invocation for each supported method established in S4, admin SSH-key input, disk selection and enrollment identity safety. Do not ship reusable device private keys in shared media or publish customer enrollment payloads.
- [ ] **Step 6: Document explicit config rollout/maintenance procedure.** A bootc OS rollout policy is not automatically a VM/config rollout policy. Verify the supported RHEM mechanism; if unavailable, require an operator-controlled staged rollout and report that limitation.
- [ ] **Step 7: Validate examples, tests/coverage and commit.** Title: `image-mode: add verified Edge Manager enrollment and device configuration`.

## Task 9: Establish health, OS update/rollback and host acceptance

**Files:**
- Create: `image-mode/runtime/vpac/health.py`
- Create: `image-mode/files/usr/libexec/vpac/vpac-health`
- Create: `image-mode/files/usr/lib/greenboot/check/required.d/50-vpac-health.sh` (only with S4-verified support)
- Create: `tests/image_mode/test_health.py`, `tests/image_mode/integration/test_boot_host.py`
- Create: `image-mode/ACCEPTANCE.md`
- Modify: `image-mode/Containerfile`, `image-mode/ARCHITECTURE.md`, `image-mode/README.md`, `tests/README.md`

**Interfaces:**
- `check_health(profile: dict, status: dict, observed: dict, *, enrollment_complete: bool) -> list[str]`: empty list means healthy; errors are structured, actionable strings.
- `vpac-health --required` exits zero only after all required readiness checks pass within a bounded documented startup window.
- Observed state includes RT kernel, CPU/hugepages, no swap, PTP mode/service ownership, management reachability and actual last reconciliation result.
- Before enrollment/config delivery: report `awaiting-config`; use a separately tested installation/bootstrap phase so a fresh device does not enter endless rollback before it can receive config.

- [ ] **Step 1: Write failure tests before the checker.**

```python
def test_stale_applied_hash_is_not_healthy(hardware, healthy_status, observed):
    from vpac.health import check_health
    healthy_status["desiredHash"] = "new"
    healthy_status["appliedHash"] = "old"
    errors = check_health(hardware, healthy_status, observed, enrollment_complete=True)
    assert any("configuration" in error for error in errors)
```

This task defines `healthy_status` and `observed` fixtures consistent with Task 2's schema and synthetic hardware. Cover wrong kernel, hugepage shortage, RT prepare failure, disabled PTP working via NTP, and awaiting-config behavior.

- [ ] **Step 2: Run failing test, then implement bounded, read-only observation and health checks.**
- [ ] **Step 3: Integrate the S4-verified health/rollback mechanism.** Do not assert greenboot works with this RHEL 10 bootc combination without a real failed-update/reboot experiment. Boot order must allow configuration convergence before acceptance. No network dependency makes a completed installation erase device identity.
- [ ] **Step 4: Write opt-in qcow2 boot integration tests.** Verify runtime imports, no post-boot package installation, file delivery, no Ansible on target, services, generated libvirt networks, repeat reconciliation and schema-valid status.
- [ ] **Step 5: Perform OS image A => B update and deliberately bad B => A rollback on the lab device.** Show RT kernel/profile restoration and explicitly show what persists: `/etc` edits, `/var` VM data and desired node values may survive OS rollback. Restore/configure compatible desired values when needed; never claim OS rollback undoes data/config updates.
- [ ] **Step 6: Run bare-metal MK2 acceptance:** port identification, PTP enabled/disabled, PHC/timestamp capability, CPU/IRQ allocations after interface/queue reset, no swap, cyclictest idle and housekeeping load. Capture measured maximum latency and command parameters; release RT workload approval awaits the companion plan's traffic-load/vendor threshold gate.
- [ ] **Step 7: Update acceptance docs during this task with commands, expected state, observed evidence and explicit blocked checks.** Host-only acceptance uses `vms: []`; do not report vendor VM acceptance. Run full coverage, parity/Ansible tests, image lint and available integration tests; commit. Title: `image-mode: validate host lifecycle and boot health`.

## Plan self-review and handoff

| Spec section | Coverage |
|---|---|
| RHEL 10 image, baked RT, package coexistence | Tasks 1, 3, 6, 7 |
| Node schema, renderer, delivery, last-good status | Tasks 2, 4, 5, 8 |
| Dedicated PTP, optional NTP, PRP assumptions | Tasks 1, 4-6; card configuration remains manual |
| OS health and transactional updates | Tasks 1, 8, 9; data/config persistence explicitly distinguished |
| Shared networking/time/domain templates | Task 3; companion plan reuses shared domain source |
| VM delivery, disks, MACs, persistent identity | Companion VM-delivery plan, not claimed by host-only acceptance |
| S1/S2/S4 evidence | Task 1 gates relevant host tasks |
| S3 vendor compatibility/persistence | Companion Task 1 gates vendor/destructive VM acceptance |
| Documentation and tests | Each task owns both; coverage gate applies throughout |
| Cluster/storage/containers | Extension boundaries only; no implementation in these plans |

Execution handoff: use **subagent-driven development**, as requested. Review both plans together before execution; run the host plan to its verified endpoint, then the companion VM plan. Do not launch implementation agents merely because these files exist.
