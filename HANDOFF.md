# Fresh-session handoff

Copy the prompt below into a fresh session.

```text
Continue Omarchy PIN Lock: https://github.com/richkapp/omarchy-pin-lock

Invoke /writing-for-agents now. Read AGENTS.md, SPEC.md, and README.md in the repository before making changes. SPEC.md is the self-contained product contract. This repository currently contains documentation only; no plugin has been implemented or installed.

Goal: a public MIT-licensed Omarchy marketplace plugin with ONE native settings window controlling:
1. Time until the screensaver appears.
2. Time until a six-digit PIN is required.
3. Time until the full account password is required.
Also provide secure PIN enrollment/change/removal, password fallback, visible policy status, and safe install/update/removal.

Preserve the staged PIN requirement. Do not substitute a longer password-only timeout. The example is screensaver at 5 minutes, PIN at 15 minutes, password at 30 minutes, all measured from the same last unlocked activity. Once locked, typing or moving the mouse must not postpone password escalation. The compositor stays locked continuously when PIN mode changes to password mode.

Your first deliverable is the feasibility/architecture decision in SPEC.md section 12, not a production patch. Verify current Omarchy clone routing, authentication service isolation, timer ownership, reason-aware manual/suspend locking, helper packaging, and lifecycle restoration. Recheck related upstream work, especially omacom/omarchy PR #11379, without assuming TPM2 is available or that its broader PAM changes solve this product.

The preferred design is one keep-loaded lock-service replacement plus a settings panel, with a private idle controller and a minimal local authentication helper. The existing generic lock IPC does not distinguish idle from manual/suspend requests. Do not infer the reason from elapsed time or downgrade an explicit lock. Do not expose an unauthenticated unlock method, store a six-digit verifier in user-readable settings, or change global login/sudo/PAM policy.

Use current source to challenge the proposed architecture if a simpler supported solution satisfies the same security contract. Prove it in a disposable environment. Do not run `omarchy plugin clone` against the live desktop merely to inspect it: cloning can enable and replace a live service immediately.

Present the verified architecture, dependency choices, unresolved blockers, and first implementation slice for approval before writing implementation code. After approval, work in small tested slices through the acceptance matrix. Use local fakes/unit tests first, then an isolated graphical session, then explicitly approved real-device tests. Never ask for a PIN/password in chat or print one in logs.

This handoff does not authorize live installation, PAM changes, shell restart, session lock/suspend, pushes, releases, or marketplace submission. Keep those approval boundaries separate. If your environment exposes project-specific host/access instructions, follow them without putting private host details in this public repository.

Keep README status truthful. Completion means a real plugin and helper have passed the specified tests, not only that the UI renders. Return concrete tool evidence and remaining gaps. Do not modify Hermes or unrelated projects.
```
