# Mika — Security Review (English)

Independent security-focused review of `wangzan101/Mika` (fork: `paul-english/Mika`),
commit `4ce3ca6` ("版本 0.1.34"). Kotlin Android app, Apache-2.0.

Mika is a phone AI agent: it drives a hidden virtual display through an Accessibility
Service, runs a per-conversation Linux sandbox (PRoot + a seccomp launcher), embeds a
Chromium WebView browser, and talks to a user-supplied LLM endpoint. It also exposes a
local `MCP` server (LAN / USB / outbound relay) so a computer can hand it tasks.

**Overall**: the security posture is unusually deliberate for this class of app. Most of
the dangerous primitives (shell-identity helper, ADB pairing, self-update, the network
entry point, the sandbox) are constrained on purpose, and the reasoning is written into
the code. The findings below are mostly *residual risk* and *hardening*, not exploitable
bugs found in the audited paths.

---

## 1. What is done well

### 1.1 Secret storage
- Provider API keys are encrypted with **AES-256-GCM** under a key held in the
  **Android Keystore** (`settings/KeyCipher.kt`, `KeystoreKeyCipher`) — non-exportable,
  void on uninstall. IV is random per record, 128-bit tag. No keys are logged anywhere
  (grepped for key/token/secret in log calls: only non-secret "refused keyCode" hits).

### 1.2 The computer-facing entry point (MCP / local HTTP server)
`core/remote/RemoteAccess.kt`, `RemoteHost.kt`, `HttpServer.kt`:
- **Constant-time token comparison** (SHA-256 both sides + `MessageDigest.isEqual`),
  and both the general and loopback tokens are always compared (no early exit).
- **Per-source rate limiting**: 5 failures / 60 s -> 60 s lockout (HTTP 429). Correct
  tokens are *also* blocked during cooldown, so the lockout can't be used as an oracle.
- **`Origin` header -> 403** on every request (DNS-rebinding / browser defense).
- **Private-address allowlist at the socket layer**: `accept()` drops any connection
  whose local or remote peer is not loopback / RFC1918 / CGNAT(100.64/10) / link-local
  / ULA. Public-internet peers are refused with no bytes written.
- **Bind scope**: loopback only for USB mode; all interfaces only in explicit
  "private networks" mode. Head/body size caps (16 KB / 256 KB), 16-connection cap,
  30 s read timeout.

### 1.3 The shell-identity helper handoff
`core/shell/HelperHandoff.kt` — the app must accept a Binder from a process running as
`shell` (uid 2000) started via ADB. Checks before accepting:
- `callingUid == 2000` (shell only), and the helper's self-reported uid/pid must match
  what Binder reports (`IdMismatch`);
