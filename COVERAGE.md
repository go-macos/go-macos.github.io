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

Re-measured **2026-09-01** on macOS 26.6.2, with the commands above.
`/System/Library/Frameworks` holds **305** frameworks. The org's **26** packages
reach **22** of them, plus **2** private ones.

Read the table for what it is: the frameworks each package opens **by path**,
which is what the grep above finds. A package that goes through
`go-macos/objc` reaches AppKit and Foundation through it without naming a path
of its own — `appicon`, `fileprogress`, `keychain` and `notify` all do, and so
appear empty here.

| Package | Frameworks |
|---|---|
| accessibility | ApplicationServices, CoreGraphics |
| appkit | CoreGraphics |
| audiotoolbox | AudioToolbox |
| avfoundation | AVFoundation, CoreMedia, CoreVideo |
| **coreml** | **CoreML**, CoreVideo |
| diskarbitration | DiskArbitration |
| hotkey | ApplicationServices, Carbon, CoreFoundation, CoreGraphics |
| iokit | CoreFoundation, IOKit |
| localauthentication | LocalAuthentication |
| **metal** | **Metal** |
| **multitouch** | CoreFoundation, *MultitouchSupport* (private) |
| objc | AppKit, CoreFoundation, CoreGraphics, Foundation, Security, WebKit |
| pointer | CoreGraphics |
| screencapture | CoreGraphics, CoreMedia, CoreVideo, ScreenCaptureKit |
| servicemanagement | ServiceManagement |
| statusitem | CoreGraphics |
| usernotifications | CoreServices, UserNotifications |
| videotoolbox | CoreMedia, CoreVideo, VideoToolbox |
| virtualdisplay | CoreGraphics |
| brightness | *DisplayServices* (private) |
| appbundle, appicon, fileprogress, keychain, launchagent, notify | none by path |

Counting 283 unbound frameworks is not a backlog. Most are iOS-shaped, SwiftUI
shims (`_MapKit_SwiftUI`), or dead (QTKit, JavaVM, Tcl). The list below is
ranked by **a named consumer that exists today**, not by breadth.
## Tier A — a consumer is already waiting

| Framework | Consumer | Why the OS, and not pure Go |
|---|---|---|
| **ServiceManagement** | `go-macos/launchagent` | `SMAppService` is the supported way to register a login item since macOS 13. Writing a plist into `~/Library/LaunchAgents` — which is what `launchagent` does — is the legacy path. The package should offer both and prefer the modern one. |
| **LocalAuthentication** | go-pdfkit reader (Touch ID unlock) | Biometry cannot be reimplemented; it is an attestation by the Secure Enclave. |
| **Virtualization**, **vmnet** | weft (microVM cloud), the Tart VM lab | VZ boot is already proven for weft; `vmnet` is precisely the socket networking the lab depends on. Today that goes through `tart`, an external binary. |
| **Network**, **NetworkExtension** | claimward (WireGuard VPN) | A macOS VPN must be a `NEPacketTunnelProvider`; there is no user-space substitute. |
| ~~**DiskArbitration**~~ **DONE v0.1.0** | ⚠ **no consumer wired yet** — see below | Enumerating block devices and mounting or unmounting them is an OS service. It has nothing to do with decoding a filesystem format, and the binding belongs HERE, exactly as the Linux ioctl surface lives in its own org (`go-fsctl`) rather than inside `go-filesystems`. |

**Correction, 2026-08-31.** This row first named `go-diskimages` as the consumer,
"which goes through `hdiutil`/`diskutil` today". That was wrong, and it was
asserted from a `grep` rather than read: **`go-diskimages` shells out to
nothing.** Every `hdiutil` mention in it is a *comment* explaining what Apple's
tool produces so the pure-Go writer reproduces its bytes, and the library's only
`exec.Command` is a cobra subcommand that happens to be named `exec`.

A sweep of the whole fleet found exactly one place that really mounts a
filesystem: `openweft/weft-firstboot/datasource/disk_bsd.go` — and it is
`openbsd || freebsd || netbsd`, never darwin. So DiskArbitration has **no
consumer in this fleet today**. The binding is still right (observing disks,
unmounting and ejecting are OS services no pure-Go library can provide), but it
was written ahead of its caller, which this org's own doctrine says not to do:
name a consumer before binding.

**FSKit is deliberately NOT listed above.** Publishing a user-space filesystem
would let the 18 pure-Go drivers in `go-filesystems` actually be mounted, which
is the prize — but two things gate it before any Objective-C is written:

1. An FSKit module is a signed app extension carrying
   `com.apple.developer.fskit.fsmodule`, and that entitlement needs a
   provisioning profile from Apple. That is a distribution wall, not a coding
   problem, and no amount of purego moves it.
2. `go-filesystems/interface`.`Filesystem` is **path-based**: `ReadFile` returns
   a whole file. A mount needs handle-based reads at an offset, or a 4 KB read
   of a 4 GB file reads 4 GB. That gap is real on every operating system, and
   closing it is work in the OS-INDEPENDENT org, not here.

