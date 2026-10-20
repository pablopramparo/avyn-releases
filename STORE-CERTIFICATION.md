# Avyn Agent: technical note for Microsoft Store certification (resubmission)

Product: **Avyn Agent**, Product ID 9PDX4PCD9C96, identity `PabloPramparo.avyn`, publisher Pablo Pramparo.
Contact: avyn.remote@gmail.com. Privacy policy: https://github.com/pablopramparo/avyn-releases/blob/main/PRIVACY.md

This note answers the four points of the rejection of package 1.0.0.0 under policy 10.6.3 (`packagedServices`). Package 1.2.0.0 is a different
implementation from 1.0.0.0, not the same one with a longer explanation: the service now has a defined lifecycle, it is inert until the user turns the
feature on, and the code that types runs in a separate, confined helper. Where something has **not** been verified, this note says so (section 6).

## What the product is

Avyn Agent is the PC side of Avyn Remote, an Android app that turns a phone into a remote control for the PC on the same home network: touchpad,
keyboard, media controls, files, display and power. The Agent is free; the Android app is the only paid product. The Agent only answers phones the
user paired by comparing a verification code on both screens, and every connection is TLS with the phone's certificate pinned.

## 1. Why `packagedServices` and `localSystemServices` are technically required

Only one feature needs them: **"Unlock from the phone"**, optional and off by default. A user who woke their PC remotely (Wake-on-LAN) can type their own PIN
on the Windows sign-in screen from the phone.

The sign-in screen runs on the Winlogon (secure) desktop. Before anyone has signed in there is no user session at all, and a process in a user session
cannot open that desktop. Only a process with LocalSystem rights can create a process on that desktop and attach to it; a scheduled task or a resident
process per session would still need to start a process in another session with those same rights, so it does not avoid the need. The service itself runs in
session 0, which has no access to the console session's desktop; that is why it starts a helper in the console session. A service is the only packaged
way to have such a process available **before sign-in**, because nothing else exists at that moment to start it.

Nothing else in the Agent uses these two capabilities.

## 2. Less privileged architectures we evaluated

| Alternative | Why it is not enough |
| --- | --- |
| **Credential Provider** (the documented way to add a sign-in method) | A COM DLL loaded into LogonUI and registered under HKLM: a bug in it can break sign-in for every user, and it cannot be shipped inside an MSIX. A small service runs outside LogonUI and cannot change how Windows signs users in. |
| **A task or process in the user's session** | Cannot reach the Winlogon desktop (section 1). |
| **`inputInjectionBrokered`** | Injects input into the user's session, not the secure desktop; it would add a restricted capability and not remove the need for the service. |
| **Shipping the Store build without the feature** | Possible, and offered in section 7. We prefer to include it so a user does not have to install a second program. |

## 3. Architecture and lifecycle

Three pieces, and only the first runs with elevated rights:

1. **The service** (`AvynUnlockStore`, LocalSystem, start type `auto`). It decides whether a request may type, and nothing else.
2. **A typing helper**, a single executable that is the same program as the service in "type" mode. The package carries it. At run time the service copies it to a folder
   only SYSTEM and administrators can write (`ProgramData\Avyn\UnlockMsix\helper\<hash>`), verifies its hash while holding it open, and starts it for **one** unlock request
   with a restricted token (no privileges), in a job object limited to one process, with a private environment block. It exits after that request. It makes no network
   connections and downloads nothing. Service and helper talk over a one-use named pipe that only SYSTEM can open, with the network denied, and authenticate each other
   with a one-time secret before anything is sent; the kernel-reported process id, session, user and image path of the peer must match the helper that was started.
3. **The Agent** (user session, no elevation) pairs phones and shows the switch for the feature.

**Lifecycle**

- **Install.** The service is registered by the manifest and starts with Windows. On a clean install it creates its data folder, a log file and the TLS certificate it will
  use, and nothing else. It is **inert**: it opens **no network port**, starts **no helper process** and ignores every request until the user turns the feature on.
  Start type `auto` is required because the feature is used at the sign-in screen after a restart, when nobody is signed in to start anything, and a packaged service's
  start type cannot be changed after install.
- **Turning it on.** The switch is a record (`activation.json`) kept in a folder only SYSTEM and administrators can write, written only by the Agent's step that the user
  approves with a Windows UAC prompt (an ordinary `runas`; `allowElevation` is not declared). Even when on, the service listens only while at least one phone is authorized.
- **Turning it off.** The record is written first (the port closes within about two seconds), then every phone's key and the entitlements are revoked. Revoking the last phone
  also closes the port. The service keeps running, inert, and stays inert after a restart.
- **Update.** The paired phones, the certificate and the record live outside the package and survive an update. A copy of the service installed by our non-Store installer uses
  different names, folder and pipe; the packaged service yields the port if that one is running and never touches it.