- the helper must be running **this** APK (cert-pinned to the installed signer);
- a **one-time nonce**, passed via **stdin, never the command line** (so other shell
  processes can't read it from `/proc/<pid>/cmdline`); nonces have TTLs and are all
  invalidated when any one is consumed.
- Exported entry points (`HelperHandoffProvider`, `McpService`,
  `SelfCheckTargetActivity`) are all guarded by `INTERACT_ACROSS_USERS_FULL`, a
  signature|privileged permission held only by shell/system — ordinary apps cannot
  reach them. Verified in `AndroidManifest.xml`.

### 1.4 Self-update
`about/ApkIntegrity.kt`, `Releases.kt`: downloads are **HTTPS-only from the official
GitHub release path**; after download it verifies size, **SHA-256**, package name,
version, and — the real anchor — the **signer certificate matches the installed app**
(with signature-rotation history). Sharing `.apk/.apks/.xapk/.aab` out of the sandbox
is refused.

### 1.5 The Linux sandbox
`app/src/main/cpp/mikarun/mikarun.c` + `core/cap/sandbox/*`, `cap/sandbox/*`:
- The launcher applies **seccomp**, rlimits (max procs, max file size), a wall-clock
  timeout, and **network is OFF by default** (`--net on` is opt-in and user-gated).
- Only a handful of character devices are bound into PRoot; **`/dev/binder` and
  `/dev/socket` are deliberately NOT bound** (they could otherwise be used to exfiltrate
  through system services and bypass the network-off guarantee).
- Path handling goes through a gate that resolves symlinks with NOFOLLOW and normalizes,
  refusing escapes; file sharing out is confined to `/workspace`, size-capped at 100 MB,
  and refuses installer file types and control/zero-width characters in names.

### 1.6 ADB pairing identity
`core/adb/AdbTls.kt`, `Spake2.kt`, `AdbIdentity.kt`: TLS 1.3 with a self-signed cert and
a deliberate **trust-all** `TrustManager` — safe *here* because identity is established
out-of-band by SPAKE2 pairing (the pairing key is bound to the TLS exporter) and the
device verifies the app's certificate public key at connect time. Connections target
the local adb daemon only (`AdbTls.connect` is driven by `LocalPortScan`, which sweeps
`127.0.0.1` then the local Wi-Fi address).

### 1.7 Process / update hygiene
- `allowBackup="false"`; no `android:debuggable`; no hardcoded secrets or public hosts
  in code (grep for high-entropy string assignments and http(s) hosts: only test fixtures
  and scheme literals).

---

## 2. Residual risks & recommendations

Ordered by how much they matter for a real user.

### R1 — Global cleartext HTTP is permitted (Medium)
`res/xml/network_security_config.xml` sets
`<base-config cleartextTrafficPermitted="true">` for the **whole app**. The stated reason
is good: users run local/LAN model servers (e.g. `http://192.168.1.10:11434/v1`) that
have no HTTPS. But the effect is that *any* `http://` the app is pointed at is allowed,
so if a user configures a **non-local** plaintext endpoint, their **API key and all
prompt/screen content travel unencrypted**. They warn only when the host is neither
loopback nor LAN.
Recommendation: keep cleartext for private ranges, but make the "non-local http" case a
hard, explicit confirmation (or block it), so a typo can't silently downgrade a public
endpoint to plaintext.

### R2 — WebView is always debuggable (Medium)
`cap/web/cdp/WebDevtools.kt` calls `WebView.setWebContentsDebuggingEnabled(true)`
unconditionally. This is required for Mika's CDP-driven browsing, but it exposes the
app's WebView (including **logged-in sessions**) to anything that can attach to its
abstract devtools socket — on a stock device that is adb (and the app itself). If a user
ever has USB/wireless debugging on for unrelated reasons, a peer with shell access can
inspect and drive those tabs.
Recommendation: gate this behind phone-control being enabled / a developer flag, and
turn it off when the feature is not in use.

### R3 — Accessibility Service is the crown-jewel surface; deny-list is opt-out (Medium)
The a11y service can read and act on **any** screen. Mika's mitigations are real and
above average: a per-package **App Access** gate wrapping every screen read/action, a
"operate on my own virtual display first" model, take-over for login/CAPTCHA/payment,
an irreversible-action detector + confirmation gate, and user consent screens for
screen sharing. However the App Access model **allows any app not explicitly
blocked**. For a user who installs Mika, a banking app, and a password manager, "default
allow" means the agent can read those unless blocked.
Recommendation: prefer an **allow-list for sensitive categories**, or ship a conservative
default block-list (banking, password managers, system settings, authenticators).

### R4 — Remote relay / gateway path (Low–Medium, not fully audited)
`core/remote/RelayClient.kt`, `RelayGateway.kt`, `settings/RelaySettings.kt` let a paired
gateway hand tasks to the phone over an outbound channel. Pairing stores a device token;
relayed requests still pass through `RemoteAccess` (token required), and screen
wake/unlock needs a **separate** opt-in. I did not exhaustively audit the relay protocol
or the gateway server; treat the gateway as a trust boundary and confirm it cannot inject
tasks that bypass the confirmation gate.

### R5 — User-configured HTTP proxy is a full MITM of app traffic (Low, by design)
`core/net/NetProxy.kt`: when a proxy is set, model calls, sign-in token exchange,
update downloads and browser traffic go through it (with local/LAN hosts bypassed).
A malicious proxy sees everything in cleartext. Expected for a proxy feature; worth a
warning in the UI.

### R6 — Update trust depends on GitHub + release metadata (Low)
Signer-cert matching is the strong anchor; the SHA-256 is only checked when the release
provides a `digest` (it's nullable). Recommendation: require a digest for APK assets, or
at least document that the signer check is the sole hard guarantee.

### R7 — Third-party model is the real data boundary (Informational)
"Your data stays on your phone / no Mika server" is true — there is no backend — but
**screen text, screenshots, files and the API key are sent to whichever model provider
the user chose.** For a user doing sensitive work, that is the dominant privacy exposure.
Worth stating explicitly in the README/disclaimer.

### Minor notes
- `FileViewerScreen.HtmlView` renders HTML with JS enabled and no native JS bridge
  (`loadDataWithBaseURL(null, ...)`, no `addJavascriptInterface`) — low risk. Add
  `settings.allowFileAccess = false` / `allowContentAccess = false` for defense in depth.
- `WRITE_SECURE_SETTINGS` is declared (protected; only adb-grantable) and used for
  wireless-debugging auto-restore. Powerful but appropriately scoped.
- Boot receiver is exported without a permission but only listens for protected
  broadcasts (BOOT_COMPLETED etc.) — fine.

---

## 3. Overall assessment for a user

If you install Mika, understand what you are trusting:

1. It holds an **Accessibility Service** (can see/operate every screen) and, if you
   enable phone control, an **ADB-driven shell helper**. Both are tightly gated *by the
   app*, but they are extremely powerful by nature.
2. Your **model API key** is stored safely (Keystore, AES-GCM) and sent only to the
   endpoint you configure — over TLS unless you point at a plaintext URL (R1).
3. The **local MCP server** is well defended (token + rate-limit + private-only + no
   browser origin). It has **no TLS**, but is meant for LAN/USB.
4. The **sandbox is jailed and offline by default**; enabling network is explicit.
5. **Updates are signature-pinned** to this app's signer.
6. Your screen content goes to **your chosen LLM provider** — that is the privacy
   boundary, not the app.

No high-severity, directly-exploitable flaw was found in the audited paths. The items to
fix are R1 (cleartext scope) and R2 (WebView debugging), plus the policy default in R3.
