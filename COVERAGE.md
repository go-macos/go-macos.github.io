# Framework coverage: what go-macos binds, and what it should bind next

Established by measurement on 2026-08-30, not from the Apple catalogue by memory.
Reproduce the two figures below before trusting them:

```sh
# frameworks each package opens
grep -rhoE '/System/Library/Frameworks/[A-Za-z]+\.framework' --include='*.go' .

# what the wider fleet opens outside this org (evidence of demand)
grep -rlE '/System/Library/Frameworks/[A-Za-z]+\.framework' --include='*.go' ~/…/github.com \
  | grep -v '/go-macos/'
```

## Where we stand

`/System/Library/Frameworks` holds **305** frameworks on macOS 26. The org's 18
packages reach **15** of them:

| Package | Frameworks |
|---|---|
| accessibility | AppKit, ApplicationServices, CoreGraphics, Foundation |
| appicon | AppKit, Foundation |
| audiotoolbox | AudioToolbox |
| avfoundation | AVFoundation, CoreMedia, CoreVideo |
| fileprogress | Foundation |
| hotkey | Carbon, ApplicationServices, CoreGraphics |
| iokit | IOKit, CoreFoundation |
| keychain | CoreFoundation (Security) |
| objc | AppKit, CoreFoundation, Foundation, Security, WebKit |
| pointer | CoreGraphics |
| screencapture | ScreenCaptureKit, CoreGraphics, CoreMedia, CoreVideo |
| statusitem | AppKit, CoreGraphics, Foundation |
| videotoolbox | VideoToolbox, CoreMedia, CoreVideo |
| virtualdisplay | CoreGraphics, Foundation |
| appbundle, brightness, launchagent, notify | none (file/shell level) |

Counting 290 unbound frameworks is not a backlog. Most are iOS-shaped, SwiftUI
shims (`_MapKit_SwiftUI`), or dead (QTKit, JavaVM, Tcl). The list below is
ranked by **a named consumer that exists today**, not by breadth.

## Tier A — a consumer is already waiting

| Framework | Consumer | Why the OS, and not pure Go |
|---|---|---|
| **ServiceManagement** | `go-macos/launchagent` | `SMAppService` is the supported way to register a login item since macOS 13. Writing a plist into `~/Library/LaunchAgents` — which is what `launchagent` does — is the legacy path. The package should offer both and prefer the modern one. |
| **LocalAuthentication** | go-pdfkit reader (Touch ID unlock) | Biometry cannot be reimplemented; it is an attestation by the Secure Enclave. |
| **Virtualization**, **vmnet** | weft (microVM cloud), the Tart VM lab | VZ boot is already proven for weft; `vmnet` is precisely the socket networking the lab depends on. Today that goes through `tart`, an external binary. |
| **Network**, **NetworkExtension** | claimward (WireGuard VPN) | A macOS VPN must be a `NEPacketTunnelProvider`; there is no user-space substitute. |
| **DiskArbitration**, **FSKit** | go-filesystems, go-volumes, go-diskimages | FSKit lands a user-space filesystem without a kext — the macOS counterpart to the Linux ioctl surface `go-fsctl` already covers. |
| **UserNotifications** | replaces the archived `tannevaled/notify` | `notify` drives `NSUserNotification`, deprecated. `UNUserNotificationCenter` is the live API. |
| **UniformTypeIdentifiers** | go-freedesktop (icontheme, thumbnail) | The macOS half of file-type resolution; today only the freedesktop half exists. |
| **IOSurface** | screencapture, xrkit | Zero-copy frame handoff. `screencapture` already pays 18 ms per 4K frame. |

## Tier B — foreign judges, per the fleet's own doctrine

The fleet's rule for conformance is a *foreign judge*: `pdftoppm` for PDF, MRI for
Ruby. These frameworks are worth binding **as test oracles**, never as runtime
dependencies of a pure-Go library:

- **CoreText** — judges go-opentype's shaping and metrics.
- **ImageIO** — judges go-images' decoders, and gives a second opinion beside `qlmanage`.
- **PDFKit** — a third judge for go-pdfkit, independent of poppler.
- **Accelerate / vecLib** — the reference to *beat* in go-simd and go-ndarray benchmarks.

A judge binding belongs behind a build tag so it can never enter a release build.

## Tier C — opportunistic, no consumer yet

OSLog (the unified log store; note `log show` returns nothing useful on this
machine), CoreWLAN and SystemConfiguration (network state), Metal and MetalKit
(GPU paths for go-gfx and xrkit), QuickLookThumbnailing, InputMethodKit.

## Explicit non-goals

SwiftUI and every `_X_SwiftUI` shim (no stable C ABI); CoreML, Vision, Speech
(outside the fleet's subject); anything iOS-only.

## The go-widgets axis: what "native" should mean

`go-widgets` already imports `go-macos/objc` in 20 places, yet
`go-widgets/window` still opens CoreFoundation and CoreGraphics itself and
`go-widgets/tray` hand-rolls its own class lookup — which is why the tray gets a
nil `NSApplication` and exits in silence. **Removing that duplication is worth
more than any new binding.** One place that names `NSApplication` is one place
that can guarantee AppKit is loaded.

Beyond that, "native toolkit" is three different ambitions and they should not be
confused:

**N1 — a native shell around drawn content.** NSMenu for the menu bar,
NSOpenPanel/NSSavePanel, NSToolbar, sheets, NSPrintOperation. Measured today:
go-widgets uses NSPasteboard (clipboard) and nothing else from this list. A file
picker is the clearest gap — every application needs one and no one wants a drawn
imitation of it.

**N2 — native input.** Measured today: no `NSTextInputClient`, no `markedText`,
no `interpretKeyEvents` anywhere in go-widgets. The toolkit therefore cannot
accept composed input at all — no Chinese, Japanese or Korean, no dead keys, no
emoji palette. This is not polish; it is a class of user that cannot type. It is
the single strongest argument for reaching into AppKit, and it is invisible to
anyone testing in English.

Accessibility is **already** provided (`window/internal/cocoa/a11y_darwin.go`),
so it does not belong on this list — note that this is the mirror image of
`go-macos/accessibility`, which *reads* other applications' trees.

**N3 — native widgets** (NSButton, NSTextField as embedded views). This breaks
the toolkit's cross-platform pixel identity and should stay the exception,
justified only where the OS widget carries behaviour that cannot be
reimplemented: secure text entry, and the system color/font panels.

The order is N2, then N1, then N3 only where forced.
