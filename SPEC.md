# Omarchy PIN Lock: product and engineering specification

- Status: **specification only; not implemented, installed, or security-audited**
- Specification date: 2026-11-08
- Repository: <https://github.com/richkapp/omarchy-pin-lock>
- License: MIT
- Proposed plugin ID: `io.github.richkapp.pin-lock`

## 1. Outcome and scope

Make frequent returns to an Omarchy desktop convenient without replacing the account password with a short PIN. A user configures one idle timeline in one native window:

1. Show the screensaver.
2. Lock the session and allow a six-digit PIN.
3. Keep the session locked but require the full account password.

The PIN stage is essential. A password-only timeout adjustment does not satisfy this product.

### Required experience

- One discoverable, theme-aware settings window with all three thresholds visible together.
- Six-digit PIN enrollment, change, removal, and a full-password fallback.
- Actual compositor-enforced screen locking, not a fullscreen overlay.
- A password-only stage that cannot be postponed by interacting with the locked screen.
- Safe coexistence with Omarchy, suspend, manual locking, multiple displays, updates, and removal.
- A public MIT project suitable for eventual Omarchy marketplace listing.
- Offline operation with no account, network service, telemetry, or cloud dependency.

### Proposed v1 defaults

These are product proposals for implementation approval, not settings already applied to a device.

| Setting | Default | Meaning |
| --- | --- | --- |
| Screensaver after | 5 minutes | Visual screensaver; not yet a security lock |
| Require PIN after | 15 minutes | Native session lock; PIN or full password can unlock |
| Require full password after | 30 minutes | PIN stops being accepted for this locked cycle |
| Full-password refresh | 24 hours | Maximum interval since this plugin last verified the full password |
| Manual lock | Full password immediately | An explicit lock never silently becomes a weaker PIN lock |
| Suspend/hibernate | Full password on resume | Lock securely before sleep; lid-close follows system suspend behavior |
| PIN failures | 5 | Then require the full password; reopening the UI does not restore PIN attempts |

Do not activate these defaults on installation. Read existing settings, show the proposed changes, and require an explicit enable/apply action.

### Not in v1

Boot or disk-encryption PINs; changing the account password; PIN authentication for login, sudo, SSH, polkit, password managers, or keyrings; fingerprint/face/FIDO2 enrollment; phone or Bluetooth proximity unlock; a second desktop shell; OS-wide idle management; a custom cryptographic algorithm; or a marketplace submission before implementation and review.

## 2. Timing contract

### 2.1 One timeline, three absolute thresholds

Define `S` as screensaver time, `P` as PIN-lock time, and `F` as full-password time. All are measured from **the same last qualifying activity while the session was unlocked**. They are not additive delays.

With `S=5`, `P=15`, `F=30` minutes:

- Before minute 5: ordinary desktop.
- At minute 5: screensaver. Activity before minute 15 dismisses it and begins a new idle cycle.
- At minute 15: secure session lock. A PIN or the full password may unlock.
- At minute 30: the same lock remains held; only the full password may unlock.

Moving the mouse, pressing keys, waking the screen, entering a wrong PIN, opening the password field, or changing display topology while locked does not reset the timeline. A screensaver animation or launch-generated compositor event does not count as human activity.

Successful PIN or password unlock starts a fresh idle cycle. PIN unlock does **not** reset the independent full-password-refresh deadline. The refresh deadline forces password authentication at the next lock, or escalates an existing PIN lock when it expires; it does not interrupt an actively used, unlocked desktop.

### 2.2 Inputs and validation

- UI uses whole minutes, with a readable equivalent timeline; stored values use integer seconds.
- Screensaver may be disabled explicitly. PIN and password deadlines remain enabled while staged unlock is enabled.
- Valid range for v1: positive enabled thresholds up to 240 minutes; `P <= F`; when screensaver is enabled, `S <= P`.
- `P = F` means password-only automatic locking; the UI labels the skipped PIN interval.
- `S = P` means the screensaver and lock start together; authentication is immediately required.
- Zero is not an ambiguous magic value in the proposed plugin schema. Use an explicit enabled flag for screensaver. Reject invalid, fractional, negative, non-finite, out-of-range, or misordered inputs with an inline explanation.
- Suggested full-password-refresh choices: 8, 12, or 24 hours. Do not offer unlimited refresh in v1.
- Import existing timing values without silently replacing them. If they do not fit this UI's supported range, show an unsupported/import warning and leave the existing policy intact until the user chooses a valid policy.