- **Uninstall.** The service is removed with the package. The data folder `ProgramData\Avyn\UnlockMsix` (the service's key material, the authorized phones' keys, its log and
  helper copies) is **left behind**; no cleanup is implemented yet. Nothing listens without the service.

## 4. Least privilege at the sign-in screen

- The service **types only a short secret from a paired phone**: no files, no mouse, no screen capture, no other commands.
- It types only while Windows reports that the **input desktop is Winlogon** and the session is signed out or locked, and it checks again **before every key**, before
  clearing the box and before pressing Enter; if that stops being true it stops and sends nothing more. It acts only on the **active console session**.
- On a freshly started PC it first wakes the screen and waits until the sign-in screen has stopped changing; if it never settles, it **types nothing** and reports so.
- Whether Windows accepted the sign-in is reported separately from "keys were sent", based on the observed session change.
- The PIN is never stored on the PC. It lives in the phone's hardware-backed keystore behind a biometric prompt, travels once inside the pinned TLS connection for a
  60-second one-time challenge authenticated with a key derived from the pairing secret, and is handed once to the helper after the helper has authenticated. No log contains it.
- Rate limits: 5 attempts per phone and 15 overall per 5 minutes, plus per-address limits on failures and on challenges; Windows' own account lockout remains.
- Phones can be added only by the elevated step the user approves; revoking a phone is instant.

## 5. Network exposure and capabilities

- Its only network exposure is one TCP port (47822) on private and domain networks, scoped to the service program by a manifest firewall rule; the connection is TLS with a pinned
  certificate that the service generates itself. The package declares four firewall rules (the Agent's TCP 47821 and UDP 47822 on private networks, and the service's TCP 47822 on
  private and domain networks). Windows additionally creates two rules of its own for the declared `privateNetworkClientServer` capability; they are created and managed by the
  operating system.
- Two local pipes exist. The helper's one-use pipe admits only SYSTEM. The Agent's pipe admits any signed-in local user by design, because all it offers is the service's status,
  revoking a phone and a licensing switch for an already registered phone; it can never add a phone. Both refuse remote clients.
- Declared: `runFullTrust`, `packagedServices`, `localSystemServices`, `privateNetworkClientServer`. Not declared: `internetClient` (the Store build never calls the Internet),
  `allowElevation`, `broadFileSystemAccess`, `appDiagnostics`, `inputInjectionBrokered`.
- `runFullTrust` is needed because the Agent is a classic desktop program that injects input, talks to monitors and Windows media sessions, and runs a local TLS server; none of
  that exists inside the app container.

## 6. What we tested, and what we did not

**Tested**

- Automated: more than 1,200 tests, including the secure channel (replay, reordering, tampering, truncation, expiry, an impostor on either side), the helper's confinement and
  verification, the inert-until-enabled behaviour with a real TCP port, and the sign-in sequence on the secure desktop.
- On a Windows 11 virtual machine (build 26300, not a retail build), with a clean install of this package re-signed only for the lab: install, the switch turned on from the Agent with
  a UAC prompt, a phone authorized from the Agent, real unlocks from a phone at the sign-in screen and at the lock screen (including after a full restart) with no failed sign-ins,
  the switch turned off (port closed, phones revoked), and the service still inert after a full restart.
- As a standard local user: cannot list, read, create or write in the service's data folder, and cannot register a phone through the Agent's pipe.
- Windows App Certification Kit 10.0.26100.7705 on Windows 11 build 26200 (retail): overall **PASS**, all 13 required tests pass; run on the final package, built from commit `d2cb802`. The optional "Blocked executables" test reports
  references to process-launch APIs and to names such as `cmd` or `reg`, found mostly in the .NET runtime files the self-contained package includes; in our own code the Agent
  launches the user's programs and files on request, and its elevated step runs `sc.exe` to start the package's own service.

**Not verified, or known limits**

- The behavioural tests above were run on build 26300, not on a retail build. The Agent pipe's refusal of remote clients was measured on an earlier build of the service, not yet on
  the packaged one.
- Like any service registered through a package manifest, its service-control permissions are the operating system's default for such services, which let any authenticated local
  user start or stop it. Stopping it only disables the feature until the next restart; it gives no privilege and no access to keys.
- The service types into the **active console session** only. With several sessions (fast user switching, a remote or Hyper-V enhanced session besides the console), the phone
  unlocks whatever the console shows.
- Registering phones when the requesting user and the administrator who approves the UAC prompt are different accounts is a known defect that we have not fixed.
- Windows Defender / AppLocker / WDAC behaviour with the helper copied under `ProgramData`, and the update path through the Store, have not been tested. The first run of a newly
  created helper file took 5 to 7 seconds in the lab, so the first unlock after installing or updating can be slow.

## 7. If the service capabilities cannot be granted

We can publish the Agent without the unlock service and offer unlock as a separate download from our site. Please tell us which you prefer.
