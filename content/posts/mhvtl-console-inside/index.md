---
title: "Inside mhvtl-console: one rule, two front ends, and a kernel bug"
date: 2026-10-07
categories: ["software"]
tags: ["mhvtl", "tape", "lto", "python", "django", "architecture", "testing", "kernel", "linux"]
---

*A web console and a command line for a virtual tape library — and the rule that held them together*

2026-09-22, written at version 2.1.1 · updated 2026-10-07 at 3.4.0

## The question that started it

A tape library is a cabinet: shelves of cartridges, a few drives that read and write them,
and a robot arm that moves cartridges between the two. Backup software talks to it over
SCSI, the same protocol a disk uses. Real ones cost as much as a car and live in rooms with
raised floors.

MHVTL is a Linux program that pretends to be one. A kernel module and a set of daemons
emulate the robot and the drives; the kernel sees real SCSI devices, and when backup
software writes to a "tape", MHVTL writes a file on disk instead. Bareos, Bacula, Amanda and
their commercial cousins never know the difference. It is how you test a backup strategy
without buying a tape robot.

It is also administered entirely by hand. A library is a record in
`/etc/mhvtl/device.conf`; what sits in its slots is `/etc/mhvtl/library_contents.N`;
the daemons are `systemctl` units; moving a cartridge is `mtx`; asking a drive anything is
`mt` or `sg_*`. None of that is hard, but all of it is easy to get slightly wrong, and
nothing tells you when you have.

So the question was simple: *can you run MHVTL from a browser, without ever losing the
ability to run it from a terminal?*

MHVTL Console is the answer. It is public, at version 3.4.0:

- source and releases: `github.com/abdelhaleemahmed/mhvtl-console`
- documentation: `abdelhaleemahmed.github.io/mhvtl-console`

This article is how it is built, what went wrong on the way, and the one rule that
decided nearly everything.

> **Written at 2.1.1 in September 2026, and kept current.** The figures below are
> remeasured; five sections at the end — *Profiles and presets*, *One frame for fifty
> pages*, *The kernel bug it found*, *How big is a cartridge?* and *What changed since
> 2.1.1* — were added in October and say what arrived after the original was written. The
> cartridge-size one also says where the rule this article is about ran out.
> Nothing in between has been removed, so where an early section describes how something
> was in September, the later sections say what it became.

## By the numbers

| | |
| --- | --- |
| Library vendors it knows | 9 — IBM, StorageTek, HP, Quantum, ADIC, Spectra Logic, Overland, Dell, Sony |
| Library combinations built end to end | 85 (vendor × model × cartridge family) |
| `mhvtl` command nouns | 14 |
| Colour themes | 4 |
| Tests | 2,126, in about two minutes, with no MHVTL and no root |
| Python versions in CI | 3.11, 3.12, 3.13 |
| Documentation trees | 5, each built in English and Arabic (right-to-left) |
| Public release | 3.4.0, GPL-2.0-only |
| Lines of code | 45,340 of application Python — breakdown below |

The code, counted from the committed source, comments and docstrings included:

| Part | Files | Lines |
| --- | ---: | ---: |
| Application Python, without the tests | 250 | 45,340 |
| &nbsp;&nbsp;of which the service layer | 90 | 22,048 |
| &nbsp;&nbsp;of which the web views | 11 | 9,134 |
| &nbsp;&nbsp;of which the `mhvtl` command | 23 | 4,764 |
| Tests | 92 | 26,181 |
| HTML templates | 66 | 14,309 |
| CSS | 9 | 3,545 |
| JavaScript | 12 | 1,180 |
| Packaging — RPM, installer, build | 15 | 2,265 |
| **Total** | **444** | **92,820** |