### 2.3 State transitions

| State | Allowed transition | Security behavior |
| --- | --- | --- |
| Disabled | Explicit enrollment and activation | Stock locking remains responsible until activation commits |
| Active/unlocked | Activity resets idle; `S` starts screensaver; `P` requests lock | No authentication merely to dismiss a pre-lock screensaver |
| Screensaver | Activity returns to active; `P` requests lock | Screensaver is not advertised as protecting access |
| Lock pending | Compositor confirms secure lock | Do not report protected or authorize sleep before confirmation |
| PIN locked | Correct PIN or full password; `F`, refresh expiry, failure limit, or strong-lock event escalates | Maintain one continuous native session lock |
| Password locked | Verified full password unlocks | PIN and biometrics do not substitute for the full password |
| Recovery/password-only | Verified full password; repair only after unlock | Corrupt state, helper failure, or uncertain session state never releases the lock |

A generic/manual/suspend lock request transitions directly to password-required, including when a PIN lock is already held. The strongest reason wins and remains latched until successful full-password authentication. Cancel pending PIN attempts and invalidate their results when this escalation occurs; do not return early merely because the session is already locked.

### 2.4 Races, clocks, and lifecycle

- Use a monotonic clock that includes suspend, such as Linux `CLOCK_BOOTTIME`, for trusted deadlines. Wall-clock adjustments and timezone changes have no effect.
- Bind each attempt to a session, boot identity, lock-cycle generation, policy generation, and authentication mode. A completion from an old generation has no effect.
- Check eligibility again when authentication succeeds and immediately before releasing the lock. A correct PIN submitted just before `F` but verified at or after `F` must fail as expired, without unlocking for a frame.
- The helper's PIN deadline is authoritative. A stalled renderer, delayed timer, IPC reconnect, or shell process restart must not extend it.
- Password escalation changes the prompt/mode in place. Never unlock, destroy the secure lock, and start another locker to switch stages.
- Enabling, changing PIN, disabling PIN, or weakening policy requires a fresh full-password verification. The current PIN alone cannot authorize those changes.
- Configuration cannot be edited while locked. Commit changes atomically while unlocked, then start a new idle cycle. Saving a setting is not proof of its live activation.
- Shell/helper restart, reboot, unknown boot identity, or lost authentication state requires the full password before PIN use. Persisted timestamps are not accepted across boots as fresh authorization.
- Existing idle inhibitors are honored before the lock begins. An inhibitor appearing after lock never stops `F` or lowers the required mode. Show when auto-lock is inhibited. Manual lock and suspend protection override presentation/media inhibition.
- Use event-driven timers and subscriptions, not a high-frequency polling daemon.

## 3. The single settings window

Working title: **PIN & Lock**. Make it reachable through an Omarchy menu entry and shell summon; an optional bar launcher is convenience, not the only entry point. All navigation remains in one window. Native authentication prompts may be modal children of that window.

### Primary area

1. **Status:** Disabled, PIN ready, Password required, Helper unavailable, or Compatibility check required. Show configured values and actual active values separately when they differ.
2. **Idle timeline:** Three labeled minute controls in order: Screensaver, Require PIN, Require full password. A compact timeline and sentence explain the resulting intervals. Each control supports keyboard entry and increment/decrement; sliders are optional, never the only input.
3. **PIN:** Set PIN, Change PIN, Remove PIN. Never display the saved PIN. Show only enrolled/not enrolled.
4. **Security:** Password-refresh interval, fixed full-password-on-suspend and manual-lock policy, attempt-limit explanation, and explicit note that this PIN does not unlock login, sudo, encrypted disks, or keyrings.
5. **Actions:** Apply, Cancel, Restore previous settings, and Preview. Preview demonstrates the three stages without taking a real lock or accepting real credentials. A separate clearly labeled lock test requires confirmation.

### Enrollment and management

