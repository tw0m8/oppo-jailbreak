# Third-party notices

This repository redistributes binaries built by other projects. Their licences apply to those
components, not the MIT licence that covers this project's own code and documentation.

## KernelSU-Next — `dist/*.apk` → `assets/ksud`

- Upstream: https://github.com/KernelSU-Next/KernelSU-Next
- Licence: **GPL-3.0**
- What it is: the `ksud` daemon bundled in the APK, used to late-load the KernelSU kernel module
  (`ksud late-load --kmi android15-6.6 --allow-shell`). Redistributed unmodified.
- Source: available from the upstream repository at the corresponding release. If you need a
  written offer of source alongside a binary redistribution, open an issue.

## Shizuku — linked into the app (`app/libs/{aidl,api,provider,shared}-13.1.5.aar`)

- Upstream: https://github.com/RikkaApps/Shizuku-API
- Licence: **Apache-2.0**
- What it is: the client library that starts a user service as the shell user, which is how the
  payload is executed without root.

## KernelSU / KernelSU-Next manager (not bundled)

Shown in screenshots only. Not distributed by this repository; install it from its own upstream.

## This project's own components

- The `timer` payload and the Android app in `dist/` are this project's work, covered by the MIT
  licence in [LICENSE](LICENSE). The payload source is not published here.
- The exploit constants are derived from the device's own kernel image, which is the property of
  its vendor and is not redistributed.
