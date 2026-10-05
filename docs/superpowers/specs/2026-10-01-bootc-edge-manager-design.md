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

## Welotec reference assessment (2026-10-05)

Reviewed [welotec/vPAC-on-RSAPC at commit
6cb52665b71cd656f44ee686a18bbe5333d0c604](https://github.com/welotec/vPAC-on-RSAPC/tree/6cb52665b71cd656f44ee686a18bbe5333d0c604).
The active installation uses `install_packages_deb.yml`, `setup_tuned.yml`,
`start_timemaster.yml`, `assign_interfaces.yml`, and `install_ssc600.yml`.
The last defines `templates/ssc600-deb.xml`; the separate RHEL XML is not
used by that playbook and does not establish RHEL 10 compatibility.

| Area | Executed reference configuration | Design consequence |
|---|---|---|
| Guest NICs | Three virtio NICs: management bridge at guest PCI slot `0x02`, macvtap `process_bus` at `0x03`, macvtap `lan5` at `0x07`. Direct interfaces use `mode="bridge"` and `trustGuestRxFilters="yes"`. | Separate guest NIC model from host attachment. Macvtap is not PCI passthrough. Preserve the guest-visible layout until the ABB interface roles are verified. |
| Physical NICs | Onboard NIC naming matches driver `igc` and per-unit permanent MACs. PRP naming matches `igb` and MAC prefix `70:f8:e7*`. `lanPRP`, `process_bus`, and `lan_ptp` identify the same NIC. | Bake a port-identification policy, not one appliance's permanent MACs. Validate physical port mapping on the MK2. |
| CPU and IRQ allocation | Four vCPUs pinned to CPUs `7,6,5,4`, FIFO priority 50; emulator and PRP IRQs/IRQ threads on CPU 3. CPU script treats `0-2` as non-RT, `3-7` as RT, and offlines `8-15`. | Distinguish housekeeping, RT I/O/emulator, and RT vCPU pools. Reference tuned isolation (`1-7`) differs from the script's pool split; derive a consistent profile rather than copying both. |
| Host tuning | Per-pool frequency controls, SMT sibling offlining, CAT/MSR programming, and disabled KVM halt polling. NIC affinity runs after a fixed 60-second delay. | Bake policy and boot services; validate hardware capabilities and establish explicit readiness before VM startup instead of relying on a delay. |
| PTP | Timemaster uses the PRP NIC also attached to the guest process bus; configuration targets IEC 61850-9-3, L2/P2P. | **User confirmed dedicated onboard PTP on 2026-10-05.** Do not adopt shared PRP/PTP or weaken package-mode PTP isolation. IEC 61850-9-3 and C37.238 are distinct profiles; verify the site's required profile. |
| PTP status | Active Debian XML uses virtiofs, shared/memfd memory, and guest tag `ptp`. Installation copies status script/service from the ABB bundle. | Preserve and validate the vendor status contract. Reference docs describing a 9p share are stale relative to this XML. |
| Disk and watchdog | Raw vendor disk, virtio block, `cache=none`, `io=threads`; `i6300esb` watchdog with reset action. | Support declared disk formats. Reference watchdog behavior conflicts with this repo's intentionally disabled reset watchdogs; do not change that policy without validation. |

The reference uses a writable system disk. It does not demonstrate a separate
SSC600 settings disk, stateless replacement, configuration export/import, or
license survival across disk replacement. These remain gates for VM update
policy, not proven properties of the reference.

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
- **MK2 hardware profile baked:** `isolcpus`/`nohz_full`/`rcu_nocbs`, hugepage count, tuned `isolated_cores`, IRQ pinning, predictable NIC naming policy, PTP port identification — in `kargs.d/` and `files/`. Values come from the hardware spike (S1), including distinct housekeeping, RT I/O/emulator, and RT vCPU pools. Per-unit permanent MAC addresses must not be baked into the shared image. A second hardware model later is a separate small profile layer, not runtime logic.
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

- **Packaging:** each golden disk is an OCI image (e.g. `vpac-vm-ssc600:<ver>`) containing a disk and `vm.yaml` (profile `rt` | `standard`, disk path/format, vCPUs, memory, NIC roles/model/guest PCI placement, disk bus, firmware/TPM needs). Welotec boots ABB's raw disk; qcow2 must not be assumed from a filename. Any conversion or qcow2 overlay on a raw backing disk must declare the correct formats and be validated.
- **SSC600 guest network contract:** virtio NICs with separately declared host bridge/macvtap attachments; direct attachments use macvtap bridge mode and trusted guest RX filters as in the reference. Preserve the three-interface ordering/PCI layout until management, process-bus, and `lan5` roles are confirmed. The host's physical NIC drivers are not virtio.
- **Declaration:** the `vms:` list in `node.yaml`. Upgrading a VM = changing its `image` reference (fleet-wide via template or per device).
- **Execution — host-side `vpac-vm@<name>.service`** (baked):
  1. pull the OCI image with podman (pull secret delivered via RHEM/install config); skip if digest unchanged
  2. extract the declared disk as read-only golden under `/var/lib/vpac/vms/<name>/`, verifying its format
  3. create/reuse the qcow2 overlay according to `diskMode`; create the data disk once if declared
  4. render domain XML from `vm.yaml` + `node.yaml` + baked core layout (profile `rt`: pinned isolated cores, emulator pin, 1 GiB hugepages, locked memory, no memballoon/watchdog, FIFO; profile `standard`: housekeeping cores only)
  5. define and start; on `enabled: false` stop and undefine (disks kept)
- **`diskMode`:**
  - `ephemeral` — overlay recreated on every start
  - `reset-on-upgrade` — overlay kept until the golden image digest changes
  - optional `dataDisk` always persists
- **SSC600 persistence gate:** the illustrative `reset-on-upgrade` entry is not an approved destructive update policy until S3 establishes how IED settings and licensing state survive. A separately attached data disk is not sufficient unless the vendor guest actually uses it for that state.
- **Windows persistence:** preserve per-instance UEFI NVRAM and software TPM state where used, as well as any data disk; an ephemeral system overlay must not implicitly reset those identities.
- **MAC addresses** come only from `node.yaml`; never generated. Use distinct stable addresses per interface. SSC600SW MAC-bound licensing is an operator-reported assumption to verify, not established by the Welotec repository (whose XML leaves MAC selection to libvirt).
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
| S1 | MK2 topology: core/sibling layout, housekeeping/I/O/vCPU pools, frequency and CAT capabilities, stable physical-port identification, dedicated PTP NIC `ethtool -T` capability, PRP interface and guest NIC role/PCI mapping | Validated baked hardware profile and guest device contract |
| S2 | RHEL 10 bootc: RT/NFV repo names, `kernel-rt` packaging, `tuned-profiles-realtime` availability | Working RHEL 10 Containerfile base |
| S3 | ABB SSC600SW on a RHEL 10 host: vendor support; raw disk/overlay compatibility; settings/license persistence; virtiofs status contract and SELinux access; watchdog and disk-I/O policy; GOOSE/SV behavior under load | Validated VM profile and non-destructive update policy; confirm `diskMode` or require vendor-supported export/import |
| S4 | RHEM 1.3: enrollment via bib config, fleet template rendering `node.yaml` from labels, pull-secret delivery | Confirmed delivery mechanism |

## Open decisions (not blocking the lab)

- **SSC600SW licensing and image distribution** (MAC-bound license, customer vs. lab registries). Lab: simple private registry.
- **Per-instance VM config injection** (cloud-init / sysprep / config disk) vs. configuration afterwards via vendor tools — both must remain possible; the `vm.yaml` format reserves a slot for an injected config disk.
- PRP card automation (API availability).

## Out of scope

- Distributed storage (Ceph) on the image-mode path.
- Changes to package-mode behaviour beyond moving shared templates.
- RT containers as workloads.
