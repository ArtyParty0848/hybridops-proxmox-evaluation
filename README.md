# HybridOps Proxmox Evaluation

Independent practitioner evaluation of [HybridOps.Core](https://github.com/hybridops-tech/hybridops-core) against a dedicated Dell PowerEdge R740xd Proxmox staging environment.

This project documents a staged evaluation of infrastructure automation covering image lifecycle management, network/IPAM authority, PostgreSQL high availability, and RKE2 deployment.

The existing production Proxmox environment remained outside the test boundary.

---

## Current Status

| Stage | Scope | Status |
|---|---|---|
| 1 | Image & Template Lifecycle | ✅ Passed |
| 2 | Network & IPAM Authority | ⏳ Planned |
| 3 | PostgreSQL HA | ⏳ Planned |
| 4 | RKE2 | ⏳ Planned |

Each stage is reviewed before proceeding to the next.

---

## Test Environment

### Compute
- Dell PowerEdge R740xd
- Dual Intel Xeon Gold 5220R processors
- Dedicated staging host

### Virtualization
- Proxmox VE 9.2.2
- Linux kernel 7.0.2-6-pve

### Automation
- HybridOps.Core 0.1.5
- Packer
- Windows 11 operator workstation
- WSL

### Storage / Networking
- Dedicated staging NFS datastore
- Proxmox bridge networking
- DHCP-enabled staging network
- 10Gb-capable server infrastructure

Internal addresses, credentials, API tokens, private keys, and unrelated production configuration are intentionally omitted from this public repository.

---

## Safety Boundary

The evaluation follows several rules intended to keep testing isolated:

- All workloads run only on the R740xd staging host.
- The existing T630 Proxmox environment remains outside the test boundary.
- Staging-only VMIDs, names, networks, storage, and credentials are used.
- Failed checks are investigated rather than bypassed.
- `--force`, `--skip-preflight`, and equivalent bypasses are avoided.
- Credentials and unrelated internal infrastructure details are redacted from public results.

---

# Evaluation Plan

## Stage 1 — Image & Template Lifecycle

**Status: ✅ Passed**

Lifecycle tested:

```text
build
  ↓
template
  ↓
full clone
  ↓
boot
  ↓
guest-agent readiness
  ↓
cleanup
  ↓
destroy

The final procedural replay successfully completed:

Proxmox target initialization
target preflight
strict module preflight
module plan
Ubuntu 24.04.3 template build
full-clone smoke test
guest-agent IP discovery
guest OS validation
smoke VM cleanup
template destruction
final verification that no Stage 1 VM resources remained
Final Stage 1 Result
Template VMID: 981001
Smoke VMID: 981100
Template: r740xd-eval-ubuntu-2404-template
Guest OS: Ubuntu 24.04.3 LTS
Guest kernel: 6.8.0-147-generic
Smoke lifecycle: ok
Final destroy: ok
Stage 2 — Network & IPAM Authority

Status: ⏳ Planned

Stage 2 will evaluate the HybridOps NetBox/bootstrap path and establish:

Proxmox network foundation
IPAM authority
NetBox
PostgreSQL Core
staging network objects
controlled address allocation
Stage 3 — PostgreSQL HA

Status: ⏳ Planned

Stage 3 will evaluate:

three-node PostgreSQL deployment
Patroni
etcd
leader election
replica health
IPAM-controlled addressing
teardown behavior
Stage 4 — RKE2

Status: ⏳ Planned

Stage 4 will evaluate:

three control-plane nodes
two worker nodes
NetBox-assigned addressing
cluster readiness
node health
automated teardown
Stage 1 Findings

Stage 1 was performed twice for different reasons.

Initial Exploratory Run

The first execution represented the real first-use experience against the new staging host.

The lifecycle ultimately succeeded, but several environmental and automation assumptions had to be discovered manually.

Issues encountered included:

Proxmox snippets storage not enabled by default
partial bootstrap state after a failed initialization
VM storage auto-discovery selecting a different datastore than intended
missing SDN.Use permission for the configured bridge
required Ubuntu ISO not initially present
client-side ISO upload failure
additional permissions required for Proxmox-side ISO acquisition
WSL NAT networking selecting an HTTP address unreachable from the Proxmox guest
strict preflight succeeding despite some runtime prerequisites still being unsatisfied

The initial successful run completed only after those conditions were identified and corrected.

Problems Discovered
Proxmox Permission Model

The automation identity initially had sufficient VM administration permissions but lacked the SDN permission required to attach a VM to the configured bridge.

The missing privilege surfaced only when VM creation was attempted.

Storage Discovery

A configured fallback datastore behaved as a fallback rather than an explicit storage preference.

This was functionally valid behavior but differed from the expectation during the initial test.

ISO Acquisition

The Packer stack supported both client-side and Proxmox-side ISO acquisition, but those controls were not exposed through the normal Stage 1 module input used during the evaluation.

WSL / Packer Networking

The largest runtime issue involved Packer's temporary HTTP server.

Under WSL NAT networking, an internal WSL address was selected for the NoCloud autoinstall source. The Proxmox guest could not reach that address.

The result was a long wait for SSH followed by a timeout.

Remediation

The exploratory run required several changes, including:

enabling Proxmox snippets storage
correcting storage ACLs
adding scoped SDN permissions
validating ISO acquisition paths
enabling required Proxmox-side URL acquisition permissions
changing WSL to mirrored networking
allowing inbound TCP/8802 through the Hyper-V firewall
verifying physical-LAN-to-WSL HTTP connectivity

Once those prerequisites were understood, the environment was reset for a procedural replay.

Clean Procedural Replay

A second HybridOps environment was created specifically to repeat Stage 1 using the published procedure as closely as possible.

Environment:

r740xd-image-eval

The replay successfully completed:

hyops init proxmox
        ↓
target preflight
        ↓
module init
        ↓
strict preflight
        ↓
plan
        ↓
apply
        ↓
smoke validation
        ↓
destroy

One plan attempt failed because I accidentally left the requested vmid and name values commented in the YAML input.

Inspection of inputs_resolved.json immediately confirmed the mistake.

After correcting the input:

strict preflight passed
plan passed
apply passed
Packer connected to the guest over SSH
template creation succeeded
smoke clone VMID 981100 was selected from the requested range
guest-agent IP discovery succeeded
Ubuntu 24.04.3 was identified
smoke cleanup succeeded
template destruction succeeded
final Proxmox verification showed no remaining Stage 1 resources

The complete apply took approximately 19 minutes.

Most importantly, the clean replay succeeded using the original HybridOps/Packer HTTP behavior without the hard-coded HTTP workaround required during the exploratory run.

This indicates that WSL mirrored networking resolved the underlying guest-to-Packer connectivity problem.

Additional Observations

A few smaller items were also observed during testing:

The Stage 1 instructions reference Proxmox/image/checksum fields that are not present in the HybridOps.Core 0.1.5 examples/inputs.typical.yml.
The documented evidence-inspection command uses jq, but jq was not available in the HybridOps WSL environment after the documented setup flow.
Python's built-in JSON formatter was used instead.
Packer emitted a warning that the generated Netplan configuration file permissions were too open.
Smoke cleanup reported purged template vmid=981100 even though 981100 was the temporary smoke clone.

None of these prevented the successful Stage 1 lifecycle.

Lessons Learned

This evaluation has already reinforced several practical infrastructure lessons:

Successful automation depends heavily on validating the environment around the automation.
A successful preflight does not necessarily prove every runtime network or permission dependency.
WSL networking behavior matters when VMs must initiate connections back to tools running on the operator workstation.
RBAC failures are easier to diagnose when permissions are scoped and added incrementally.
Clean staging boundaries make aggressive testing much safer.
Keeping failed run records is often more valuable than keeping only the successful result.
Replaying a workflow after troubleshooting is important for separating environmental prerequisites from actual software defects.

The initial run demonstrated the real onboarding experience.

The procedural replay demonstrated reproducibility.

Both were useful.

Upstream Evaluation

The authoritative evaluation and stage results are being posted to the HybridOps project itself.

Upstream HybridOps discussion:
Add link to the evaluation discussion here

About This Repository

This repository is a portfolio-oriented case study of the evaluation.

It is not a fork of HybridOps.Core and does not claim ownership of the HybridOps project.

The purpose is to document:

environment design
testing methodology
troubleshooting
evidence collection
infrastructure automation behavior
lessons learned during the evaluation
