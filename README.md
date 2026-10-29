# User-grantable backup transport role — AOSP proposal

Android restricts the system backup transport to apps that are **both** on the
system transport whitelist and privileged (`TransportManager.isTransportTrusted`:
`SystemConfig.getBackupTransportWhitelist()` + `PRIVATE_FLAG_PRIVILEGED`). A user
cannot install a backup provider such as Seedvault, and stock phones cannot hand
app data to another OS.

This repository proposes a user-grantable role,
`android.app.role.BACKUP_TRANSPORT`, built on the existing
`PermissionInfo.PROTECTION_FLAG_ROLE` mechanism (already used for the dialer's
voicemail permissions and `EXEMPT_FROM_AUDIO_RECORD_RESTRICTIONS`).

## Files

- [design.md](design.md) — the proposal: role semantics, permission change,
  transport trust, consent UX, security model, test plan, patch outline.
- [issue.md](issue.md) — the text intended for the AOSP issue tracker.
- [patch/](patch/) — reference patch (four commits), compile-checked on
  `android-latest-release` (`android17-release`).

## Status

Draft, not yet filed. The reference patch compiles; the AOSP issue is not filed
yet. See [patch/README.md](patch/README.md) for the base, apply order and known
RFC caveats.

## Why

- Stock ROM users are locked to the vendor/Google transport and its cloud.
- Open backup formats (Seedvault) work only where the app is compiled into the
  system image (GrapheneOS, CalyxOS, LineageOS).
- Cross-OS device migration is blocked by the same rule: a stock phone cannot
  hand app data to another OS without a privileged transport on the source.

## Security

The role is consent-based, like accessibility services, VPN apps or the SMS
role. The transport only receives data through the `BackupTransport` interface;
it does not get sandbox access. Role-based transports must not advertise
`FLAG_DEVICE_TO_DEVICE_TRANSFER`, so `android:allowBackup="false"` remains
respected.

## License

Text: CC BY 4.0. Code (patch/): Apache-2.0.
