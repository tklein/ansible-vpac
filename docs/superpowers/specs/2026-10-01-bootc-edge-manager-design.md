# Design: bootc host image + Red Hat Edge Manager deployment

- **Date:** 2026-10-01
- **Status:** Approved design, pre-implementation
- **Scope:** Sub-project 1 of 4 (see "Decomposition")

## Goal

Deploy vPAC protection hosts as **RHEL image-mode (bootc) images managed by Red Hat Edge Manager (RHEM) 1.3 standalone**, with as much configuration and RT tuning as possible baked into the image and **no Ansible after deployment**. The host runs stateless VMs (golden disk + optional persistent data). Start with a single node; the design must extend to a 3-node cluster with PTP and PRP.

The package-mode Ansible path (`site.yml`, `roles/`) remains a first-class, upstream path and keeps working unchanged in behaviour.

## Decomposition

| # | Sub-project | This spec |
|---|---|---|
| 1 | Host image + per-device config via RHEM (single node) | **Yes** |
| 2 | Workload delivery: stateless VMs (SSC600SW, Windows engineering) | Yes — contract and mechanism; VM-specific details refined during implementation |
| 3 | Time & redundancy: PTP, PRP | PTP toggle + PRP assumptions only |
| 4 | 3-node cluster (HA, optional shared storage) | Extension points only |

## Fixed inputs

| Topic | Decision |
|---|---|
| Edge manager | RHEM 1.3, standalone (RHEL-hosted) |
| Base OS | `rhel10/rhel-bootc` (RHEL 10.x) |
| Hardware | One model: Welotec Rugged Substation Automation Computer MK2 |
| PRP | Welotec HSR/PRP time-aware RedBox DAN PCIe card — PRP in card hardware, standard Ethernet driver on the host, configured via the card's embedded web UI |
| PTP | Dedicated onboard NIC; enabled by default, disable-able per device |
| Workloads | ABB SSC600SW (RT VM), Windows engineering workstation (non-RT VM) |
| Statelessness | Immutable golden disk + small persistent data volume |
| Connectivity | Mixed: connected lab now, private-WAN sites later |
| Install method | Any of: USB ISO (bootc-image-builder), PXE/HTTP boot, factory pre-imaged disk |
| Coexistence | Package mode stays first-class side by side |

## Approach

**Single per-device values file + baked renderer.** RHEM (or the installer, or a factory image) delivers one small file `/etc/vpac/node.yaml` containing only *values*. A renderer baked into the image turns it into concrete OS configuration and applies it. Configuration logic is versioned with the image, testable without RHEM, and shares templates with the Ansible roles.

Rejected:
- *RHEM fleet templates render every config file* — logic in RHEM's template language, not testable offline, unusable without RHEM, duplicates the Ansible Jinja templates.
- *Baked `ansible-pull`* — contradicts "no Ansible after deployment"; mutable drift.

## 1. Image architecture

- Evolve `image-mode/` in place; the existing prototype (kernel-rt swap, kargs, tuned, libvirt, cockpit) is the starting point, rebased onto RHEL 10.
- **Layers**
  - `vpac-node` — FROM `rhel10/rhel-bootc`. kernel-rt (stock kernel removed; exactly one kernel), RT tuning, libvirt (modular daemons), linuxptp, chrony, nmstate, cockpit, greenboot, `flightctl-agent`, `vpac-configure`, `vpac-vm@.service`, shared templates, JSON Schema for `node.yaml`.
  - `vpac-cluster` — FROM `vpac-node`, added in sub-project 4. Single-node images never carry cluster packages.
- **MK2 hardware profile baked:** `isolcpus`/`nohz_full`/`rcu_nocbs`, hugepage count, tuned `isolated_cores`, IRQ pinning, predictable NIC names, PTP NIC name — in `kargs.d/` and `files/`. Values come from the hardware spike (S1). A second hardware model later is a separate small profile layer, not runtime logic.
- **Site-agnostic image:** no hostnames, addresses, credentials, or RHEM enrollment material in the image. `/etc/flightctl/config.yaml` (enrollment) is injected at install time: bib config for ISO/PXE, or written by the integrator for factory images.
- **Registry:** connected lab pushes to a private registry; private-WAN sites use the existing air-gapped mirror flow (`build.sh airgapped`).
- **OS updates:** new image tag → RHEM fleet `os.image` → staged bootc update + reboot → greenboot health checks → automatic rollback on failure.

## 2. Per-device configuration

### Contract: `/etc/vpac/node.yaml`

Versioned (`schemaVersion: 1`) and validated against a baked JSON Schema. Illustrative content:

```yaml
schemaVersion: 1
hostname: sub1-node1
networks:
  mgmt:    {address: 10.1.1.11/24, gateway: 10.1.1.1, dns: [10.1.1.2]}
  station: {address: 10.1.2.11/24, vlan: 20}
ptp:
  enabled: true
  domain: 0
  profile: power-c37.238
time:
  ntpServers: [10.1.1.2]          # used when ptp.enabled is false, and as fallback
vms:
  - name: ssc600
    image: registry.example/vpac-vm-ssc600:1.0
    enabled: true
    diskMode: reset-on-upgrade
    macs: {station: "52:54:00:aa:bb:01", process: "52:54:00:aa:bb:02"}
  - name: win-eng
    image: registry.example/vpac-vm-win-eng:2026.09
    enabled: true
    diskMode: ephemeral
    dataDisk: {sizeGiB: 20}
    macs: {station: "52:54:00:aa:bb:03"}
# cluster: reserved for sub-project 4
```

