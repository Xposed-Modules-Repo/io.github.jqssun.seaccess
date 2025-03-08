# Secure Element Access

[![Stars](https://img.shields.io/github/stars/jqssun/android-se-access)](https://github.com/jqssun/android-se-access/stargazers)
[![LSPosed](https://img.shields.io/github/downloads/Xposed-Modules-Repo/io.github.jqssun.seaccess/total?label=LSPosed&logo=Android&style=flat&labelColor=F48FB1&logoColor=ffffff)](https://github.com/Xposed-Modules-Repo/io.github.jqssun.seaccess/releases)
[![GitHub](https://img.shields.io/github/downloads/jqssun/android-se-access/total?label=GitHub&logo=GitHub)](https://github.com/jqssun/android-se-access/releases)
[![release](https://img.shields.io/github/v/release/jqssun/android-se-access)](https://github.com/jqssun/android-se-access/releases)
[![build](https://img.shields.io/github/actions/workflow/status/jqssun/android-se-access/apk.yml)](https://github.com/jqssun/android-se-access/actions/workflows/apk.yml)
[![license](https://img.shields.io/github/license/jqssun/android-se-access)](https://github.com/jqssun/android-se-access/blob/master/LICENSE)
[![issues](https://img.shields.io/github/issues/jqssun/android-se-access)](https://github.com/jqssun/android-se-access/issues)
  
A module to explicitly give trusted apps access to the secure element (eSE) by using a safer application ID based implementation, regardless of the ARF configuration on the eUICC.

## Compatibility

- Android 10+
- Rooted devices with Xposed framework installed (e.g. LSPosed)
- Access can be granted to any of the following Local Profile Assistant (LPA) apps:
  - (OEM) [com.google.android.euicc](https://play.google.com/store/apps/details?id=com.google.android.euicc)
  - (FOSS) [chat.jmp.simmanager](https://f-droid.org/en/packages/chat.jmp.simmanager/)
  - (FOSS) [im.angry.easyeuicc](https://gitea.angry.im/PeterCxy/OpenEUICC)

## Implementation

On Android, eUICC APIs are by default restricted to whitelisted apps. This makes sense because most eUICC chips are integrated to the device. If someone wants to access the chip, say for instance to provision a profile, they generally need to do so through a [Local Profile Assistant (LPA) app](https://source.android.com/docs/core/connect/esim-overview#making_an_lpa_app), which can either be
- [a system app that provides EuiccService](https://source.android.com/docs/core/connect/esim-overview#making_an_lpa_app) (in the case of an integrated eUICC), or 
- [a carrier app signed by a certificate](https://source.android.com/docs/core/connect/uicc#arf) whose hashes match the [ARF/ACCF stored in the eUICC](https://source.android.com/docs/core/connect/uicc#validation)

In the case of a removable eSIM (such as those from [sysmocom](https://sysmocom.de/products/sim/sysmocom-euicc/), [estk.me](https://www.estk.me/) or [esim.me](https://esim.me/)), the options are limited as they do not have control over device firmware. Users also have to rely heavily on carrier support for their apps as the ARF is baked into the eUICC itself.

However, there is a caveat. Probably to faciliate Android vendor testing, full access to the eUICC can _also_ be granted if the ROM firmware is built with a debuggable flag, i.e. `ro.debuggable=1`, whilst configured to accept all channel access requests, `persist.service.seek=fullaccess`. Effectively, this translates to setting `mUseAra` and `mUseArf` fields to false while granting `mFullAccess` in `com.android.se.security.AccessControlEnforcer` according to [the sources](https://android.googlesource.com/platform/packages/apps/SecureElement/+/refs/heads/main/src/com/android/se/security/AccessControlEnforcer.java).

A previous version of this module enables eSE access to third-party apps by exactly patching these 3 flags in `AccessControlEnforcer`. However the approach is more of a one-size-fits-all solution that does not disciminate potentially malicious apps from legitimate ones, posing a significant risk when enabled.

Tracing [the source code](https://android.googlesource.com/platform/packages/apps/SecureElement/+/refs/heads/master/src/com/android/se/Terminal.java) to see how access can be more granularly provided, I found that app eligibility eventually comes down to the check at `isPrivilegedApplication` method in `com.android.se.Terminal`. There is no need to touch `AccessControlEnforcer` at all, as the check directly operates on one of the default steps intended for carrier applications. 

Therefore, this module instead modifies the response there without any drastic changes in terms of how `AccessControlEnforcer` behaves. Of course, it is also possible to be even more specific with the patching, down to the exact application signature rather than using application ID, but this may be a bit more cumbersome to support and maintain.

For users that are still not entirely convinced with how secure this is, you are welcome to cut down the list even further and include your own application ID in your build. Since Android by default does not allow app signatures to be overwritten duing updates, you should be protected so long as your custom LPA app is signed properly 👍