- First show the scope of changes, helper/package requirements, and recovery instructions.
- Verify the current full account password through the dedicated authentication path. Enter PIN twice in masked fields. Accept exactly six ASCII decimal digits, preserving leading zeros as text.
- Reject documented trivial PIN patterns, at minimum all-identical and straight ascending/descending sequences. Do not imply that a denylist makes a six-digit PIN equivalent to a strong password.
- Require explicit submit by default. Do not automatically send a guess whenever six digits are typed.
- Clear entry buffers promptly on completion, mode switch, cancellation, and timeout. Do not persist UI drafts containing credentials.
- A user who forgets the PIN can select **Use password instead**, unlock with the full password, then replace/remove the PIN after fresh password verification.
- PIN removal disables PIN verification and retires its verifier. It must not accidentally disable automatic locking. Let the user choose an explicit password-only timeout or restore the previous stock policy.
- Password failure feedback must follow the account's PAM policy and avoid revealing stored credentials. PIN failure limits are separate from account password lockout counters.

### Lock surface

- Clearly label **PIN** or **Password** mode, with a persistent full-password alternative in PIN mode.
- Show why a password is needed, for example idle deadline, manual lock, sleep, refresh interval, or too many PIN attempts.
- At `F`, discard the PIN entry and switch to the password field without an unlock gap. Announce the mode change accessibly; do not misinterpret the six digits already typed as a password.
- Hide private notifications/content on every output. Support multiple monitors, fractional scaling, keyboard layouts, screen-reader/accessibility behavior supported by the shell, and keyboard-only operation.
- Do not implement password reveal or clipboard-copy controls on the locked screen. Do not claim perfect memory erasure for QML/JavaScript strings.

## 4. Verified Omarchy integration evidence

Evidence was inspected on 2026-11-08. The observed installation was Omarchy `4.5.0-3` with Hyprland `0.56.2-7` on aarch64. These are evidence targets, not a promised compatibility range.

