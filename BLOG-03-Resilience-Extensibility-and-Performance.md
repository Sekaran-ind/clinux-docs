# Local-First Clinical Software: Offline, Sync and Conflict Without Losing Data

*Building Clinical Software That Deserves Trust, part 3*

**For:** engineers and architects building clinical apps for places where the network is a
convenience, not a guarantee.

---

A clinic in a tier-3 town loses power twice before lunch. The inverter keeps the router alive,
but the broadband line doesn't come back until four. A cloud-only clinical app stops registering
patients at 10:40 and resumes at 4:05, and in between someone writes on paper and someone else
retypes it that evening. That retyping is where errors come from.

ClinuxFlow is local-first: every record is written to the device before anything else, and
network sync is added on top. This post covers what that takes in practice: storage limits,
tenancy on shared machines, the sync strategies that are and aren't safe for clinical data, and
the bugs we hit. It also replaces an earlier draft of this post that promised a "self-evolving"
system that tuned itself with reinforcement learning. We didn't build that, and the problems
below matter more.

## What local-first means

The phrase comes from Kleppmann and colleagues' 2019 essay "Local-first software", which lists
properties like fast (no round-trip to save), works offline, and the user retains ownership. For a
clinic, three matter most:

1. **Saving never waits for the network.** The save completes on the device; sync is a separate,
   later concern.
2. **The device is the source of truth between syncs.** A server copy is a mirror, not the master.
3. **Network features degrade, they don't fail.** When the cloud is unreachable or not paid for,
   the app keeps working and says so.

ClinuxFlow has three storage tiers, added in layers:

| Tier | Where | Who gets it |
|---|---|---|
| Device | Browser storage (localStorage, IndexedDB) or the mobile app's equivalent | Everyone |
| LAN | A small server on the clinic's own network (our desktop app runs one), which devices mirror to | Every clinic that runs it, free |
| Cloud | Our API and database | Paid clinics |

Each layer wraps the same collection interface, so a screen never knows which tier answered it.
A save to the cloud happens after the local save succeeds. If the cloud refuses (for example
because the clinic is on the free tier), the app treats that as "stay local", not as an error.
We learned this the hard way: one of our screens treated a free-tier refusal from a paid-only
endpoint as a failure, and free clinics could not open Checkout at all.

## Browser storage is smaller and less permanent than you think

- **localStorage** is synchronous and limited to a few megabytes per origin. It's fine for
  configuration and small records and wrong for anything that grows. An append-only audit log or a
  batch of synthetic test records will hit the limit.
- **IndexedDB** holds far more but is asynchronous, and the browser may evict it under storage
  pressure. Ask for persistence (`navigator.storage.persist()`) and check the answer. Some
  browsers, notably Safari, may clear script-written storage for sites that haven't been used for
  a while. A packaged mobile or desktop app is more predictable than a browser tab.
- **Neither is a backup.** A clinic whose only copy of a record is in a browser profile on one
  laptop has one hard-disk failure between itself and data loss. The LAN or cloud tier is the
  backup story, and the app should tell the user plainly when a record exists only on this device.

We moved our workflow snapshots and audit log to IndexedDB because the audit log is append-only
and grows forever. Two bugs surfaced immediately, both caught by tests: a key-value library only
created its object store the first time a database was opened, so a second collection sharing
that database had nowhere to write; and `crypto.randomUUID()` doesn't exist on pages served over
plain HTTP on a LAN IP, which is exactly how clinic devices reach a local server.

## Shared devices and tenancy

Front-desk computers are shared. A receptionist who works mornings at one clinic and evenings at
another may use the same browser for both. Our early collections weren't scoped by clinic, so
records from one clinic appeared in the other's lists on a shared browser. Every local record now
carries its clinic id, and every read filters by it. Treat the device as multi-tenant from day
one; it is cheaper than finding out from a user.

## Choosing a sync strategy for clinical data

This is the heart of the problem. There are four broad options.

