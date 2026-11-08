# Omarchy PIN Lock

One window for screensaver timing, six-digit PIN unlock, and full-password escalation on Omarchy.

**Status: specification only. There is no installable plugin or authentication implementation yet.**

The intended experience:

- Start a screensaver after a configurable period of inactivity.
- Require a six-digit PIN after a second threshold.
- Require the full account password after a longer threshold.
- Keep all three timers, PIN management, and security options in one native settings window.

Example: screensaver at 5 minutes, PIN at 15 minutes, full password at 30 minutes. All thresholds start from the same last-activity time, not from the previous stage.

## Documents

- [Product and engineering specification](SPEC.md): behavior, integration evidence, security requirements, acceptance tests, and implementation gates.
- [Fresh-session handoff](HANDOFF.md): copyable instructions for continuing the project.
- [Contributor instructions](AGENTS.md): scope and safety boundaries for agents and contributors.

## Intended platform

Omarchy Quattro with its Quickshell plugin system and native Wayland session locking. Both aarch64 and x86_64 are intended targets. Exact supported versions will be established by implementation testing.

The proposed implementation is a real lock-service plugin with a narrowly scoped local authentication helper. It is not an overlay that merely hides the desktop, and it does not change the user's login, sudo, disk-encryption, or keyring password.

Marketplace publication is a future milestone. A repository and marketplace validation are not evidence that an authentication plugin is safe.

## License

[MIT](LICENSE). Preserve upstream notices if implementation later includes upstream code.
