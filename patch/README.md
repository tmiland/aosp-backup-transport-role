# Reference patch (RFC)

Four commits, split across two AOSP projects.

## Base

| Project | Commit | Date |
|---|---|---|
| manifest (`android-latest-release` → `android17-release`) | `ad156f32` | 2026-06-15 |
| `frameworks/base` | `94b4c163b7df` | 2026-05-21 |
| `packages/modules/Permission` | `1093eca74075` | 2026-05-15 |

## Apply

```bash
# frameworks/base
git am patch/frameworks-base/0001-*.patch patch/frameworks-base/0002-*.patch

# packages/modules/Permission
git am patch/packages-modules-permission/0001-*.patch \
       patch/packages-modules-permission/0002-*.patch
```

## Compile check

```bash
source build/envsetup.sh
lunch aosp_arm64-trunk_staging-userdebug
m framework-minus-apex PermissionController
```

Verified on the base above: `build completed successfully`.

## What the series does

1. `backup: add role flag to BACKUP permission` — `BACKUP` becomes
   `signature|privileged|role`.
2. `backup: accept the backup transport role holder` — `TransportManager`
   trusts the current `ROLE_BACKUP_TRANSPORT` holder in addition to the
   whitelist/privileged path.
3. `role: add ROLE_BACKUP_TRANSPORT` — new role constant plus `current.txt`.
4. `role: define the backup transport role` — role definition, required
   `android.backup.TRANSPORT_HOST` service, `BACKUP` grant on role grant,
   consent strings and behavior class.

## Known caveats (RFC)

- The role constant is annotated `@SuppressLint("UnflaggedApi")`; upstream
  requires a new aconfig flag before merge.
- `framework-s/api/current.txt` is updated; API approval is part of review.
- Open question: whether a role-based transport may advertise
  `FLAG_DEVICE_TO_DEVICE_TRANSFER`. It is needed to restore a device-to-device
  backup that contains apps with `allowBackup="false"`, but it is a broad
  capability for a permanent transport. See `design.md`.
- Settings UI is expected to pick the role up through PermissionController's
  dynamic role list ("Default apps"); not separately tested here.