*(Remeasured October 2026 at 3.4.0. At 3.3.1 the same count was 438 files and 89,644
lines; the September figures at 2.1.1 were 299 files and 59,923 lines. The counts exclude
the documentation trees and the vendored `asciinema-player` assets in the recording kits,
which would otherwise add about twelve thousand lines of somebody else's CSS.)*

*Each row is one path rule, applied to the committed tree at the release tag, so the
figures are comparable between releases rather than remeasured a different way each time:
the service layer is everything under `services/`, the web views are `views.py` and
`*_views.py`, the command is all of `mhvtl_cli/`, and the tests are every `tests/`
directory under `apps/`.*

## What it does

From the dashboard and one page per library, and equally from the `mhvtl` command:

- **create libraries** from nine vendors' profiles — IBM, StorageTek, HP, Quantum, ADIC,
  Spectra Logic, Overland, Dell and Sony — and delete them again;
- **fill them with tapes**, with barcodes that say their generation (`K50001L8` is an
  LTO-8 cartridge);
- **move cartridges with the robot**: load a drive, unload it, move between slots;
- **export a library over iSCSI**, so another machine sees a real tape library, and take it
  back cleanly;
- **watch each drive while a backup writes to it** — writing, reading, holding a tape or
  empty, and how full the cartridge is;
- see the host itself: the daemons, the kernel modules, the SCSI devices MHVTL made, the
  disk the tapes live on.

Creating a library, from the terminal, looks like this — a real run:

```console
$ sudo mhvtl library create --profile STK --id 50 --drives 2 \
     --media-type LTO8 --drive-model ULT3580-TD8 --tapes 3 --empty-slots 3

ok   validate  Specification is valid
ok   create    Library 50 created: STK SL500 with 2 drives
ok   restart   Restarted vtllibrary@50.service
ok   verify    MHVTL can see library 50
ok   media     3 tape(s) created, 0 already present in library 50
Library 50 created
```

And watching it, while a backup writes to it:

```console
$ sudo mhvtl status activity 50

Library 50: 1 of 2 drive(s) hold a tape, 1 working

DRIVE  STATE    BARCODE   COUNTERS
-----  -------  --------  --------------------------------------------
51     writing  K50001L8  300.0 MB written - 29.6% of the tape
52     empty    -         -
```

The web page shows the same thing, as a panel that refreshes every five seconds, with a
bar for how full the tape is. It is the same answer, from the same code — and that is the
whole story.

The dashboard: the MHVTL service, every library, and the common tasks.

![The dashboard: the MHVTL service running, four libraries, fourteen drives and ninety-nine tapes](images/dashboard.png)

One page per library, with everything that can be done to it and the library already
chosen:

![Library 50: what it is, its slots, and the links to manage its tapes, drives and robot](images/library-page.png)

## The rule

Every console like this one starts the same way. The page needs to show whether a drive
is busy, so a little JavaScript fetches two readings, compares them, decides the drive is
writing, formats the bytes, picks a colour. It works. It demos well.

And then somebody asks for the same information on the command line, and there is no
command that can answer, because the answer only exists inside a browser. The only way to
provide it is a second copy of the rule, in Python — and two copies of a rule do not stay
equal. They only appear to, until a threshold is adjusted in one of them. In this project
it happened for real: the same cartridge came out *filling* on one card and *normal* on the
one beside it, because tape tiles and the drive panel each had their own idea of "nearly
full".

So the project runs on one rule:

> **The services decide. The page and the command line only display.**

Every decision — what a drive is doing, how full a tape is, what "full" means, which
drives a library model can take, whether a barcode is valid, how a size is written — is
made once, in a Python service layer. The web views and the CLI call it and print what it
returns. JavaScript on the page swaps in HTML the server rendered and remembers which
colour theme you picked. It computes nothing.

The consequence is that the command line can never fall behind the pages. Anything the
console can do, `mhvtl` can do, because there is only one implementation of it.

## Three layers

![The three layers: a browser and a terminal, Django views and the mhvtl command, one service layer where every decision is made, and MHVTL underneath](images/architecture.png)

Almost half of the application's Python is the service layer: 22,048 of 45,340 lines. The
views that serve every page are 9,134, and the whole command line is 4,764 — small,
because neither decides anything. And the tests, at 26,181 lines, weigh *more* than the
services, deliberately — more on that below.

### Every service returns the same shape

```python
@dataclass
class ServiceResult:
    success: bool
    message: str                    # one sentence, for a person
    data: Optional[Dict] = None     # what the caller asked for
    errors: List[str] = field(default_factory=list)
    operation_id: Optional[str] = None
```

Not a bare dictionary, not `None`, not an exception for an ordinary failure. *There is no
library 99* is an answer to a question, not an exceptional event, so it comes back as a
value — `message` is the headline, `errors` is the help (*device.conf has 10, 20, 30*).

That shape is what makes the command line cheap. One function prints any result, as text
or as `--json`, with the right exit code, without knowing which service produced it. And a
web view can render the ordinary page with the failure in it, in the console's own frame
and theme, with status 404 — instead of Django's bare error page.

The module that defines this type carries a scar in its docstring: before the refactor it
was imported by *nothing*, while four rival `ServiceResult` classes grew in different
modules, each with its own idea of the fields. A caller had to know which service it had
called to read the answer. Collapsing them onto one is what made the CLI possible.

### Every command goes through one function

`core/shell.py` is the only module in the service layer that imports `subprocess`. It
enforces three things no call site can forget:

- **a timeout, always** — a wedged `mtx` otherwise holds a web worker until someone
  restarts the service;
- **a list, never a string** — a barcode with a semicolon in it stays a barcode;
- **one result type** — `.ok`, `.output`, and a failure that quotes its own command.

It also makes the suite possible, because a test can hand back what a command *would*
have printed without running it. More on that too.

### The CLI is a noun, then a verb

`mhvtl library create`, `mhvtl tape list 50`, `mhvtl status activity 50`,
`mhvtl iscsi unexport 50` — fourteen nouns (`library`, `drive`, `tape`, `operations`,
`status`, `service`, `config`, `scsi`, `iscsi`, `console`, and `profile`, `preset`,
`ltfs` and `settings`, which arrived later), each a module that registers
its verbs. Reading is open; every verb that changes something starts with a permission
check that says what would allow it — *creating a library needs membership of the mhvtl
group; you are …, in …. Run "sudo usermod -aG mhvtl …" and log in again, or run this
under sudo* — rather than just *permission denied*. And it reads
`/etc/mhvtl` by default, whatever the web server's settings say: a command that decides
what to do from a stale copy and then acts on real systemd units is dangerous.

## Watching a drive: the hardest feature

A drive does not report *writing*. That is a conclusion.

MHVTL's drive daemons keep counters — bytes written, bytes read — and we added a way to
ask for them: `vtlcmd <drive> stats`, our patch 0004 to MHVTL. The kernel holds a SCSI
reservation for whoever is running the backup, so `mt` and `mtx` cannot ask the drive
anything while it works; the daemon's own counters can. Compare two readings a few seconds
apart: if *written* grew, the drive is writing.

We proved it end to end with a real 600 MB write, watching the bar move from 0% to 59.2%.
And then it started lying.

**The state flickered.** During a steady backup, the panel alternated between *writing*
and *holding a tape*. Two causes, at once:

1. The console runs under gunicorn with three worker processes. Each kept its own memory
   of the last reading, so consecutive requests — landing on different workers — compared
   against different pasts.
2. MHVTL's counters are buffered. Two readings less than a second apart can return the
   same number from a drive that is busy.

The fix is a small shared store — a file — for the last reading of each drive, with two
rules: a sample younger than three seconds is not worth comparing against, and a tie keeps
the last state rather than falling back to *holding*. The file is readable and writable by
both the web service and a root `mhvtl` on the command line, so the terminal and the page
see the same history.

**A drive that had written nothing showed a dash**, not *0 MB*. The template used
Django's `|default` filter, which fires on anything false — and `0` is false. The fix is
`|default_if_none`. The same trap had already cost the library page a correct free-slot
count once: a full library, with zero free slots, showed a stale stored number instead of
0.

The whole decision now lives in one service method, `activity()`. The page asks for a
server-rendered fragment every five seconds and swaps it in; `mhvtl status activity` prints
the same values. There is no JavaScript that compares counters.

![Drive 31 writing G03001TA: 173.8 MB written, 14.8% of the tape, 2.35x compression](images/drive-writing.png)

## How full is a tape that is sitting in a slot?

A drive can tell you how much is on the tape in it. A cartridge on a shelf has no drive.

MHVTL keeps each cartridge as a directory of files, and one of them, `mam`, is the
cartridge's *Medium Auxiliary Memory* — the chip a real LTO cartridge carries, with its
attributes. It is binary: an eight-byte header, then a run of attributes, each an id, a
length and a value. Attribute `0x0000` is *remaining capacity*, `0x0001` is *maximum
capacity*, each an eight-byte big-endian number in MiB. Read those, and a tape in slot 12
has a size — the tape tiles show a bar, and `mhvtl tape list` stopped printing `-` in its
capacity column for every tape not in a drive.

![Two tape tiles, each with a bar for how much of the cartridge is used and how much room is left](images/tape-tiles.png)

The files belong to root; the console does not run as root. It reads them through `sudo`,
and a `sudoers` file lists exactly which programs it may run that way. The first version
used `sudo sh -c 'cat …'`, and was caught by a test that checks every command the code runs
under `sudo` against that list: `sh` is not on it, and should never be. It was rewritten to
read a whole library's MAM files with a single `tar -cf -`.

A tape's path is built from its barcode and later handed to `rm -rf`, so it is checked
twice: the barcode must be a barcode, and the resolved path must be directly inside the
media directory. That is one of the two places the service layer *does* raise — because a
caller that gets there with a bad path has a bug, not a bad day.

## Knowing the hardware

`device.conf` records what somebody chose. It does not know what was allowed. Which
drives does an STK SL500 take? Which cartridges does an LTO-7 drive write, and which can it
only read?

The console carries its own knowledge, and none of it was typed out by hand. The profiles
were read out of MHVTL's source — `usr/pm/*.c`, where each library *personality* is
registered — and the compatibility tables follow the drives' actual rules.

That matters because the rules of thumb are wrong exactly where people rely on them. The
famous one for LTO is *a drive writes its own generation and one back, and reads two
back*. It holds for LTO-3 through LTO-7. **LTO-8 broke it**: an LTO-8 drive reads nothing it
cannot write, and cannot read an LTO-6 cartridge at all. A formula would be confidently
wrong in exactly that place; a table is not. (We found this the embarrassing way: a first
draft of the project's tutorial applied the rule of thumb, said an LTO-8 drive reads LTO-6,
and had to be corrected against the real tables.)

A companion article, *Every Library, Through the Browser*, describes building every
library the console offers — 85 combinations of vendor, model and cartridge family — and
the three crashes in MHVTL that turned up doing it.

## iSCSI: exporting a library, and taking it back

A virtual library is most useful when a *different* machine can use it — the backup
server, not the MHVTL host. The console exports a library over iSCSI with `targetcli`: a
target, and a *backstore* for the robot and for each drive, all exposed to one initiator.

Taking it back taught the sharpest lesson in the project. **Deleting an iSCSI target
leaves its backstores behind**, still holding the SCSI devices open. The library looks
gone; its devices are not free. The fix is `mhvtl iscsi unexport <library>`, which removes
the target *and* the backstores that belong to it — recognised by a single naming rule,
`lib<id>_changer` and `lib<id>_drive<n>`, written once as a regular expression. An early
version used two different rules for "is this ours?" in two places, and one of them
accepted `lib50_drivefoo`; that is now a test.

## One look, four themes

The console has four colour themes — Console Navy, Graphite, Daylight and Sepia — and
none of its rules names a colour. Every colour is a token (`--surface`, `--accent`,
`--ok`, `--warn`, `--bad`), and a theme is only a different set of values for the same
names. A service says a tape is `full`; the template writes that word as a class; the
stylesheet gives the class `--bad`; the theme says what `--bad` looks like. Changing the
threshold is one line of Python; changing the colour is one line per theme.

Two details earned their place the hard way:

- **`--accent-ink`**, the colour of text *on* the accent. The header is accent-coloured;
  in Daylight the text on it must be white, in Console Navy dark. A theme without this
  token has a header you cannot read, in exactly one theme.
- **Every rule is scoped to a class that only the base template sets.** That makes a page
  that skipped the base template fail *visibly* — unstyled — instead of half-working. The
  project had exactly the half-working kind: dialogs with their own `<head>`, white in a
  dark console, which looked fine in the theme their author used. Converting them took a
  wave of the refactor per area.

## Traps we hit, so you don't

Some of these are specific to Django or MHVTL. Most are not.

**A blank page reads as a broken console.** A URL that only takes POST — a form's action —
answers a browser's GET with 405 and an *empty body*. Someone who bookmarks it, or presses
Enter in the address bar after submitting, gets a white page. Those URLs now redirect to
the page the form lives on; the ones JavaScript calls answer with JSON saying what they
take. A test checks every such URL.

**An upgrade must restart the service.** gunicorn's workers keep the code they started
with, so version 2.1.0 was installed and the footer went on saying 2.0.0 — on a host where
every file was already 2.1.0. It reads as the version being wrong, not the service being
old. The package now runs `systemctl try-restart` on upgrade only (not on first install,
where there is nothing to restart), and a test checks it.

**Static files must not be `immutable` unless their names change.** nginx served the
stylesheets and scripts with a thirty-day `immutable` cache, under names that never
change. After an upgrade, anyone who had visited before kept getting the previous
release's JavaScript — and the new feature looked broken. They are now `public, no-cache`
(revalidate every time, still cached), and a test reads the nginx configuration.

**Numbers are written in the reader's language — including inside `style`.** A bar's
width was `style="width: {{ percent }}%"`. Django prints numbers in the active language,
and in Arabic 81.9 is *81,9*; `width: 81,9%` is not a CSS length, so every bar draws
empty, silently, and only in Arabic. The console is being translated, so every number in a
style now goes through `|unlocalize`, and a test renders the pages in Arabic and scans all
templates for a width written the old way.

**A page can ask for fields that do not exist, and say nothing.** Django renders an
unknown variable as an empty string. The disk usage page read `percent_used` and
`total_mb`, which the disk service never had — so on every host it showed an empty bar, a
bare "%" and " MB" three times. The dashboard had had the same bug and been fixed; the
separate disk page was missed. Only a test that renders the page finds this, which is why
there are page tests beside the service tests.

**Documentation commands rot.** The documentation once showed `mhvtl iscsi unexport 50
--json`, which fails: `--json` is a global option and goes *before* the noun. Another page
documented an `mhvtl -V` that did not exist. There is now a test that finds every `mhvtl`
command in the documentation and checks that it parses.

**A test can pass only on the machine it was written on.** When the project got public
CI, two tests failed on clean machines. One used an f-string with a backslash in its
expression — legal only from Python 3.12, and the CI also runs 3.11. The other called the
real `media.list_media`, which runs `sudo find`; it passed on the development host, where
`sudo` needs no password, and found nothing anywhere else. The suite's own promise was that
every external tool is faked. Now it is.

## The tests

2,126 tests, and they run in about two minutes on a laptop with no MHVTL, no tape
library and no root — because every command goes through `shell.py`, and every test fakes
it. (There were 1,225 when this section was first written, at 2.1.1.) `mtx`, `mt`,
`lsscsi`, `vtlcmd`, `targetcli` and `systemctl` all answer from captured output of the
real tools. The test settings never touch the real `/etc/mhvtl`: every run copies the
fixtures into a fresh temporary directory.

Beside the feature tests is a set of **guard tests** — tests of rules rather than
features, and the most useful idea in the project:

- every command run under `sudo` is on the `sudoers` list;
- every `mhvtl` command shown in the documentation exists, with those options, in that
  order;
- no POST-only URL answers a browser with a blank page;
- the package installs what the build produces, and a fresh clone passes the suite;
- the web server does not cache unhashed static files as immutable;
- no template prints a number into a style without `|unlocalize`;
- the beginner tutorial's code still runs, chapter by chapter.

Each is the same move: something agreed once, asserted by a test, so that it does not
have to be remembered.

And one discipline for the tests themselves: **for every rule you care about, break it
once and check that a test goes red.** Writing the tutorial's test chapter, two tests
that looked like guards turned out to guard nothing. Moving the *full* threshold from 90%
to 95% passed, because the test checked 80 and 95 — comfortably inside each band. Removing
an `int()` that protects against string ids passed, because every caller converted first.
Both now test the exact boundaries, and both were run against the broken code before being
trusted.

## Documentation, in two languages

Five documentation trees, each an independent Sphinx project with its own search and its
own Arabic catalogue: a User Guide (with the command-line equivalent of every task), a
Developer Guide, an API reference built from the docstrings, the original guides, and a
beginner tutorial that builds a small version of the console from an empty directory —
fifteen chapters, each ending with something that runs, and a machine that follows them
one at a time and checks that each chapter's code works at its end.

The User Guide and the API reference are published first, as HTML on GitHub Pages. The
Arabic builds are laid out right-to-left and the translation catalogues exist, one per
page — but the words are still English, waiting for translators.

### A tutorial is code, so it gets tests

The tutorial's first draft read well and did not work. Followed in order, a reader
typing along hit an `ImportError` at chapter 7 and again at chapter 8: the web view
imported the tape service at the top of the file, and the tape service was not written
until chapter 12. One chapter wrote a module nothing ever called, another wrote nothing
new at all, and a third built data that no page or command ever showed. All of it had
been checked — but against the finished code, which was already on disk, so each chapter's
code only ever ran with every *later* chapter's code beside it.

The fix was to stop trusting the prose and test it, twice over:

- **The walk.** Each chapter's files live in their own folder, as the reader should have
  them at the end of that chapter. A test copies them into one empty directory a chapter
  at a time — the way a reader's tree grows — and after each chapter runs the command that
  chapter ends with: `library list` at 6, `drive list 10` at 7, `manage.py check` at 11,
  `manage.py test` at 14. Then it runs all of them again on the finished tree, so a late
  chapter cannot break an early one. Thirty steps, a few seconds. Its first run found that
  a package's `__init__.py`, copied from the finished code, imported a file that chapter 2
  had not yet written.
- **The prose check.** The walk proves the downloadable files run; it cannot prove that
  the code *printed in the chapter* is those files. A second test takes every code block
  in every chapter — whole files, excerpts, and diffs against a file the reader already
  has — and checks that it appears in the chapter's files, allowing an excerpt to be shown
  without its surrounding indentation. 142 blocks. On its first run it found a diff in
  chapter 9 whose lines were one space short of the file's: a reader comparing their code
  against the chapter would have found a difference that was not really there.

Documentation that teaches code is code. If nothing runs it, it is already wrong; you just
have not found out where.

## Packaging and release

The console installs as an RPM on Rocky Linux, AlmaLinux and RHEL 9 — Python 3.12,
gunicorn behind nginx on ports 80 and 443, one shared password (`mhvtl` until you change
it, and every page warns until you do). There is also a tarball with an installer for
other distributions.

There is exactly one version number, in `mhvtl_system/__init__.py`. The build script
reads it and stamps it into the spec; the documentation reads it as text; the page footer
gets it from the application. It used to be written out by hand in four places, which is
how the console once said *Version 1.0.0* under packages that said 2.0.0. A test checks the
two places that cannot read it — the spec's own `%define` and its changelog.

The first public release is 2.1.1: one commit, the code, its packaging and tests, the
User Guide and the API reference. The CI runs the suite on Python 3.11, 3.12 and 3.13 and
builds the documentation with warnings as errors. It is licensed GPL-2.0-only — the same
licence as MHVTL, whose `COPYING` carries its author's note that version 2 is the only
valid one.

The public repository has no history, deliberately. The project was developed for more
than a year in a private repository, with the Developer Guide, the tutorial and the original guides
still to be reviewed before anyone else reads them. So the public copy is a *snapshot*: a
script takes a tagged release with `git archive` — which can only see committed, tracked
files, so nothing half-written in a working folder can leak — keeps the application, its
packaging and tests, and only the documentation trees on an approved list, and makes that
the first commit of a new repository. Publishing another tree later is adding its name to
the list. Even the tests follow: they read the list of documentation trees from the
checkout itself, so the public copy checks the two trees it has, and the private one keeps
checking all five.

The documentation website is built from that same snapshot, never from the private
checkout, so a page that is not published cannot reach it. And GitHub Pages has one trap
worth knowing: it runs Jekyll by default, and Jekyll drops every folder whose name starts
with an underscore — which is where Sphinx keeps its stylesheets, `_static/`. Every page
loads, unstyled. An empty file called `.nojekyll` at the root of the site turns Jekyll off.

## The pattern, if you want to steal it

1. **Put every decision in one layer, and make every front end call it.** If you catch
   yourself writing an `if` about the domain in a template or a script, the command line
   just lost a feature.
2. **Give every service one result shape** — success, a sentence for a person, the data,
   and the reasons. Callers stop needing to know who they called.
3. **Run every external command through one function**, with a timeout, as a list, with
   one result type. It is the difference between a suite that needs the hardware and one
   that runs anywhere.
4. **Make decisions words, and let the presentation choose what the words look like.** A
   service returns `full`, not `#ef4444`.
5. **Take hardware facts from a source, and name the source in the code.** A rule of thumb
   is wrong exactly where people lean on it: the LTO one holds for five generations and
   then breaks at LTO-8, which is the generation most people are buying.
6. **Turn every agreement into a guard test**, then break the rule once to prove the test
   notices.
7. **Treat "only on my machine" as a bug.** A clean machine — a CI runner, a colleague's
   laptop — is the real test of whether your tests test the code or the host.

---

*The sections from here were written in October 2026, at 3.3.1 and 3.4.0. Everything above
is as it was at 2.1.1, with the figures remeasured.*

## Profiles and presets: two nouns, added in 3.2.0

The article above talks about "profiles" loosely, meaning the vendor catalogue. Version
3.2.0 made that precise, because two different things were wearing one word.

A **profile** is a vendor's catalogue — what IBM makes. It describes hardware that exists,
it ships with the console, and it has **no write verbs at all**: no `profile set`, no
`profile delete`. Nothing may edit what a vendor makes. If a drive is not in the list, the
vendor does not make it, and inventing one produces a library that identifies itself as
hardware that does not exist.

A **preset** is a set of choices made from a profile, under a name, and it is the
operator's. Build one a piece at a time, or keep the configuration a create has just
proved:

```
$ sudo mhvtl library create --profile IBM --model 03584L32 \
     --drive-model ULT3580-TDA --media-type LTO10 --drives 2 --tapes 20 \
     --id 90 --save-preset ibm-latest
```

The separation is the interesting part, and it followed from the rule: a preset may not take
a profile's name, because that would make the two words mean the same thing again.

```
$ sudo mhvtl preset set IBM --drives 2
mhvtl: 'IBM' is a vendor profile, not a configuration
  a preset is built from a profile and cannot share its name
  try a name of your own: ibm-small
```

Eleven presets ship as examples, one per vendor, each at the newest cartridge that vendor's
drives can **write**. Three are not the obvious row: HP stops at LTO8 because HP's catalogue
does, Sony is AIT4 rather than LTO at all, and StorageTek gets two because an SL500 takes
both its own T10000C and an IBM LTO drive.

One of them is a trap worth stating plainly: **a vendor's default is not its newest.** IBM
defaults to an LTO-8 drive while its catalogue reaches LTO-10, and Quantum's default is an
SDLT600, which is not LTO at all. `--profile QUANTUM` on its own builds an SDLT library —
correctly, and probably not the one you wanted.

3.2.0 also added `mhvtl library create --interactive`, which asks one question at a time and
offers only what is still possible: answer IBM, then `03584L22`, and the drive list is four
3592 drives and no LTO drive, because that is a 3592 library. Nothing is written until the
last answer.

## One frame for fifty pages: what 3.3.0 was

The console had **six complete HTML documents**, each with its own header. That is how the
version came to appear on 8 pages of about 46 and on none of the operator pages — the footer
that carries it was included by one of the six.

3.3.0 replaced all of them with one `base.html` extended by a thin wrapper per section, and
the symptoms it cured say more about the shape of the problem than the fix does:

- **"Libraries" led to two different pages**, depending on which header you clicked it from.
- **The create-a-library form was unreachable** from most of the console.
- **Three pages had no inbound link at all.**
- **The landing page's five numbers had never been anything but zero**, through two faults at
  once: the context keys were never set, and the poller read keys the endpoint did not
  return. Both counted database rows rather than `device.conf`.

A test now walks the URL resolver and renders every page it finds, so a page cannot be added
outside the frame quietly. That is rule 6 from the list above, applied to a thing that had
gone wrong six times.

The same release let a library hold **two generations of drive and tape** — two LTO-10 drives
and two LTO-9 drives in one library, which is what a real site looks like a year after an
upgrade — and made adopting a loose tape offer **only the libraries whose drives could load
it**, which is where the service already drew the line while the form offered every library
and said so afterwards.

It also renamed the product. `mhvtl-gui` is the name of an older, unrelated PHP interface to
MHVTL, so a search found both and a bug report quoting a version did not say which program it
was about. The **package** keeps the old name on purpose — the RPM, `/opt/mhvtl-gui`, the
service account and the unit are an upgrade path on every host that already has one, which is
a migration rather than a label.

## The kernel bug it found

The hardest thing this project produced was not in this project.

Building and removing libraries in a loop, the host began refusing to create new ones.
`rmmod mhvtl` said the module was in use with everything stopped. Only a reboot cleared it.

It was not the console and not MHVTL. The Linux `ch` driver — the one that makes `/dev/sch0`
for a medium changer — looks up the `scsi_device` of every drive the changer reports, takes a
reference to each, and never drops it. `ch_destroy()` frees the array with `kfree()`, which
drops nothing. Remove the library and those references outlive it, pinning SCSI addresses
until reboot.

It had been reported once before, with an RFC patch in July 2022 that got no replies and was
never merged.

The fix was nine lines added and twenty removed. It was reviewed in five hours by Laurence
Oberman at Red Hat, needed a **v2** because the `Fixes:` tag was wrong — `1da177e4c3f4`, the
initial git import, when `ch.c` was actually added a month later by `daa6eda65a53` — and was
**applied to the SCSI tree on 5 October 2026** as `27c0f718d3b3`, with `Cc: stable` so it
reaches the stable kernels.

Two things that cost time and are worth knowing:

**A reply can add a trailer; it cannot replace one.** Correcting the tag by replying to the
thread left both tags in it, and nothing says which supersedes which. The fix for a bad
trailer is a new version, not a reply.

**The `Fixes:` tag is read by machines.** Stable maintainers use it to decide how far back to
backport. It is the one line in a patch where a plausible guess is worse than no guess.

Until the backport reaches a given distribution the console detects the condition, says so on
the page, and the recommended host configuration blacklists `ch`. The console never needed
it: every changer lookup goes through `sg`, which answers the same questions.

## How big is a cartridge? The rule, and its limits

Everything above argues for one rule: every decision in one place, both front ends asking.
This section is the project's own answer to the obvious objection — *and then what?* — on
the simplest question it has:

**How big is a new cartridge?**

For two days in October the answer was "whatever the hardware holds". Create an LTO-8 and
you got a 12 TB cartridge. That is correct about the hardware and wrong about this
software. MHVTL's media files are **sparse**, so a 12 TB tape and a 1 GB tape both start
at a few kilobytes and the size costs no disk at all. It costs time. At the 55 MB/s this
host writes, filling a native LTO-8 takes about **61 hours**:

| Cartridge | Time to fill at 55 MB/s |
| --- | --- |
| 1,000 MB — the default | 18 seconds |
| LTO-6 at its real 2.5 TB | 13 hours |
| LTO-8 at its real 12 TB | **61 hours** |
| LTO-10 at its real 30 TB | 152 hours — six days |

And the things a virtual tape library exists to exercise are all at the *end* of a tape:
does the backup span to the next cartridge, does it ask for one, does the drive report end
of medium, does the fullness bar move. On a cartridge nobody can fill, none of it is
reachable. The default made the product's main purpose untestable.

So the default is now 1,000 MB, and anyone who wants a realistic cartridge says so. That
part is a one-line change. The interesting part is that this is the **second** fix to the
same question, and the first one did everything this article argues for.

### The first fix obeyed the rule and was still wrong

Until 4 October the question had no owner at all. Five files each held a number, and they
were not the same number:

```
services/tapes/service.py        DEFAULT_SIZE_MB = 500
mhvtl_cli/commands/tape.py       --size-mb default=500000     (twice)
tape_operations_views.py         '500000'                     (three times)
create_tape.html, _bulk.html     value="500000"               (twice)
static/js/tape-media.js          UNKNOWN_SIZE_MB = 1000
```

A library's own cartridges were 500 MB and any cartridge added to it afterwards was
500 GB — a thousand times larger, in the same library, depending on which path made it.
`mhvtl tape list` printed them side by side, which is how it was found.

The fix was the move this article recommends everywhere else: delete all five numbers,
give the question one owner in the service layer, and let both front ends ask. Neither of
those numbers had ever been a cartridge anyway, while the real capacities were sitting in
the profile catalogue, transcribed from MHVTL — so the owner answered with the density's
**native capacity**. One implementation, one source, hardware facts from a source with the
source named in the code. By every rule in the list above, that was the right fix.

It is also what made a new LTO-8 twelve terabytes, and nobody could fill one.

**So the rule is necessary and not sufficient.** Putting a decision in one place says
nothing about whether the decision is any good; it only means there is now exactly one
thing to change when it isn't. The second fix changed it, and kept the single owner:
`native_mb(density)` and `size_for(density)` became two functions, because they are two
questions — *what does this cartridge hold*, which is about hardware, and *how big will
one be made here*, which is about this host. One function answering both is what made the
right answer to the first question the wrong answer to the second.

One chain, four levels, each narrower than the last, and **no fifth**:

| | Where | What it covers |
| --- | --- | --- |
| 1 | the code | 1,000 MB, every generation |
| 2 | `tape.size.default` | every generation, on this host |
| 3 | `tape.size.LTO8` | one generation |
| 4 | `--size-mb`, or the form | one cartridge, once |

Levels 2 and 3 are a new file, `/etc/mhvtl-gui/settings.toml` — the console's own
preferences, beside `presets.toml`, which is the other file that holds the operator's
choices rather than MHVTL's configuration. The native capacities stay where they were, in
the profile catalogue transcribed from MHVTL's source: that table holds facts, these files
hold choices, and an operator editing one never changes what an LTO-8 is.

```console
$ mhvtl settings list tape.size.lto
SETTING           VALUE            FROM                 THE CARTRIDGE HOLDS
----------------  ---------------  -------------------  ---------------------
tape.size.LTO7    1 GB (1,000 MB)  the shipped default  6 TB (6,000,000 MB)
tape.size.LTO8    1 GB (1,000 MB)  the shipped default  12 TB (12,000,000 MB)
tape.size.LTO9    1 GB (1,000 MB)  the shipped default  18 TB (18,000,000 MB)

11 setting(s), all at the shipped default

$ sudo mhvtl settings set tape.size.LTO8 12TB
tape.size.LTO8 is now 12 TB (12,000,000 MB)
Tapes created from now on use this; the ones you already have keep the size they were made with.
```

The **FROM** column is the point. *Why is this cartridge 1 GB* is the question an operator
actually asks, and a number with no provenance does not answer it. So the service returns
where the value came from, and both front ends print it — a column in the terminal, a
column on the page.

### What went wrong anyway, and what each one teaches

The feature was planned, the code was read before it was changed, and it still went wrong
four times: once in the design, and three times in ways that shipped. Each is worth more
than its fix.

**The design put the size on the wrong noun.** It gave a library one size. But a library
holding LTO-8 and DLT-4 holds two capacities, and one number gives the second kind the
first kind's. The size belongs to the *media run* — the row that says "twenty LTO-8
cartridges" — not to the library. So it is a field on each row of the wizard,
`--media-size LTO8:12TB` repeated on the command line, and `size_mb` on a preset's run.
This one was caught before it shipped, during the implementation, by the first library
that held two generations. *A value attached to the wrong noun survives review, because
the noun is only wrong when it occurs twice.*

**The drop-downs went on naming the old answer.** The creation forms label each cartridge
`LTO8 (12 TB)`, built from the capacity table. That was correct while a cartridge was made
at its native capacity — the option named what you would get. The moment the two became
separate questions, the option said 12 TB about a tape that would be made at 1 GB: wrong
by a factor of twelve thousand, in the one place an operator is choosing. The label now
comes from the same call the create makes, so it cannot name a size the create will not
produce. *When one question becomes two, every label that answered the old one is now
answering the wrong one — and labels are not where anybody looks for a bug.*

**`12TB` meant three different things.** `settings set` took `12TB`. `--media-size
LTO8:12TB` took `12TB`. `--size-mb` took a bare integer, because it was `type=int` from
before any of this existed:

```console
$ sudo mhvtl tape create 50 K50001L8 --size-mb 12TB
mhvtl tape create: error: argument --size-mb: invalid int value: '12TB'
```

The comment beside `--media-size` said all three parsed the same way. It was wrong about
the flag six lines above it. All four now go through one parser, and refuse `MiB`/`GiB`
with the sentence that explains why — 12 TB and 12 TiB are ten per cent apart, and a
cartridge ten per cent off the one you meant is a difference you find out about later.
*The rule was never "one service decides"; it was "one implementation". A second parser is
a second implementation even when it is four characters long.*

**The provenance column overstated itself.** The file's default is written on every save
whether or not anybody set it, so that changing the shipped default in a later release
cannot silently resize a host that already has a file. Good reason, bad consequence:
change one density and all thirty-three settings reported as changed, each claiming the
file as its source. The pin stays; reporting it as a decision nobody made does not. *A
guard that protects the data can still lie about it, and the lie is in the one column the
feature was built to fill.*

The last two were found by checking the user-guide page against real output rather than by
a test — three commands it documented could not run. That is the same lesson as the
tutorial, in a smaller place: documentation that is not executed is already wrong, and you
have not found out where.

## What changed since 2.1.1

For a reader who knows the earlier version, in one table:

| | 2.1.1, September | 3.4.0, October |
| --- | --- | --- |
| Tests | 1,225 | 2,126 |
| `mhvtl` command nouns | 10 | 14 — `profile`, `preset`, `ltfs` and `settings` are new |
| Newest cartridge | LTO8 | LTO10 |
| Page frames | 6 | 1 |
| Hardware vocabulary | "profiles", loosely | profile and preset, separated |
| Drives per library | one kind | two generations at once |
| A new cartridge's size | its native capacity — 12 TB for an LTO-8 | 1,000 MB, and a setting per generation |
| Where a capacity is decided | five places, holding three different numbers | one chain of four levels, with no fifth |
| What `mhvtl --version` printed | `mhvtl-gui 2.1.1` | `mhvtl-console 3.4.0`, package still `mhvtl-gui` |
| Upstream | nine MHVTL patches sent | four merged, plus a kernel commit |

The one that matters least on paper and most in practice is the page frame. Fifty pages
behind one header is the difference between a console and a collection of pages that happen
to share a database.

## Where to find it

- Source, issues and releases: `github.com/abdelhaleemahmed/mhvtl-console`
- User Guide and API reference: `abdelhaleemahmed.github.io/mhvtl-console`
- MHVTL itself, by Mark Harvey: `github.com/markh794/mhvtl`
- The kernel fix: `git.kernel.org/mkp/scsi/c/27c0f718d3b3`
