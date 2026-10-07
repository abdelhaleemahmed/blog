---
title: "A tour of mhvtl-console: a tape library in your browser"
date: 2026-10-07
categories: ["software"]
tags: ["mhvtl", "tape", "lto", "ltfs", "iscsi", "backup", "bacula", "bareos", "linux"]
---

If you need to test how your backup software handles tape, you need a tape library. A real
one costs as much as a car and arrives on a pallet.
[**MHVTL**](https://github.com/markh794/mhvtl) solves that — it emulates a SCSI medium
changer and its drives in software, well enough that Bacula, Bareos, Amanda and NetBackup
cannot tell the difference. Mark Harvey has maintained it since 2009 and it is genuinely
good.

Configuring it is the part nobody enjoys. You hand-edit `/etc/mhvtl/device.conf`, invent
SCSI target numbers that do not collide, write a second file listing every cartridge, run
`make_vtl_media`, and restart a systemd unit per drive. Get one number wrong and the library
comes up missing a drive, with nothing to tell you which number.

[**mhvtl-console**](https://github.com/abdelhaleemahmed/mhvtl-console) does that part. This
is a tour of what you actually see and do — not how it is built.

## First look

![The dashboard: the MHVTL service, seven libraries, twenty-one drives, a hundred and fifty-six tapes](images/mhvtl-1-dashboard.png)

Whether the service is up, how many libraries exist, how many drives are running, how many
cartridges there are and how much disk they occupy. Then every library in a row, with its
model, serial and state.

Everything on that page is read from `/etc/mhvtl/device.conf` at the moment you load it. The
console keeps a database, but only as a view — if you edit the file by hand, the page agrees
with you on the next refresh. And if a file cannot be read, the page says so rather than
counting it as zero.

## Building a library

![Choosing a vendor and a model; the drive list narrows to what that model takes](images/mhvtl-2-create.png)

Pick a vendor. The model list becomes that vendor's models. Pick a model and the drive list
becomes the drives *that model* takes, with its default already selected. Pick a drive and
the media list becomes the cartridges that drive can write.

**You cannot build a library that does not exist**, because the wrong option is never
offered. MHVTL emulates specific machines — an IBM `03584L32` takes the LTO line, an
`03584L22` takes 3592 drives and no LTO drive at all — and getting that combination wrong
produces a library that starts and then behaves strangely.

There is a reverse mode for when you know the cartridge and not the hardware: say `LTO10`
first and it leaves only the vendors whose libraries can take it.

Then how many drives, how many slots, and how many slots to leave empty. Four is the
default, because a library with no empty slots cannot take a new cartridge without being
reconfigured. Press **Create**, and the console writes both files, starts the daemons, and
makes the cartridges.

## How big is a cartridge?

Each row of cartridges has a size, and it starts at **1 GB** rather than the real capacity
of the generation. That is deliberate, and it is worth a paragraph because the obvious
choice is wrong.

MHVTL's media files are *sparse*: a 12 TB cartridge and a 1 GB cartridge both occupy a few
kilobytes until something is written to them, so a realistic size costs no disk at all. It
costs time. At the 55 MB/s a typical host writes, filling a native LTO-8 takes about **61
hours** — and everything you want to test is at the *end* of a tape. Does the backup span
to the next cartridge? Does it ask for one? Does the drive report end of medium? Does the
fullness bar move? On a cartridge nobody can fill, you cannot reach any of it.

So a small cartridge is the default, and a realistic one is a choice you make:

```console
$ sudo mhvtl settings set tape.size.LTO8 12TB
tape.size.LTO8 is now 12 TB (12,000,000 MB)
Tapes created from now on use this; the ones you already have keep the size they were made with.
```

**System Console → Settings** is the same thing as a page: a row per cartridge type, with
what it will be made at, where that value came from, and what the generation really holds
beside it — click that figure to use it. A library holding two generations gets a size for
each, because one number would give the second kind the first kind's capacity.

And for a single full-size tape without changing anything:

```console
$ sudo mhvtl tape create 50 K50001L8 --size-mb 12TB
```

Sizes are written `1000`, `2000GB` or `12TB` wherever one is asked for, and they are
decimal — an LTO-8 is sold as 12 TB, which is 12,000,000 MB. `12TiB` is refused rather
than quietly converted, because the two are ten per cent apart.

## Building the same one again

![Saved configurations, above the library form](images/mhvtl-3-presets.png)

Name a configuration and it becomes a **preset** — the whole library in one word:

```
$ sudo mhvtl library create --preset ibm-latest --id 90
```

Eleven ship as examples, one per vendor, each at the newest cartridge that vendor's drives
can write. Clicking one fills the form in, where every value can still be changed before you
create anything.

Worth knowing, because it surprises people: **a vendor's default is not its newest.** IBM
defaults to an LTO-8 drive while its catalogue reaches LTO-10. Quantum's default is an
SDLT600, which is not LTO at all — so asking for "a Quantum library" and nothing else gets
you an SDLT library, correctly, and probably not the one you meant.

## Watching a drive write

This is the thing you actually want when you are testing a backup, and it is harder than it
looks: while the backup runs, the kernel holds the SCSI reservation for the initiator, so
`mt status` and `mtx status` answer "Device busy" and tell you nothing.

The console asks the drive's daemon over a message queue instead, which has nothing to do
with SCSI and answers while the backup is running.

![Drive 31 writing, 378.1 MB, 75.8% of the tape, the bar amber](images/mhvtl-4-drive-writing.png)

A line per drive, refreshed every five seconds: what it holds, how much has been written,
and how full the cartridge is. The bar turns amber past 75% and red past 90%. An empty drive
says `empty`; a drive holding a cartridge with nothing happening says `holding`; and the
counters keep the last figures until the tape is unloaded, so a finished backup still shows
what it wrote.

## One library's page

![A library: what it is, its drives, its tapes and its slots](images/mhvtl-5-library.png)

What the library is, what each drive is doing, a tile per cartridge with how full it is, and
links to everything you can do to it — create tapes, move them between slots and drives,
mount, unmount, add a drive, remove the library.

A library can hold more than one generation: two LTO-10 drives and two LTO-9 drives together,
which is what a real site looks like a year after an upgrade. When you do that, one command
answers what the combination can actually load:

```
$ mhvtl tape media 10
DENSITY  SUFFIX  USE         IN DRIVES
-------  ------  ----------  -----------
LTO8     L8      read/write  ULT3580-TD8
LTO7     L7      read/write  ULT3580-TD8
LTO6     L6      read/write  ULT3580-TD6
LTO5     L5      read/write  ULT3580-TD6
LTO4     L4      read-only   ULT3580-TD6
```

The `read-only` row is the one to read. That cartridge can be loaded and read, and the
library will refuse to make a new one — because an LTO-6 drive reads LTO-4 and cannot write
it.

## Losing a tape and getting it back

Delete a library without asking for its media to go, and the cartridges stay on disk with
everything written to them. **Adopting** one puts it back into a library, data and all,
without copying or rewriting anything.

The list only offers the libraries whose drives could actually load that cartridge, and the
command line says where it could go when it refuses:

```
$ sudo mhvtl tape adopt 20 I60006L7
mhvtl: No drive in library 20 loads LTO7 tapes; its drives take AIT4, AIT3, AIT2
LTO7 can go in: library 10 (STK L700), library 50 (STK SL500), library 60 (IBM 03584L32)
```

The test is whether a drive can *read* it, which is looser than whether it can write one —
an LTO-7 drive reads LTO-5, and adopting recovers what is already on a tape.

## Handing it to another machine

![iSCSI: the libraries exported, and who is allowed to connect](images/mhvtl-6-iscsi.png)

A virtual tape library is more useful when the backup server is somewhere else. The console
drives Linux's LIO target to export a library over iSCSI — backstores, targets, CHAP, and the
bindings recorded so they survive a reboot, which by default they do not.

## All of it from the terminal

Everything above has a command, because the web page and the command line call the same code.
Nothing is decided in JavaScript.

```console
$ sudo mhvtl library create --profile IBM --id 50 --drives 2 \
     --media-type LTO8 --drive-model ULT3580-TD8 --tapes 3 --empty-slots 3

ok   validate  Specification is valid
ok   create    Library 50 created: IBM 03584L32 with 2 drives
ok   restart   Restarted vtllibrary@50.service
ok   verify    MHVTL can see library 50
ok   media     3 tape(s) created, 0 already present in library 50
Library 50 created
```

Each step reports, and a failure names the step and leaves the configuration as it was. Add
`--dry-run` to print the `device.conf` it *would* write and stop, or `--interactive` to be
asked one question at a time with only the still-possible answers offered.

## Get it

Rocky Linux, AlmaLinux or RHEL 9, with MHVTL 1.7 or newer:

```bash
sudo dnf install mhvtl-gui-3.4.0-1.el9.noarch.rpm
```

Other distributions get a tarball with an installer. Then open `https://your-server/` — the
first password is `mhvtl`, and it asks you to change it.

- **Source and releases:** [github.com/abdelhaleemahmed/mhvtl-console](https://github.com/abdelhaleemahmed/mhvtl-console)
- **User guide**, in English and Arabic: [abdelhaleemahmed.github.io/mhvtl-console](https://abdelhaleemahmed.github.io/mhvtl-console/)
- **MHVTL itself**, which does the real work: [github.com/markh794/mhvtl](https://github.com/markh794/mhvtl)

The package is still called `mhvtl-gui` — that is the upgrade path for hosts that already
have one. The program calls itself `mhvtl-console`.

How it is built, the rule that held the browser and the terminal together, and the Linux
kernel bug it turned up along the way are a longer piece of their own, which I will put up
separately.