The current official repository default branch is `quattro`; the reference revision inspected was [`d9cc361cddf5d75dcb880de3c7fb8b5542ae6e39`](https://github.com/omacom/omarchy/commit/d9cc361cddf5d75dcb880de3c7fb8b5542ae6e39). Installed code may differ from that revision; recheck both before implementation.

| Verified surface | Consequence |
| --- | --- |
| Quickshell hosts plugins in one long-running, unsandboxed user process | Do not start a second Quickshell; do not claim isolation from hostile same-user plugins |
| `~/.config/omarchy/shell.json` contains `idle.screensaver` and `idle.lock` in seconds | Preserve unrelated JSON; these are not Hypridle/Hyprlock settings on the observed version |
| Built-in `omarchy.lock` is a `service` with `keepLoaded: true` | Use the service lifecycle and real `WlSessionLock`; hot-reload must not destroy the lock |
| Built-in password and fingerprint PAM flows are separate | A full-password stage must not accidentally accept a fingerprint or another alternate factor |
| `PluginRegistry.qml` uses `omarchy.clonedFrom`, routes enabled clones, and inherits trusted source capabilities | A lock-service replacement has an existing native route; claiming an `authentication` capability alone does not grant it |
| `AuthServiceStore.js` keeps authentication services out of the public service map | Do not try to retrieve credential-bearing lock objects through an ordinary panel's `serviceFor()` |
| `omarchy-system-lock`, idle locking, and pre-suspend locking reach the same `lock.lock` IPC | Current generic lock calls carry no safe idle/manual reason distinction |
| Removing/disabling a clone can restore its first-party source | This does not yet prove a safe active-lock handover or restoration of separately modified idle policy |
| `keepLoaded` preserves the running service during rescan | New service code may need a shell restart; an update must not pretend its new code is already active |

The inspected password service uses `pam_faillock`, `pam_unix` and optional `pam_systemd_home`. Preserve account-management checks and lockout behavior. No authentication files were changed for this specification.

### Related work, not an equivalent implementation

[Omarchy PR #11379](https://github.com/omacom/omarchy/pull/11379), by LoboHacks, proposes TPM2-backed `pinpam` for login, sudo, polkit, and the lock screen. At inspection it was open, unmerged, and conflicted. Its review identifies PAM control-flow and lockout issues. It does not provide this product's staged idle policy or one-window plugin experience, and TPM2 availability cannot be assumed for aarch64 devices. Reuse lessons and attribution; do not apply that broader PAM stack as a shortcut. Hardware-backed PIN storage can be evaluated later without making it a v1 prerequisite.

## 5. Proposed architecture and seams

This section is a recommended design to validate, not a claim that unverified APIs already exist. Security and behavior requirements remain binding if the architecture changes.

### 5.1 Plugin surface

One installable repository and one root manifest. Proposed kinds are `service` plus `panel`, with a keep-loaded service replacing `omarchy.lock` through the supported clone relationship, and a summoned settings panel. Optional bar-widget packaging may follow after the minimal experience works.

- Keep the authentication/lock service separate from the settings object's exposed properties.
- The service owns the native session lock, the mode state machine, and a private idle controller.
- The settings window uses scoped settings APIs and a minimal non-secret status/management interface. It never receives the service's password/PIN buffers or helper internals.
- Preserve the established `lock.lock`, `lock.isLocked`, and `lock.status` compatibility contract, including `requested` versus compositor-confirmed `secure` state. Generic `lock.lock` means full password immediately.
- Only the plugin's private idle controller may initiate a PIN-eligible lock cycle. A public arbitrary `unlock`, `authSuccess`, or caller-selected idle timestamp is forbidden.

### 5.2 Idle ownership and lock reasons

Preferred v1: the replacement service includes its own private idle controller and takes over the built-in `omarchy.idle` responsibility only after successful preflight and explicit activation. This avoids treating an indistinguishable generic manual/suspend lock as an idle PIN lock.

- Preserve stock screensaver launch/dismissal behavior, idle-inhibitor handling, display wake, keyboard layout, password-manager locking hooks, and stay-awake controls where applicable.
- Exactly one idle controller and one session-lock owner may be active after activation.
- Reuse narrowly verified Omarchy behavior rather than creating competing timers alongside the stock idle plugin.
- Trace stock helper side effects before reusing commands: an idle path must not call a generic lock command recursively and accidentally escalate itself or erase its deadline.
- If an official reason-aware extension is simpler and sufficient, prefer it after proving its contract. Do not infer a lock reason solely from how long the user appears idle.

**Lifecycle gate:** safe takeover and restoration of both stock lock and stock idle must be proven for enable, disable, removal, failed load, shell restart, and update. The clone mechanism by itself only establishes lock replacement, not all these guarantees. If stock plugin lifecycle APIs cannot guarantee restoration, document the exact missing seam and obtain a decision before expanding the integration. Do not silently patch packaged Omarchy files or proceed with a known interval of no automatic lock.

### 5.3 Local authentication helper

A six-digit PIN has a small search space. A salted hash in a user-readable JSON file and a counter in QML are not adequate for the promised protection.

Proposed design: a minimal native, system-owned local helper with an unprivileged client path, a root-owned verifier/secret store, and strict per-user/session state. Prefer a small memory-safe implementation and maintained libraries; the language and dependency selections are an implementation-gate decision. Privileged code does not render UI or host QML.

Required properties:

- No TCP listener, arbitrary command execution, arbitrary file access, caller-selected UID, arbitrary PAM service, shell interpolation, or general-purpose privileged RPC.
- Determine UID from authenticated peer credentials and validate the local session/seat through system session facilities. Do not trust a UID, session identity, boot timestamp, or “password verified” boolean supplied by UI code. Peer credentials identify an OS account, not a particular plugin; this is not a security boundary against hostile same-user code.
- Restrict operations to enrollment/change/removal, password verification, PIN verification, policy application, bounded status, and lock-cycle lifecycle.
- Each lock cycle has one authorized live owner and generation. Other requests cannot mint a fresh cycle just to clear lockouts or extend a deadline. Treat reconnect as recovery requiring stronger authentication unless continuity is proven.
- Use authenticated private local IPC, bounded messages/timeouts, request/response correlation, and a clear cancellation protocol. Never pass credentials in process arguments, environment variables, shell commands, diagnostic dumps, or files used as IPC mailboxes.
- Helper-side state owns PIN attempt counts, PIN eligibility, policy generation, strong-lock latches, and password-refresh authorization.
- Store PIN verifiers using a maintained Argon2id implementation with a random per-credential salt and a separate root-only secret/pepper through the library's supported design. Record algorithm and work parameters for upgrades. Exact parameters need benchmark and review on both target architectures; do not invent cryptography in the plugin.
- Store verifier/pepper/policy data in system-owned directories with restrictive permissions, atomic writes and symlink/traversal protection. A copied user home must not contain an offline-verifiable PIN database. This does not protect against an attacker who obtains root/system secrets.
- Verify full passwords through PAM with account checks. A dedicated screen-unlock-only service must preserve existing `pam_faillock` and applicable `pam_systemd_home` behavior while excluding alternate factors from the full-password branch. Prove the actual PAM control flow for supported configurations; do not copy numeric jump rules blindly or edit `system-auth`, `sudo`, `login`, or `polkit-1`.
- Unknown/custom PAM configurations require compatibility review or a clear unsupported state. They are not an excuse to drop account policy.
- A short-lived, purpose-bound full-password authorization may cover one enrollment/policy transaction. It is not a reusable general authentication token.
- Installer/admin authentication and the screen-unlock account password are distinct. Enrollment must not bind a different administrator's password/PIN to the current user's account.

### 5.4 PIN limits and fallback

- Serialize/count attempts across parallel windows and clients before checking the PIN. At most five failed PIN attempts are allowed before full-password-only mode.
- Enforce a non-busy-wait delay after each failure, starting at one second and increasing up to eight seconds. The fifth failure disables PIN until a successful full-password verification resets it.
- Counter state survives UI close, shell reload, helper restart, and a new idle cycle. Failed attempts accumulate until successful full-password verification; a successful PIN does not replenish the budget. A reboot or lost state requires the full password before PIN is available, rather than silently creating a new budget.
- Revalidate account eligibility before PIN unlock. A disabled/expired account, changed account identity, or revoked enrollment cannot be bypassed with a still-valid PIN. Password changes and recovery resets must invalidate prior PIN authorization or force full-password reauthorization; specify and test the supported account-provider signal during the PAM feasibility gate.
- UI requests and timers cannot lower the required mode. PIN mode may only be re-established by an authorized new unlocked cycle with valid strong-auth freshness.
- Full-password fallback remains available when the PIN is expired, blocked, or unavailable. Helper loss must not remove the native lock or leave the password path absent. Use the existing password-only PAM surface as the fail-closed fallback where proven safe; require helper reauthorization before PIN returns.
- A fallback success may unlock the real session but must not silently renew helper PIN privileges with an untrusted “success” notification.

## 6. Threat model and honest guarantees

Protect against an unattended person using the keyboard, guessing PINs, trying to reset limits by interacting with the lock UI, exploiting timing races, and unprivileged other-user access to credential state. Keep the existing compositor and PAM protection intact.

Do not claim protection against root compromise, malicious code already running as the logged-in user, a malicious unsandboxed Omarchy plugin, a compromised compositor/kernel, or an attacker who obtains the root-only credential secrets. Omarchy plugins are not a sandbox. Full-disk encryption and secure boot are separate controls.

A short software PIN is a convenience credential with lower resistance than a strong password; helper-side attempt limits and protected storage are necessary but not a claim of Windows Hello/TPM-equivalent protection. No TPM or Apple secure hardware is assumed.

Additional invariants:

- Only verified authentication for the current session/cycle can release `WlSessionLock`.
- No anonymous IPC verb may directly mark authentication successful or unlock.
- A helper failure, malformed response, expired response, policy mismatch, storage failure, or unknown state selects password-only/recovery while retaining the lock.
- Multi-output coverage and compositor `secure` confirmation are part of locking, not just UI polish.
- A sleep lock request is not proof that sleep is safe. Keep the delay inhibitor until compositor-confirmed protection, but do not assume it can block sleep indefinitely. If its budget expires, report an unsecured failure explicitly and never return a false secure result. The implementation gate must prove a supported mechanism to prevent/defer unsafe suspend, or mark that system integration unsupported for activation; a settings checkbox cannot promise a guarantee the OS path does not enforce.
- Password failures use the account's existing lockout policy. PIN denial-of-service must not become a global account lockout.
- Diagnostics are opt-in and credential-free. No raw authentication messages, entered lengths/prefixes, verifier material, or sensitive memory dumps in support exports.
- User settings are never treated as a privileged assertion that a deadline, authentication, or rate-limit reset occurred.

## 7. Proposed configuration model

This is a schema proposal, not an existing runtime file or command.

Non-secret policy fields:

- `schemaVersion`
- `enabled`
- `screensaverEnabled`, `screensaverAfterSeconds`
- `pinAfterSeconds`, `passwordAfterSeconds`
- `passwordRefreshHours`
- `manualLockPolicy: full-password`
- `suspendPolicy: full-password`
- `revision`

Enrollment status, attempt counters, strong-auth timestamps, helper generation, PIN verifier, and pepper are not user-writable configuration. UI receives only the minimum non-sensitive status needed to render.

Store authoritative authentication policy with the helper; use the plugin's native settings for presentation or non-secret desired values only. Commit UI and helper policy with revision checks, explicit acknowledgement, and rollback on failed activation. Preserve unrelated `shell.json` fields and concurrent changes.

Keep a minimal installation receipt of the exact prior stock idle/lock values and keys the plugin owns. Removal uses compare-and-restore: never overwrite unrelated user edits or a newer external configuration. Conflicts must be shown and resolved without leaving the machine unprotected.

## 8. Install, update, disable, and removal

### Installation and activation

1. Validate supported Omarchy/Quickshell/Hyprland, current session, PAM fallback, display coverage, helper package signature/hash, permissions, and recovery access.
2. Add the plugin in a disabled/password-only preparation state. Installation must not silently change timers, weaken locking, or enable PIN.
3. Explain the privileged helper requirement and obtain native administrator consent. No network install from a root `curl | sh` pipeline and no broad passwordless sudo rule.
4. Show before/after policy and verify the full account password. Enroll PIN through protected IPC.
5. Stage the replacement and complete the tested idle/lock ownership transaction while unlocked. Confirm live service readiness and effective settings before reporting enabled.
6. On any failure, keep or restore the proven stock password lock and prior timers; never claim the PIN plugin is active from a saved config alone.

### Updates and reload

- Review pinned source changes before update; release artifacts must identify exact source and dependency lock versions.
- Do not replace a running lock owner or restart Quickshell while locked. Defer code activation until a safe unlocked transition.
- `keepLoaded` is required, but does not itself implement safe package removal or restart.
- A failed update retains the previous functioning owner and rollback data. A mismatched helper/plugin protocol fails to password-only, not permissive compatibility guessing.

### Disable/remove

- Ask for fresh full-password verification for in-app disable/removal. Defer mutation while locked.
- Restore stock lock, idle controllers, and the user's prior values before retiring the helper and plugin integration.
- Prove ordinary marketplace/plugin CLI disable and removal do not strand a locked session or leave auto-lock disabled. An in-app uninstall button alone is insufficient.
- Remove only project-owned helper, policy, verifier, and service artifacts. Never remove the account, account password, global PAM files, or unrelated plugin settings.
- If recovery requires restarting the desktop session, tell the user explicitly and obtain separate consent. Do not silently terminate work.
- Provide tested TTY/admin recovery instructions in release documentation. Development recovery must be available before a real-user lock test; do not advertise a passwordless bypass.

## 9. Marketplace and open-source delivery

The current marketplace requires a public GitHub repository, valid root `manifest.json`, README, license, and safe install/removal. Submission goes through the official issue form. Automated listing validation is not a security review.

### Manifest and listing decisions to prove

- Proposed permanent ID: `io.github.richkapp.pin-lock`, not reserved `omarchy.*`.
- Proposed root manifest uses `schemaVersion: 1`, explicit MIT license, service/panel entry points that actually exist, and `keepLoaded: true`.
- Preserve `omarchy.clonedFrom: omarchy.lock` **only if verified as the supported production replacement contract**. The generic marketplace tutorial says to remove clone metadata before publishing, while the inspected runtime uses it for service routing, restoration, and authentication capability inheritance. Resolve this discrepancy with the validator/current maintainer contract before release. Do not remove required metadata merely to make a listing look normal, or fake first-party trust.
- Declare the authentication/helper privilege boundary, root package dependency, background service, compatibility range, recovery procedure, and uninstall behavior prominently.
- Use a real settings screenshot without private data and a clear description of what the PIN does and does not unlock.
- Ship reproducible or independently verifiable helper packages for aarch64 and x86_64. Select the exact packaging/delivery mechanism during feasibility work; do not promise installation in one command until exercised.
- Preserve the Omarchy copyright notice and MIT terms in any copied/adapted source. Documentation in the initial repository is newly authored; no Omarchy source is vendored yet.

This specification does not contain a root manifest or stub entry points. That is intentional: the repository must not appear installable before a real, validated plugin exists. Do not submit it to the marketplace in the specification phase.

## 10. Acceptance and verification matrix

Use a controllable clock and fake authentication for deterministic unit tests. Use a disposable graphical session and real test account for integration, then explicitly authorized hardware tests. Time-accelerated tests complement, not replace, a real example-timeline run.

| ID | Acceptance condition | Required evidence |
| --- | --- | --- |
| UX-01 | One window exposes all three timers with clear units, a common timeline, validation, Apply/Cancel, and enrollment status | Keyboard and pointer UI test; theme/scaling screenshots |
| UX-02 | PIN enrollment/change/removal requires a fresh full password, preserves leading zeros, and rejects mismatch/invalid input | Isolated helper/UI integration; no secrets in logs |
| TIME-01 | Example timeline reaches screensaver, PIN lock, and password mode at the three absolute deadlines | Fake-clock tests plus one real-duration run |
| TIME-02 | Activity before locking resets the idle cycle; locked activity and failed attempts never extend `F` | Event/race tests |
| TIME-03 | At exact `F`, a valid PIN is rejected; an in-flight PIN success after `F` cannot unlock | Boundary and delayed-result tests |
| TIME-04 | PIN unlock resets idle, not password-refresh age; full password resets both; unlocked active work is not interrupted by refresh expiry | State-machine tests |
| TIME-05 | Manual lock/suspend during PIN mode escalates immediately, cancels pending PIN checks, and rejects their late successes | Native IPC, verification-race and sleep tests |
| AUTH-01 | Only current-cycle verified authentication releases the compositor lock | Replay, parallel request, cancellation and stale-generation tests |
| AUTH-02 | Five wrong PINs require password; concurrent requests, UI restart and helper restart cannot replenish attempts | Helper process tests with persistent-state inspection |
| AUTH-03 | PIN never authenticates login/sudo/polkit/disk/keyring, cannot bypass account disable/expiry, and is reauthorized after a password change; fingerprint never satisfies password-only mode | PAM fixture tests, alternate-factor bypass tests and isolated account checks |
| AUTH-04 | Corrupt config/verifier, missing helper, IPC failure and storage-full conditions leave password-only protection | Fault injection and usable recovery demonstration |
| AUTH-05 | Malformed/oversized IPC, wrong UID/session, arbitrary paths/services and duplicate ownership are rejected | Negative security tests |
| LOCK-01 | One real `WlSessionLock` covers every output, including hotplug, before lock reports secure | Native multi-output test and compositor evidence |
| LOCK-02 | PIN-to-password escalation never exposes desktop frames or returns input to ordinary applications | Instrumented native lock test; not screenshot-only proof |
| LOCK-03 | Lock client crash does not unlock; supported recovery requires authentication | Controlled crash and recovery test |
| LIFE-01 | Enable failure retains stock protection; a successful enable leaves exactly one idle controller and one locker | Process/plugin inventory and live behavior |
| LIFE-02 | Rescan, theme change, plugin update, disable, direct CLI removal and shell restart preserve safe ownership and effective idle locking | Lifecycle matrix, including locked and unlocked attempts |
| LIFE-03 | Removal restores owned settings without clobbering concurrent edits and retires only project artifacts | Before/after configuration and package comparison |
| LIFE-04 | Sleep/lid-close/resume/reboot require password as specified, do not re-enable PIN from stale state, and handle exhausted sleep-inhibitor budgets without falsely claiming protection | Approved native tests, rapid sleep/resume and failure injection; distinguish unsupported hardware behavior |
| PRIV-01 | No credential appears in logs, process argv/env, crash exports, user settings, public repo, or screenshots | Automated redaction/fixture checks and manual review |
| DIST-01 | aarch64 and x86_64 builds pass; QML and plugin validation succeed on the declared Omarchy versions | Exact build/test logs and commit IDs |
| DIST-02 | Install/update/removal instructions work from a clean supported system | Disposable-system run and documented privileges |
| DIST-03 | Marketplace validation accepts the real manifest and replacement metadata; maintainer requirements resolved | Validator output and any required approval before submission |

Before calling a release safe to test on a primary laptop, have an independent reviewer examine PAM control flow, helper IPC authorization, attempt persistence, deadline authority, secret handling, and lock lifecycle. Track findings to resolution. Do not describe this as a formal security audit unless one actually occurred.

## 11. Implementation slices

1. **Contract and feasibility:** resolve section 12 without modifying a live desktop. Pin upstream evidence and choose the minimal supported integration.
2. **Pure policy model:** timer validation, state machine, refresh age, lock reasons, and stale-result rejection with deterministic tests.
3. **Settings shell:** one-window UI against a fake service; no real PIN or password storage and no authentication claims.
4. **Helper and credential lifecycle:** privileged boundary, PAM verification, PIN enrollment, protected storage, limits, protocol, and negative tests in isolation.
5. **Native lock integration:** real compositor lock, private idle controller, password fallback, deadline escalation, and display/suspend behavior in a disposable session.
6. **Transactional lifecycle:** package setup, activation, rollback, update, disable and ordinary CLI removal, with both stock services restored correctly.
7. **Review and distribution:** independent review, architecture matrix, documentation, packaging, and only then an explicitly approved marketplace submission.

A later session must obtain approval for implementation before starting source/runtime patches. The current task authorizes only this specification, handoff, and creation/publication of this documentation repository.

## 12. Feasibility decisions required before implementation

The product is specified; these engineering seams are intentionally not misrepresented as proven.

1. **Authentication-service replacement:** demonstrate supported persistent clone routing and marketplace acceptance. Check first-party capability inheritance and no credential-object exposure to the settings panel.
2. **Lifecycle restoration:** establish a safe, supported transaction restoring both idle and lock on generic disable/remove/load failure. Determine what the host can do while locked. A small upstream extension may be necessary; it needs separate proposal/approval rather than a covert patch.
3. **Reason-aware timing:** prove private idle events versus generic strong-lock events and preserve suspend confirmation, screensaver, inhibitors, and stock side effects. Choose one owner, not two racing idle plugins. Determine the enforceable response when the sleep-inhibitor budget expires before a secure lock exists; fail activation on unsupported guarantees rather than inheriting stock timeout behavior unnoticed.
4. **Helper/authentication boundary:** select maintained libraries, packaging, service privileges, peer/session authorization, authoritative clocks, storage, rate-limit persistence, and password-only fallback. Benchmark PIN hashing on the two target architectures.
5. **PAM support contract:** document supported account models and alternate-factor exclusion without removing account checks or breaking systemd-homed. Unknown stacks must fail safely and explain remediation.
6. **Testing/recovery environment:** establish a disposable native graphical session with an alternate authenticated recovery route before any lock test. Verify hardware support instead of assuming TPM or fingerprint availability.

If one gate fails, report the exact missing capability and the smallest supported alternative that still delivers staged PIN unlock. Do not quietly replace the product with password-only locking.

## 13. Sources and provenance

Primary public references, retrieved 2026-11-08:

- [Omarchy plugin development](https://plugins.omarchy.org/develop.html)
- [Omarchy marketplace publishing](https://plugins.omarchy.org/publish.html)
- [Marketplace submission form](https://github.com/omacom/omarchy-plugin-marketplace/issues/new?template=submit-plugin.yml)
- [Pinned Omarchy shell reference](https://github.com/omacom/omarchy/blob/d9cc361cddf5d75dcb880de3c7fb8b5542ae6e39/docs/omarchy-shell.md)
- [Pinned built-in plugin reference](https://github.com/omacom/omarchy/blob/d9cc361cddf5d75dcb880de3c7fb8b5542ae6e39/shell/plugins/README.md)
- [Pinned plugin registry](https://github.com/omacom/omarchy/blob/d9cc361cddf5d75dcb880de3c7fb8b5542ae6e39/shell/services/PluginRegistry.qml)
- [Pinned shell host and authentication-service lifecycle](https://github.com/omacom/omarchy/blob/d9cc361cddf5d75dcb880de3c7fb8b5542ae6e39/shell/shell.qml)
- [Pinned lock service](https://github.com/omacom/omarchy/blob/d9cc361cddf5d75dcb880de3c7fb8b5542ae6e39/shell/plugins/lock/Service.qml)
- [Pinned idle service](https://github.com/omacom/omarchy/blob/d9cc361cddf5d75dcb880de3c7fb8b5542ae6e39/shell/plugins/services/idle/Service.qml)
- [Wayland ext-session-lock protocol](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/ext-session-lock/ext-session-lock-v1.xml), also inspected through its [GitHub mirror](https://github.com/wayland-mirror/wayland-protocols/blob/main/staging/ext-session-lock/ext-session-lock-v1.xml)
- [Related TPM2 PIN proposal, not merged at inspection](https://github.com/omacom/omarchy/pull/11379)

Installed counterparts under `/usr/share/omarchy` and the active idle status were inspected read-only. No private machine identifiers, credentials, or user screenshots are required to implement this specification.
