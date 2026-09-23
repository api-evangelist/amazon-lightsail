---
name: amazon-lightsail-back-up-before-you-destroy
description: >-
  Make an Amazon Lightsail resource recoverable before you change or delete it — manual snapshots that
  outlive the resource, automatic snapshots that do not, and the seven-day windows that decide whether
  an undo exists.
api: Amazon Lightsail
apiId: amazon-lightsail:amazon-lightsail-instances-api
generated: '2026-09-17'
method: generated
source: >-
  openapi/amazon-lightsail-openapi.yml and
  https://docs.aws.amazon.com/lightsail/latest/userguide/amazon-lightsail-configuring-automatic-snapshots.html
operations:
  - EnableAddOn
  - DisableAddOn
  - GetAutoSnapshots
  - CreateInstanceSnapshot
  - GetInstanceSnapshots
  - CreateInstancesFromSnapshot
  - CreateDiskSnapshot
  - CreateDiskFromSnapshot
  - CopySnapshot
  - CreateRelationalDatabaseSnapshot
  - CreateRelationalDatabaseFromSnapshot
  - DeleteInstance
  - GetOperationsForResource
---

# Back up before you destroy

The rule that decides whether a Lightsail mistake is recoverable is one sentence from AWS's own
documentation: **all automatic snapshots associated with a resource are deleted when you delete the
source resource, and manual snapshots are not.** Anything that relies on automatic snapshots as an undo
is relying on a backup that disappears at exactly the moment you need it.

## What exists, and for how long

| Mechanism | Covers | Window | Survives deleting the source? |
|---|---|---|---|
| Automatic snapshots (`EnableAddOn`) | Linux/Unix instances and disks | Latest **7** daily snapshots; oldest is replaced | **No** |
| Manual snapshots (`CreateInstanceSnapshot`, `CreateDiskSnapshot`) | Instances, disks | No expiry | **Yes** |
| Database point-in-time backup | Managed databases | 5-minute increments over the previous **7 days**, restored to a *new* database | n/a — restore before deleting |

Automatic snapshots are not available for Windows instances or managed databases.

## Steps

1. **Turn on automatic snapshots** for anything long-lived: `EnableAddOn` with `addOnType`
   `AutoSnapshot` and a `snapshotTimeOfDay`. Verify with `GetAutoSnapshots`. `DisableAddOn` turns it off.
2. **Before any destructive change, take a manual snapshot.** `CreateInstanceSnapshot` (or
   `CreateDiskSnapshot`, or `CreateRelationalDatabaseSnapshot`). Poll the returned operation to a
   terminal state before you touch the source — a snapshot still in progress is not a backup.
3. **To keep an automatic snapshot permanently**, promote it: `CopySnapshot` with
   `sourceResourceName` and `restoreDate` turns an automatic snapshot into a manual one.
4. **Restore** with `CreateInstancesFromSnapshot`, `CreateDiskFromSnapshot`, or
   `CreateRelationalDatabaseFromSnapshot`. All three create a **new** resource. Nothing in this API
   restores in place, so plan for a name change and a DNS or static-IP move.
5. **Then delete.** `DeleteInstance`. Watch it with `GetOperationsForResource`.

## Agent guidance

- Treat every `Delete*` operation as **destructive, high consequence, confirm before calling**. The
  overlay in `overlays/amazon-lightsail-overlay.yaml` marks all 22 of them.
- `ReleaseStaticIp` deserves the same caution: the address is gone and you cannot re-reserve the same
  one.
- There is no dry-run mode anywhere on this API. `GetCostEstimate` will price a resource without
  creating it, which is the closest thing to a rehearsal available.
