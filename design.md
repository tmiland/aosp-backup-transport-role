# AOSP proposal: user-grantable backup transport role

Status: draft · 2026-10-29
Target: AOSP (Android 17 / mainline Permission module)
Author: prepared for filing on issuetracker / Gerrit

---

## 1. Summary

Android only allows **system-whitelisted, privileged** apps to act as the backup
transport. A user cannot install a backup provider. This proposal adds a
user-grantable role, `android.app.role.BACKUP_TRANSPORT`, so an app such as
Seedvault can be installed on any Android device, selected by the user with an
explicit consent screen, and used as the system backup transport.

The mechanism already exists: role-granted permissions
(`PermissionInfo.PROTECTION_FLAG_ROLE`, `0x4000000`) are used today for the
dialer's voicemail permissions and for
`EXEMPT_FROM_AUDIO_RECORD_RESTRICTIONS`.

---

## 2. Current enforcement

- `android.permission.BACKUP` is `signature|privileged`
  (`frameworks/base/core/res/AndroidManifest.xml`, protection level at the
  `BACKUP` declaration).
- `TransportManager.isTransportTrusted()` requires **both**:
  - the component is in `SystemConfig.getBackupTransportWhitelist()`
    (`BackupManagerService` reads it at startup), and
  - the package has `ApplicationInfo.PRIVATE_FLAG_PRIVILEGED`.
- Seedvault's manifest requests `android.permission.BACKUP` and declares its
  `android.backup.TRANSPORT_HOST` service `android:exported="false"`; it is
  designed to be compiled into the OS.
- `RoleManager` has roles for assistant, browser, dialer, SMS, home, call
  screening, notes, wallet, etc. — **none for backup**.

Result: stock ROMs can only use the vendor/Google transport. Users who want an
open, portable backup must flash a ROM that bundles one (GrapheneOS, CalyxOS,
LineageOS all ship Seedvault as a privileged app). This is also the practical
blocker for cross-OS device migration: the source phone cannot hand app data to
another OS without a privileged transport.

---

## 3. Proposal

### 3.1 New role

`RoleManager.ROLE_BACKUP_TRANSPORT = "android.app.role.BACKUP_TRANSPORT"`

- Availability: all devices, per user (backups are per-user).
- Holder qualification: the app declares a service with
  `android.backup.TRANSPORT_HOST` and implements `BackupTransportService`.
- Default holder: the preinstalled transport (if any), so existing behaviour is
  unchanged out of the box.

### 3.2 Permission

Change `android.permission.BACKUP`'s protection level from
`signature|privileged` to `signature|privileged|role`.

Role grant → `Permissions.grantAsUser(BACKUP)`; role loss →
`Permissions.revokeAsUser(BACKUP)` (both helpers already exist and are used by
`DialerRoleBehavior`).

### 3.3 Transport trust

`TransportManager.isTransportTrusted()` accepts either:

1. the current rule (whitelist + privileged), or
2. the component is the current `ROLE_BACKUP_TRANSPORT` holder and holds
   `BACKUP`.

The whitelist path stays for OEM transports; role holders do not need to be
added to `config_backupTransportWhitelist`.

### 3.4 UI

- Settings → System → Backup becomes a role picker: current transport, change
  transport, transport settings.
- Consent screen before granting:
  - app name, publisher/signature, install source;
  - destination (local, USB, network, cloud);
  - explicit warning: *"This app will be able to read and store the data of
    apps that allow backup, including SMS, call log and contacts where the
    system includes them."*
- Revoking the role unbinds the transport, disables its scheduling, and revokes
  `BACKUP`. Stored backups are left where they are; the app loses access to new
  backups.

### 3.5 D2D flag restriction (v1)

Role-based transports must **not** advertise
`FLAG_DEVICE_TO_DEVICE_TRANSFER`. That flag makes the system ignore
`android:allowBackup="false"` for non-system apps, which is appropriate for a
one-time OEM migration but too broad for a permanent third-party transport.
If device migration from a role transport is wanted later, it needs a separate
user opt-in and a distinct role/flag, not the backup role.

---

## 4. Security considerations

- `BACKUP` is a high-trust permission: the transport receives app data. The
  protection is user consent, not signature checks. This is the same trust model
  as accessibility services, VPN apps and the SMS role.
- The transport only sees data the system hands it through `BackupTransport`;
  it does not gain direct filesystem access to app sandboxes.
- Role revocation must be immediate and complete (permission revoked, binding
  dropped, scheduling cancelled).
- Work profile / secondary users: role is per user; the transport runs in the
  same user.
- OEMs remain free to preinstall a transport and keep the role granted by
  default; the change is additive.

---

## 5. Compatibility

- Existing whitelisted privileged transports: unchanged.
- Devices without a preinstalled transport: no behaviour change until the user
  grants the role.
- CTS: add role tests; no existing test should regress.

---

## 6. Test plan

- CTS (role): grant/revoke `ROLE_BACKUP_TRANSPORT`; `BACKUP` granted only while
  holder; non-qualifying app cannot hold the role.
- CTS (backup): a non-role app declaring `TRANSPORT_HOST` is rejected
  (`ERROR_TRANSPORT_INVALID`); the role holder registers successfully; a
  previously whitelisted transport still registers.
- Backup/restore round trip with a CTS app that has `allowBackup=true`, and a
  second app with `allowBackup=false` (must be excluded, because role
  transports cannot set the D2D flag).
- Revocation: no further backups; transport unbound; `BACKUP` no longer granted.
- Settings UI: picker, warning text, revocation flow.

---

## 7. Alternatives

- **Open the D2D protocol** (Google/OHA): enables live migration but is a much
  larger, proprietary-protocol problem.
- **Every OEM bundles Seedvault**: works today where it happens, but requires
  thousands of ROMs and does not help existing devices.
- **Status quo**: users on stock ROMs stay locked to the vendor transport.

---

## 8. Patch outline

| File | Change |
|---|---|
| `frameworks/base/core/res/AndroidManifest.xml` | `BACKUP` → `signature\|privileged\|role` |
| `frameworks/base/services/backup/.../TransportManager.java` | accept role holder in `isTransportTrusted()` |
| `packages/modules/Permission/framework-s/java/android/app/role/RoleManager.java` | `ROLE_BACKUP_TRANSPORT` constant + docs |
| `packages/modules/Permission/PermissionController/role-controller/.../behavior/BackupTransportRoleBehavior.java` | qualification, grant/revoke `BACKUP`, default holder |
| `packages/apps/Settings` (backup screen) | role picker + consent warning |
| `cts/tests/...` | role + transport registration tests |

---

## 9. Open questions

- Should the role be available on all form factors (TV, Automotive, Wear)?
  Backups exist there too; start with handhelds and expand.
- Should the default holder be pre-granted on devices that already ship a
  transport? Recommended yes, to preserve current behaviour.
- Should Play policy require disclosure of the transport's data destination?
  Out of AOSP's scope, but worth coordinating.