Physical NIC names are not in the file — they are part of the baked hardware profile; the file references logical networks only.

### Delivery

- **RHEM:** a fleet template renders `node.yaml` from device labels (e.g. `vpac/station-address`, `vpac/ptp=false`); only values live in RHEM. Per-device inline config is allowed for exceptions.
- **Without RHEM:** the same file can be placed by the ISO/PXE install config or by the factory imaging step.

### Renderer: `vpac-configure`

- Python 3 + Jinja2 (RHEL packages). Code in `/usr/libexec/vpac/`, templates in `/usr/share/vpac/templates/`.
- Triggered by `vpac-configure.path` (changes to `node.yaml`) and once at boot (`vpac-configure.service`).
- Sequence: validate → render to a staging directory → diff against live config → apply only changed parts:
  - hostname
  - `nmstatectl apply` (nmstate checkpoint auto-rolls back if connectivity is lost)
  - PTP: enable ptp4l/phc2sys/timemaster when `ptp.enabled`, otherwise chrony with `time.ntpServers`
  - libvirt logical networks (station bridge, process-bus macvtap, optional mgmt)
- **Failure semantics:** invalid file or failed render → nothing applied, last-good config stays active, error logged. A partially failed apply step is reported; nmstate's own rollback protects management connectivity.
- **Status:** journal + `/run/vpac/status.json` (last applied file hash, per-step result).

### Shared templates

nmstate, timemaster/ptp4l, chrony, libvirt-network and VM domain templates move to one shared location consumed by both the Ansible roles and the renderer, so the two paths cannot drift. Roles keep their variable interface; a thin mapping adapts inventory variables and `node.yaml` to the shared template inputs.

## 3. VM workload delivery

- **Packaging:** each golden disk is an OCI image (e.g. `vpac-vm-ssc600:<ver>`) containing `disk.qcow2` and `vm.yaml` (profile `rt` | `standard`, vCPUs, memory, NIC roles, disk bus, firmware/TPM needs).
- **Declaration:** the `vms:` list in `node.yaml`. Upgrading a VM = changing its `image` reference (fleet-wide via template or per device).
- **Execution — host-side `vpac-vm@<name>.service`** (baked):
  1. pull the OCI image with podman (pull secret delivered via RHEM/install config); skip if digest unchanged
  2. extract `disk.qcow2` as read-only golden under `/var/lib/vpac/vms/<name>/`
  3. create/reuse the qcow2 overlay according to `diskMode`; create the data disk once if declared
  4. render domain XML from `vm.yaml` + `node.yaml` + baked core layout (profile `rt`: pinned isolated cores, emulator pin, 1 GiB hugepages, locked memory, no memballoon/watchdog, FIFO; profile `standard`: housekeeping cores only)
  5. define and start; on `enabled: false` stop and undefine (disks kept)
- **`diskMode`:**
  - `ephemeral` — overlay recreated on every start
  - `reset-on-upgrade` — overlay kept until the golden image digest changes
  - optional `dataDisk` always persists
- **MAC addresses** come only from `node.yaml`; never generated. Required because the SSC600SW license is MAC-bound.
- VMs are not RHEM applications; their state is reported via `/run/vpac/status.json`.

## 4. Lifecycle, time, cluster path, testing

- **greenboot checks:** RT kernel running; isolated CPU set and hugepages match the baked profile; `vpac-configure` last run succeeded; enabled `rt` VMs running. Failure → rollback to the previous image.
- **VM updates** are independent of OS updates; the rollout window is governed by RHEM rollout policy.
- **PTP:** templates baked; status continues to be exported to the SSC600 via the existing virtiofs status share.
- **PRP:** done by the Welotec card; the host attaches its interface as station/process bus like any NIC. Card configuration stays manual via its web UI until an automation interface is identified.
- **Cluster (sub-project 4):** `vpac-cluster` layer + `cluster:` section in `node.yaml` (peers, heartbeat NIC). The HA mechanism (Pacemaker vs. alternatives without shared storage) is decided there.
- **Testing:**
  - renderer: schema validation and golden-file render tests, no hardware, runnable locally/CI
  - image: `bootc container lint` during build
  - boot test: qcow2 boot in a VM (existing prototype flow), `vpac-configure` applied from a sample `node.yaml`
  - RT: `cyclictest` and validation checks on a real MK2

## Spikes before implementation

| ID | Question | Output |
|---|---|---|
| S1 | MK2 topology: core count/layout, NIC names, PTP NIC `ethtool -T` capability, PRP card interface name | Baked hardware profile values |
| S2 | RHEL 10 bootc: RT/NFV repo names, `kernel-rt` packaging, `tuned-profiles-realtime` availability | Working RHEL 10 Containerfile base |
| S3 | ABB SSC600SW on a RHEL 10 host: vendor support; where IED configuration is stored (system disk vs. separate) | Confirm `diskMode` for SSC600 or add export/import step |
| S4 | RHEM 1.3: enrollment via bib config, fleet template rendering `node.yaml` from labels, pull-secret delivery | Confirmed delivery mechanism |

## Open decisions (not blocking the lab)

- **SSC600SW licensing and image distribution** (MAC-bound license, customer vs. lab registries). Lab: simple private registry.
- **Per-instance VM config injection** (cloud-init / sysprep / config disk) vs. configuration afterwards via vendor tools — both must remain possible; the `vm.yaml` format reserves a slot for an injected config disk.
- PRP card automation (API availability).

## Out of scope

- Distributed storage (Ceph) on the image-mode path.
- Changes to package-mode behaviour beyond moving shared templates.
- RT containers as workloads.
