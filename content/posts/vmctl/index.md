---
title: "vmctl: Virtual Machines as Code"
date: 2026-09-11
categories: ["cloud"]
tags: ["virtualbox", "iac", "automation", "vm"]
---

Spinning up a local VM still feels like 2010. If you run VirtualBox for development,
testing, or disaster-recovery drills, you know the ritual: click through the GUI, or
paste a wall of `VBoxManage` flags from a note you wrote six months ago. Do it once,
fine. Do it for the tenth identical test box — or try to hand a teammate the *exact*
same setup — and it's slow, error-prone busywork.

The frustrating part is that we already solved this — for the cloud. Terraform,
Pulumi, CloudFormation: describe infrastructure as data, review a plan, apply it,
keep it in Git. Reproducible, reviewable, boring in the best way. But the virtual
machines on your own machine never got the memo.

[**vmctl**](https://github.com/abdelhaleemahmed/vmctl) closes that gap. It's a small
command-line tool that manages virtual machines with **config-as-code**: describe a
VM as plain YAML or JSON, keep it in Git, and re-create it anywhere — one machine or
a whole fleet — with the same *plan → apply* discipline that makes Infrastructure-as-Code
trustworthy.

## The core idea: describe, preview, apply

A VM in vmctl is just **data** — CPU, memory, firmware, disks, networks, boot order.
You either write that document by hand or capture an existing machine with `export`.
From there, three verbs:

- **describe** — the config *is* the VM. It's diffable, reviewable in a pull
  request, reproducible on any host.
- **preview** — every state-changing command (`create`, `import`, `batch create`)
  is a **dry-run by default**. It prints the exact provider commands it *would*
  run and changes nothing.
- **apply** — only when you add `--apply` does it actually build.

![vmctl describe → preview → apply](images/vmctl-apply.png)

That dry-run-by-default behavior is the whole safety model — the local-VM equivalent
of `terraform plan`. You always see the concrete commands before reality changes.
The screenshot above is a real run: an `import` prints the ten `VBoxManage` commands
it would execute, and nothing happens until `--apply` is on the line.

## Config-as-code, concretely

A VM's full shape lives in one readable document:

![vmctl read — a real VM config as YAML](images/vmctl-config.png)

```bash
# Capture a live VM into a versionable file, then commit it
vmctl export web-server -o infra/web-server.yaml
git add infra/ && git commit -m "Add web server VM definition"
```

Only `name` is strictly required; everything else falls back to sensible defaults.
Now your VM definitions are part of your codebase — reviewed, diffed, versioned —
instead of living in your memory and a scattering of shell snippets.

## It models real hardware — and checks it

vmctl isn't a thin wrapper around `VBoxManage`. It has an actual model of VM
hardware, and it **validates a config against the target provider's capabilities
before it builds anything**, so you don't discover a bad disk controller halfway
through:

| Category | Supported |
|---|---|
| **Storage controllers** | IDE, SATA, SCSI, SAS (up to 255 disks/controller) |
| **Drives** | HDD, SSD, CD/DVD |
| **Disk formats** | VDI, VMDK, VHD, RAW — thin (dynamic) or thick (fixed) |
| **Firmware** | BIOS, EFI (32/64), Secure Boot, TPM |
| **Networking** | NAT, Bridged, Host-only, Internal, NAT Network (up to 8 adapters) |
| **Boot devices** | disk, dvd, floppy, network |

`vmctl validate <file>` runs those checks on demand; `vmctl read` and `vmctl list`
show you what's already there.

## Clone a VM to test scenarios

Sometimes you don't want a new VM from scratch — you want *this* one again, to try
something risky without touching the original. `create` clones an existing VM's
shape into a fresh replica, and lets you tweak it on the way:

![vmctl create — clone a VM for testing](images/vmctl-clone.png)

```bash
# Same shape as the original, but 4 GB / 4 vCPUs, under a new name
vmctl create master.puppet.vm --new-name puppet-test --memory 4096 --cpus 4 --apply
```

Stand up as many identical-shaped replicas as you need — to test an upgrade, a
config change, or a failure scenario — while the original stays exactly as it was.
It's an export/import under the hood, so the clone is defined by the same
config-as-code you can commit and share.

## Whole fleets from one file

Batch operations turn a single template into many VMs, each with its own overrides —
inherit a base and vary what matters:

```yaml
# cluster.yaml
name: dev-cluster
base_vm: ubuntu-template
instances:
  - name: dev-web-01
    memory: 4096
    cpu: 4
  - name: dev-db-01
    memory: 8192
    cpu: 8
    disks:
      - size_mb: 102400
```

```bash
vmctl batch create cluster.yaml --apply   # web + db, stamped out in one shot
```

## The full lifecycle, from one CLI

Beyond describe/apply, vmctl covers the day-to-day:

![vmctl — the real VM inventory](images/vmctl-list.png)

`list`, `read`, `start`, `stop` (graceful ACPI or `--force`), `status`, `edit`,
`create`, `export`, `import`, `delete`, `validate`, and `batch` — one tool instead
of a mental map of `VBoxManage` subcommands.

## Built to grow beyond one hypervisor

The architecture is deliberately layered: a **core engine** and config model,
pluggable **providers** behind a thin interface, **serializers** for YAML/JSON, and
**validators** for capability checks. VirtualBox is fully supported today;
**libvirt/KVM and QEMU are on the roadmap** — and because the provider seam is thin,
the same commands and the same configs will carry across them. Adding a hypervisor
means implementing one small interface, not rewriting the tool.

## Where it earns its keep

- **Reproducible dev environments** — one `batch create` and the whole team is on
  an identical setup, defined in a file you can review.
- **Disaster-recovery / bare-metal-recovery testing** — export a production VM's
  shape, spin up a clone, validate the restore, throw it away.
- **Infrastructure in Git** — VM definitions become code: reviewed, diffed,
  versioned, reproduced on demand.

## Get it

vmctl is MIT-licensed and needs only Python 3.8+ and PyYAML (plus VirtualBox to
actually drive VMs):

```bash
pip install https://github.com/abdelhaleemahmed/vmctl/releases/download/v2.0.0/vmctl-2.0.0-py3-none-any.whl
```

- **Code:** https://github.com/abdelhaleemahmed/vmctl
- **Docs:** https://abdelhaleemahmed.github.io/vmctl/

If you manage local VMs and wish they behaved more like code, give it a try — and if
you want a provider it doesn't have yet, the interface is small and the PRs are
welcome.