A pure-Go NFS or WebDAV server in front of `Filesystem` would mount on macOS
*and* Linux *and* Windows with no OS-specific code at all, needs no entitlement,
and would prove the whole idea first. FSKit is then a native-polish option, not
the way in. Nothing of the sort exists in the fleet today.
| **UserNotifications** | replaces the archived `tannevaled/notify` | `notify` drives `NSUserNotification`, deprecated. `UNUserNotificationCenter` is the live API. |
| **UniformTypeIdentifiers** | go-freedesktop (icontheme, thumbnail) | The macOS half of file-type resolution; today only the freedesktop half exists. |
| **IOSurface** | screencapture, xrkit | Zero-copy frame handoff. `screencapture` already pays 18 ms per 4K frame. |

### Landed since this census

`go-macos/servicemanagement` v0.1.0, `go-macos/usernotifications`,
`go-macos/localauthentication`, `go-macos/diskarbitration` v0.1.0,
`go-macos/multitouch` v0.1.0, and — 2026-09-01 — `go-macos/metal` v0.1.0 and
`go-macos/coreml` v0.1.0.

#### metal and coreml: the argument was CPU TIME, not speed

Both were written for one consumer, `go-xrkit/player -3d`, which turns an
ordinary flat film into 3D as it plays. Measured on an M4 Max:

| | per frame | processor time per frame |
|---|---|---|
| 4K image pipeline, 16 cores | 65 ms | 82 ms |
| the same on the GPU | 4 ms | **0.16 ms** |
| depth network, CPU only | 46 ms | 114 ms |
| depth network, GPU | **13 ms** | 5.1 ms |
| depth network, Neural Engine | 23 ms | **0.4 ms** |

The Neural Engine is NOT the fastest of the three — the GPU is, by nearly half.
It is the one that leaves the machine alone, and on a laptop that is also
drawing a browser and syncing files that is the number a person feels. So
`coreml.Open` makes the caller choose rather than hiding a default.

Four things these two found by being wrong first, worth knowing before the next
binding:

- **`MTLSize` is 24 bytes, and arm64 passes a composite that large
  INDIRECTLY** — the caller leaves it in memory and hands over a pointer. Get
  it wrong and the dispatch covers the wrong range in silence.
- **An `.mlpackage` is not what Core ML runs.** `compileModelAtURL:` needs no
  Xcode, but it writes into a temporary directory macOS empties whenever it
  likes: MOVE the result or repay the seconds on every start.
- **The Neural Engine answers in IEEE binary16**, which Go does not have, and
  the naive expansion reads SUBNORMALS as zero — which is exactly where a depth
  model keeps its far detail.
- **CoreVideo pads rows.** 1088 bytes for 518 pixels, measured. Read as width
  times pixel size, the image shears a little more on every row and still looks
  entirely plausible.

Both are checked against an independent implementation rather than against
themselves: the GPU synthesis and the portable one in `go-images/depth` agree
on **0 bytes out of 86 999 040** for a real photograph and a real network's
depth map.

`diskarbitration` is worth reading before writing another CoreFoundation
binding: it reaches 100% coverage of the BINDINGS by making the bound C entry
points themselves the seams, so `DASessionCreate` can be made to answer NULL
without a broken machine. Four defects it found by measuring rather than by
reading the headers:

- `CFNumberGetValue` returns **false** for a lossy conversion, so trusting the
  boolean silently discards a double-typed number.
- A refusal does not carry a `kDAReturn` constant: a busy unmount answers
  `0x0000C010`, which is `unix_err(EBUSY)`.
- An attached disk image is marked `DADeviceModel = "Disk Image"` on macOS 26,
  not by the historical `DADeviceProtocol` spelling most code checks.
- `CFStringGetLength` on a non-CFString **segfaults**. Type-check dictionary
  keys before reading them.

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
machine), CoreWLAN and SystemConfiguration (network state), MetalKit,
QuickLookThumbnailing, InputMethodKit.

**Metal has left this tier.** It was listed here as opportunistic, "GPU paths
for go-gfx and xrkit" — a capability with no caller. What moved it was a
MEASUREMENT rather than an intention: see below.

## Explicit non-goals

SwiftUI and every `_X_SwiftUI` shim (no stable C ABI); Vision, Speech; anything
iOS-only.

**Correction, 2026-09-01: Core ML was on this list and should not have been.**
It was ruled "outside the fleet's subject", which was a judgement about subject
matter made without a subject in front of it. A consumer then appeared —
turning an ordinary flat film into 3D for the glasses needs a depth map, and a
single-image depth network is the only thing that produces a good one — and the
question stopped being philosophical. The lesson is the one this document
already applies in the other direction: rank by a named consumer, and do not
rule a framework OUT by breadth either.

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
