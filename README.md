# Node Capability Matrix

A source-true reference for what an AI agent can do on a mobile or desktop **node**, and exactly how each capability is gated.

> **Independent reference, not official OpenClaw documentation.** Built on the open-source
> [OpenClaw](https://github.com/openclaw/openclaw) node protocol and verified against its node command
> policy and the iOS/Android node clients.

**Live, designed version:** https://saeronline.io/mobile-edge

---

## Two gates

Every `node.invoke` passes two independent checks:

1. **Declaration** — the node lists the command in its `connect.commands` during the WebSocket
   handshake, and only when the matching capability is enabled and the OS permission is granted on
   the device.
2. **Gateway policy** — the gateway's platform allowlist must permit the declared command.

A command runs only when both pass. `gateway.nodes.allowCommands` adds commands to the allowlist;
`gateway.nodes.denyCommands` removes them and always wins.

## Gating classes

- **Default (`D`)** — allowed out of the box for the platform. The device still needs the capability
  enabled and the OS permission granted.
- **Opt-in / high risk (`O`)** — blocked until added to `gateway.nodes.allowCommands`: `camera.snap`,
  `camera.clip`, `screen.record`, `contacts.add`, `calendar.add`, `reminders.add`, `sms.send`,
  `sms.search`.
- **Not exposed (`—`)** — the platform does not offer the command.
- **Capability gated** — advertised only when the user enables the feature (Camera, Location, Voice
  wake, Installed Apps sharing) or the hardware supports it.
- **Permission gated** — returns an error until the OS permission is granted.
- **Trusted talk** — push-to-talk (`talk.ptt.*`) is default-allowed only for nodes that advertise the
  `talk` capability.

## Mobile nodes (iOS and Android)

| Command | iOS | Android | Foreground | Device requirement |
| --- | --- | --- | --- | --- |
| `canvas.present` `canvas.hide` `canvas.navigate` `canvas.eval` `canvas.snapshot` `canvas.a2ui.*` | D | D | iOS, Android | A2UI needs the canvas host configured |
| `screen.snapshot` | — | — | — | Desktop only; on mobile use `canvas.snapshot` |
| `screen.record` | O | — | iOS | Screen-recording permission (in-app capture) |
| `camera.list` | D | D | iOS, Android | Camera enabled |
| `camera.snap` `camera.clip` | O | O | iOS, Android | Camera enabled + OS permission |
| `location.get` | D | D | iOS: bg needs Always | Location enabled + OS permission |
| `talk.ptt.start` `talk.ptt.stop` `talk.ptt.cancel` `talk.ptt.once` | D | D | iOS | Talk capability (trusted node) |
| `device.status` `device.info` | D | D | — | None |
| `device.permissions` `device.health` | — | D | — | None |
| `device.apps` | — | D | — | Installed Apps sharing |
| `notifications.list` `notifications.actions` | — | D | — | Notification access |
| `photos.latest` | D | D | — | Photos permission |
| `contacts.search` | D | D | — | Contacts permission |
| `contacts.add` | O | O | — | Contacts permission |
| `calendar.events` | D | D | — | Calendar permission |
| `calendar.add` | O | O | — | Calendar permission |
| `reminders.list` | D | — | — | Reminders permission |
| `reminders.add` | O | — | — | Reminders permission |
| `callLog.search` | — | D | — | Call-log permission |
| `sms.send` `sms.search` | — | O | — | SMS permission + telephony |
| `motion.activity` `motion.pedometer` | D | D | — | Motion sensors + permission |
| `system.notify` | D | D | — | Notification permission |

**Legend:** `D` default-allowed · `O` opt-in (high risk) · `—` not exposed / not applicable

The iOS node also declares `chat.push` and `watch.status` / `watch.notify` (Apple Watch); those are
owned by their respective surfaces rather than the core node allowlist.

## Desktop and headless nodes

**macOS node mode** (the menubar app acting as a node) exposes the richest surface: `canvas.*`,
`camera.list` (default) with `camera.snap` / `camera.clip` (opt-in), `location.get`, a real
`screen.snapshot` of the display plus `screen.record` (opt-in), `system.notify`, `system.run` /
`system.which` with `system.execApprovals.*`, and `browser.proxy`. `system.run`, `browser.proxy`,
and `screen.snapshot` are desktop-host commands: held back until the node is connected and declared,
and `system.run` is further gated by exec approvals (ask / allowlist / full).

**Headless node host** (`openclaw node run`, on Linux, Windows, or macOS) exposes only `system.run`,
`system.which`, and `system.execApprovals.*`, plus any plugin-provided node-host commands. No canvas,
camera, or sensors.

## Foreground and background

- **iOS** requires the app foregrounded for `canvas.*`, `camera.*`, `screen.*`, and `talk.*`.
  Background invokes return `NODE_BACKGROUND_UNAVAILABLE`.
- **iOS** `location.get` works in the background only when the location permission is set to
  **Always**; otherwise it returns `LOCATION_BACKGROUND_UNAVAILABLE`.
- **Android** requires the foreground for `canvas.*` (including A2UI) and `camera.*`. Other commands,
  including `talk.ptt.*`, run in the background.
- Voice wake is a separate capability and is best effort while the app is backgrounded.

## Connect an iOS node

A phone is a *node*, not a gateway: it connects to a gateway you run, and pairing is approved by you,
never bypassed (that is Gate 02 in practice).

**You will need:** a machine running the OpenClaw gateway (macOS, Linux, or Windows via WSL2)
reachable from the phone over the same LAN or a tailnet, plus the OpenClaw iOS app.

1. **Run a gateway** (on your Mac, Linux box, or WSL2):
   ```bash
   openclaw gateway --port 18789
   ```
2. **Install the app** — via TestFlight or an Apple build when a release enables it, or build from
   source from `apps/ios` in Xcode.
3. **Discover and connect** — in the app Settings, pick a gateway found on your LAN (Bonjour) or
   tailnet, or enable Manual Host and enter `host:port` (default `18789`).
4. **Approve the pairing** on the gateway host:
   ```bash
   openclaw devices list
   openclaw devices approve <requestId>
   ```
5. **Verify**:
   ```bash
   openclaw nodes status
   ```

Full install, discovery, and troubleshooting: https://docs.openclaw.ai/platforms/ios

## Error codes

- `command not allowlisted` / `command not declared by node` — one of the two gates failed. Check the
  platform allowlist and the node command list.
- `NODE_BACKGROUND_UNAVAILABLE` — bring the node app to the foreground.
- `LOCATION_BACKGROUND_UNAVAILABLE` — set the location permission to Always for background reads.
- `LOCATION_DISABLED` / `LOCATION_PERMISSION_REQUIRED` — enable Location and grant the OS permission.
- `*_PERMISSION_REQUIRED` (for example camera or SMS) — grant the matching OS permission on the
  device.
- `SYSTEM_RUN_DENIED` — the exec approval policy rejected the run.

## Credits

Derived from the open-source [OpenClaw](https://github.com/openclaw/openclaw) project (node command
policy and the iOS/Android node clients) and the [OpenClaw documentation](https://docs.openclaw.ai/nodes).
This is an independent reference and is not official OpenClaw documentation. OpenClaw names and marks
belong to their respective owners.
