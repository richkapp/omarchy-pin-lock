# Contributor instructions

## Current stage

This repository contains a specification, not a working plugin. `SPEC.md` owns the product contract. `HANDOFF.md` defines the next session's scope. Keep this status explicit until real execution proves otherwise.

## Start here

1. Read `SPEC.md` and `HANDOFF.md`.
2. Recheck the installed Omarchy plugin contract and the pinned evidence before choosing an integration seam.
3. Resolve the feasibility gates before implementing authentication. Present the concrete architecture and obtain owner approval before source/runtime patches. A spec or handoff is not deployment permission.

## Safety and scope

- Develop in an isolated repository/test environment. Obtain separate approval before installing a plugin/helper, editing PAM, changing live idle/lock policy, restarting a shell, or locking/suspending a person's session.
- Preserve the real Wayland session lock continuously during authentication changes. Failure keeps the session locked; it never bypasses authentication.
- Keep PIN authentication specific to screen unlock. Preserve full-password recovery and existing account authentication protections.
- PINs, passwords, verifier records, pepper material, private logs, host access details, and screenshots of real user data do not belong in this public repository or agent chat.
- Use maintained cryptographic and PAM libraries rather than implementing cryptography or shell-based password checking.
- Preserve attribution for reused upstream code. The project license is MIT.
- Keep work focused on this project. Do not modify Hermes, unrelated services, or system-wide authentication as a shortcut.

## Verification and publication

Tests must exercise timing boundaries, stale asynchronous results, authentication attempt limits, crashes, multiple outputs, suspend, reload, and uninstall. A passing settings mockup is not a working lock plugin. Report the exact platform and tests actually exercised.

Commits, pushes, release creation, marketplace submission, live installation, and service lifecycle are distinct actions. Obtain the applicable authorization rather than inferring it from implementation intent.
