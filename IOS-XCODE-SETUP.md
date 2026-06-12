# OpenClaw iOS Node — Xcode Setup Handoff

A from-source guide to building the OpenClaw iOS app in Xcode and connecting it to a
gateway as a `role: node`, so an agent can drive the phone's capabilities (see the
[Node Capability Matrix](README.md) for what those are and how they're gated).

> **Super-alpha, internal use.** The iOS app is early. Expect breaking changes and rough
> edges. Foreground use is the only reliable mode today. There is no public App Store build —
> building from source in Xcode (this guide) is the supported path. (Source of truth:
> `apps/ios/README.md` in the OpenClaw repo.)

You end up with: the OpenClaw app on your iPhone/iPad, paired to a gateway on your Mac, and
node commands working from the CLI.

---

## 0. What you need

- A **Mac with Xcode 16+** (the project targets **iOS 18.0**, Swift 6).
- **Node ≥ 22.19** and **pnpm**.
- **XcodeGen** — `brew install xcodegen` (the Xcode project is generated, not checked in).
- An **Apple ID** for signing. A free personal team works for installing on your own device
  (the app expires after 7 days; just reinstall). A paid Apple Developer account avoids the
  expiry and is required for push (APNs).
- An **iPhone/iPad on iOS 18+** (recommended — camera/location/motion need real hardware) or
  the iOS 18 Simulator (UI/canvas only).
- The Mac (gateway) and the phone on the **same LAN** (or a shared tailnet).
- A clone of the OpenClaw repo: `git clone https://github.com/openclaw/openclaw`

---

## 1. Run a gateway (the phone connects to this)

A node is a peripheral; it needs a gateway to talk to. From the repo root:

```bash
pnpm install
pnpm openclaw gateway --port 18789
```

Leave it running. (If `pnpm openclaw` errors on a fresh clone, run `pnpm build` once, then retry.)

Reachability: the phone must be able to reach the Mac at `:18789`. Same-LAN discovery uses
Bonjour automatically; if discovery is blocked you'll enter the Mac's LAN IP manually in step 3.
For tailnet/manual setups see https://docs.openclaw.ai/gateway/discovery.

---

## 2. Build and run the app in Xcode

From the repo root, one command does signing config + version + project generation + opens Xcode:

```bash
pnpm ios:open
```

(Equivalent manual steps, if you prefer:)

```bash
./scripts/ios-configure-signing.sh
cd apps/ios
xcodegen generate
open OpenClaw.xcodeproj
```

In Xcode:

- **Scheme:** `OpenClaw`
- **Destination:** your connected iPhone/iPad (recommended) or an iOS 18 Simulator
- **Configuration:** `Debug`
- **Product → Run** (⌘R)

First run on a device: on the iPhone, trust the developer profile under
**Settings → General → VPN & Device Management**.

### If signing fails

Usually you don't need to touch signing at all: `pnpm ios:open` / `pnpm ios:gen` run
`scripts/ios-configure-signing.sh`, which detects your Apple Team ID from Xcode and writes a
git-ignored `apps/ios/.local-signing.xcconfig` with **Automatic** signing and **unique
per-developer bundle IDs** (`ai.openclaw.ios.test.<you>-<teamid>`), overriding the repo defaults
(`ai.openclaw.client*`). Personal-team bundle-ID collisions are handled for you.

If the script warns `Unable to detect an Apple Team ID` (no Apple account signed into Xcode yet),
or you want explicit control, set signing manually:

```bash
cp apps/ios/LocalSigning.xcconfig.example apps/ios/LocalSigning.xcconfig
```

Edit `apps/ios/LocalSigning.xcconfig` (it overrides both the repo defaults and the auto-generated
`.local-signing.xcconfig`):

- Set `OPENCLAW_DEVELOPMENT_TEAM` to your Team ID (Xcode → Settings → Accounts).
- On a personal team, also change each bundle ID to something unique, e.g.
  `ai.openclaw.client` → `com.yourname.openclaw.client` (and the matching `.share`,
  `.activitywidget`, `.watchkitapp*` ones).

Then regenerate and run again:

```bash
pnpm ios:gen
```

---

## 3. Pair and connect

1. With the gateway running and the app launched, open the app's **Settings → Gateway**.
2. Pick the discovered gateway, **or** enable **Manual Host** and enter the Mac's LAN IP +
   `18789`. Accept the TLS fingerprint trust prompt if shown.
3. Connecting creates a **device pairing request**. Approve it on the gateway host:

   ```bash
   pnpm openclaw devices list
   pnpm openclaw devices approve <requestId>
   ```

   (If the app retries with changed auth, a new `requestId` is created — re-list before approving.)
4. Verify the node is paired and online:

   ```bash
   pnpm openclaw nodes status
   ```

---

## 4. Try a capability

Foreground the app first (most commands are blocked in background). Then, from the gateway host:

```bash
# snapshot the in-app canvas (canvas has no CLI subcommand; drive it via `nodes invoke`)
pnpm openclaw nodes invoke --node "<id-or-name>" --command canvas.snapshot --params '{"format":"png"}'

# get location (enable Location in the app first)
pnpm openclaw nodes location get --node "<id-or-name>"
```

What's available and how each command is gated: see the [Node Capability Matrix](README.md) or
the designed version at https://saeronline.io/mobile-edge.

---

## Gotchas

- **Foreground-first.** `canvas.*`, `camera.*`, `screen.*`, and `talk.*` return
  `NODE_BACKGROUND_UNAVAILABLE` when the app is backgrounded.
- **Simulator limits.** No real camera/GPS/motion on the Simulator — use a device for those.
- **iOS 18.0 minimum.** Older devices won't run this build.
- **Background location** needs the **Always** permission.
- **Push (APNs) is optional** for basic node use and needs proper provisioning + gateway APNs
  credentials. Skip it unless you need wake/push; see `apps/ios/README.md` for the full APNs setup.
- **Free team = 7-day expiry.** Reinstall from Xcode to renew, or use a paid account.

## Troubleshooting

- Regenerate the project after any signing/config change: `pnpm ios:gen`.
- In **Settings → Gateway**, check the status text / server / remote address; enable
  **Discovery Debug Logs** if the gateway isn't found.
- Can't connect over LAN? Switch to **Manual Host** + the Mac's IP, and confirm the gateway is
  reachable at `:18789` from the phone.
- In the Xcode console, filter for `ai.openclaw.ios`, `GatewayDiag`, or `APNs registration failed`.
- Pairing/auth errors intentionally pause the reconnect loop until you fix pairing
  (`pnpm openclaw devices approve <requestId>`), then reconnect from the app.

## Reference

- Canonical build doc: `apps/ios/README.md` (OpenClaw repo)
- OpenClaw iOS docs: https://docs.openclaw.ai/platforms/ios
- Node capability matrix: [README.md](README.md) · https://saeronline.io/mobile-edge