**Last write wins.** Whichever copy was saved most recently replaces the other. It's simple and
almost every system starts here. For clinical data it's dangerous: a nurse's vitals and a
doctor's allergy update, made on different devices a minute apart, cannot both survive a
whole-record overwrite.

**Automatic merging (CRDTs and similar).** Conflict-free replicated data types merge concurrent
edits deterministically. They're excellent for collaborative text. For clinical facts they can be
quietly wrong: merge two concurrently edited medication lists and you may get a regimen nobody
prescribed. A merge that is mathematically consistent and clinically meaningless is worse than a
visible conflict.

**Single writer with leases.** One device at a time holds a time-limited lock on a unit of work,
renews it while working, and releases it when done. Others can read but not write. This is what
ClinuxFlow uses for encounter stages (only one device works a patient's consultation at a time)
and for running workflows. The lease expires if the device crashes, so a dead tablet can't hold a
patient hostage. It's boring and it's safe.

**Detect and ask a human.** When two copies have both changed since their last common version, stop
and show both, field by field, and let a person choose, recording the value that lost. This is the
right answer for the cases a lock can't prevent (two devices offline at once), and it's what our
design calls for. It isn't built yet; today, outside locked work, our LAN sync is poll-and-merge
with a short protection window after a local save, which avoids the race we saw in practice but
still ends in last-write-wins.

Our rule is: **lock where you can, ask a human where you can't, and never auto-merge clinical
facts.**

## Change the software without stranding the data

Local-first means data lives on devices you don't control, running whatever version the user last
loaded. Two lessons:

**Version anything you persist that has a shape.** We persist the state of running workflows so a
user can resume after a reload. When we added six steps to a workflow under the same id, every
device with a saved state from the old version restored into a machine that no longer matched it,
and every step appeared locked. The fix is general: before restoring, compare the saved state's
shape with the current definition, and start fresh if they differ. The clinical data itself lives
separately and is untouched; only the progress marker resets.

**Keep the audit log append-only.** Our workflow audit is a list of transitions that is only ever
appended to, locally and in the cloud mirror. Even if a lock is bypassed and two devices write,
the log lets you reconstruct what happened in which order. Snapshots answer "where are we"; logs
answer "how did we get here", and clinical work needs both.

## Races you'll meet

Every one of these happened to us:
- **Two instances of the same form** on one page (a hidden modal and a visible panel) both
  reacting to one successful save; the hidden one navigated the user away mid-task. Each instance
  must react only to actions it started.
- **Subscribing after the start**: a listener attached after a state machine started never saw
  its initial state, so statuses read as undefined until the next change.
- **A resumed page racing its own restore**: reads that ran before the stored state finished
  loading. The fix is to make every route that needs data wait for hydration, not only the first
  page.

None of these show up in a demo. All of them show up in the first week at a real front desk.

## What to measure

The earlier draft of this post promised learning loops. What a clinic operator actually needs:

| Metric | Why it matters |
|---|---|
| Age of the oldest unsynced record, per device | Tells you who is one disk failure from loss |
| Sync lag (save to mirror), p50 and p95 | Whether staff on other devices see current data |
| Lock contention and expiry counts | Whether the workflow design matches how staff actually work |
| Conflicts detected and how they were resolved | Whether devices are routinely editing the same records |
| Storage headroom per device | The next quota failure, before it happens |
| Time to complete registration and consultation | Whether the software helps or slows the visit |

## Where ClinuxFlow is today

- Built: device, LAN and cloud tiers behind one collection interface; clinic-scoped local data;
  IndexedDB for the growing collections; leased single-writer locks for encounter stages and
  workflows; an append-only audit log mirrored to the cloud; workflow-state version checks.
- Not built: field-level conflict detection and the human resolution screen; a persisted per-record
  "not yet synced" marker; peer-to-peer data sync between devices without a server; encryption of
  on-device data at rest; the metrics above as a dashboard.

The next post turns to security and privacy, where local-first raises questions of its own.
