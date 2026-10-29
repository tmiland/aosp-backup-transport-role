# Issue tracker draft — user-grantable backup transport role

Component suggestion: Platform → Frameworks → Backup (closest match)
Type: Feature request
Title: User-grantable backup transport role so third-party backup apps can run on stock Android

---

## Summary

Android currently restricts the system backup transport to apps that are both
listed in the system transport whitelist and privileged
(`TransportManager.isTransportTrusted()`: `SystemConfig.getBackupTransportWhitelist()`
plus `ApplicationInfo.PRIVATE_FLAG_PRIVILEGED`). A user cannot install and select
a backup provider such as Seedvault.

Proposal: add a user-grantable role, `android.app.role.BACKUP_TRANSPORT`, and
grant `android.permission.BACKUP` to its holder through the existing
`PROTECTION_FLAG_ROLE` mechanism (already used for the dialer's voicemail
permissions and `EXEMPT_FROM_AUDIO_RECORD_RESTRICTIONS`).

## Why this matters

- Users on stock ROMs are locked to the vendor/Google backup transport, including
  its cloud storage and account requirements.
- Portable, open backup formats (Seedvault) work only on ROMs that compile the
  app into the system image (GrapheneOS, CalyxOS, LineageOS).
- It is also the practical blocker for device migration across OSes: a stock
  phone cannot hand app data to another OS without a privileged transport on the
  source device.
- `adb backup` is deprecated and excludes apps that set
  `android:allowBackup="false"`, so it is not a substitute.

## Proposal

1. Define `RoleManager.ROLE_BACKUP_TRANSPORT` ("Backup app"), per user, available
   on handhelds.
2. Qualify apps that declare a `android.backup.TRANSPORT_HOST` service.
3. Change `android.permission.BACKUP` protection level from
   `signature|privileged` to `signature|privileged|role`; grant it on role
   grant and revoke it on role loss (`Permissions.grantAsUser` /
   `revokeAsUser`, as `DialerRoleBehavior` already does).
4. In `TransportManager.isTransportTrusted()`, also accept the current role
   holder (whitelist path unchanged).
5. Settings → System → Backup: role picker plus a consent screen listing the
   app, publisher/signature, destination and the fact that it can read app data.
   Revocation unbinds the transport and cancels its scheduling.
6. Restrict role-based transports from advertising
   `FLAG_DEVICE_TO_DEVICE_TRANSFER` (that flag bypasses
   `android:allowBackup="false"`; too broad for a permanent transport). Device
   migration, if wanted later, needs a separate opt-in.

## Security considerations

- This is a consent-based trust model, like accessibility services, VPN apps and
  the SMS role. The transport only receives data through the
  `BackupTransport` interface; it does not get direct sandbox access.
- The role and the permission must be revoked atomically when the user changes
  the transport.
- Existing privileged/whitelisted transports keep working; the change is
  additive.

## Alternatives

- Publish the device-to-device transfer protocol so other OSes can implement the
  receiving side (larger, different problem).
- Require every OEM to bundle Seedvault (works where it happens, not a platform
  solution).

## Requested outcome

Let a user install a backup app and select it as the system transport after an
explicit warning, without requiring it to be a privileged system app.

## Attachments

- Design note: role semantics, consent UI, security mitigations, test plan
  (available on request / to be linked once published).
- Reference patch outline: RoleManager constant, BackupTransportRoleBehavior,
  BACKUP protection level, TransportManager role check, Settings picker, CTS.
- Prior art: Seedvault (github.com/seedvault-app/seedvault) — "compiled with the
  operating system"; shipped as a privileged app by GrapheneOS, CalyxOS and
  LineageOS.
