# AinkradPluginTemplate

A starter for an [Ainkrad](https://github.com/AhmedMElhalaby) Marketplace plugin. A plugin is an
`AinkradApp` — a small SwiftUI-based app — compiled into a macOS bundle (`mh_bundle`,
`.bundle`) that the Ainkrad host loads at runtime and hands a `HostServices` object to. Your
app never links against the host binary; it only sees the surface `AinkradAppKit` exposes
(`HostServices`, `AinkradApp`, `AinkradPluginEntryPoint`).

This repo builds `TemplatePlugin.bundle` — a working, loadable, do-nothing plugin — so you can
start from something that already builds, sideloads, and publishes correctly.

## Getting started

1. Click **"Use this template"** on GitHub (or `git clone` and repoint `origin`) to create your
   own repo from this one.
2. Rename the plugin identity throughout:
   - `MyApp` (the `AinkradApp` conformer) and `MyPluginEntryPoint` (the `@objc` entry point) in
     `Sources/Plugin/*.swift` → your own type names.
   - `myplugin` (the `AinkradAppID` / `static let id`) → your own app id.
   - `com.example.plugin.myplugin` (`CFBundleIdentifier` / `PRODUCT_BUNDLE_IDENTIFIER` in
     `Info.plist` and `project.yml`) → your own bundle id, e.g. `com.you.plugin.<id>`.
   - `TemplatePlugin` (the Xcode scheme/target name, `CFBundleName`, `CFBundleExecutable`) →
     your own product name, if you want one that isn't "TemplatePlugin".
   - `ID`, `NAME`, `ICON`, `DESC` in `scripts/release.sh` → match your `Info.plist`.
3. Replace the placeholder UI in `Sources/Plugin/PluginApp.swift` with your app.

## Required `Info.plist` keys

Every plugin bundle's `Info.plist` must declare these keys:

| Key | Value | Why |
|---|---|---|
| `CFBundlePackageType` | `BNDL` | Marks it as a bundle rather than an app/framework. |
| `CFBundleIdentifier` | `com.you.plugin.<id>` | Unique bundle id, distinct from the host's and every other plugin's. |
| `CFBundleExecutable` | your product name (e.g. `TemplatePlugin`) | **Mandatory.** The host installs a plugin by copying and renaming its bundle to `<appID>.bundle` — the bundle's directory name no longer matches its build product name. Without an explicit `CFBundleExecutable`, `Bundle` can't locate the executable inside the renamed bundle and the load fails. |
| `NSPrincipalClass` | your `@objc` entry point class (e.g. `MyPluginEntryPoint`) | How the host finds and instantiates your code after loading the bundle. |
| `AinkradAppID` | your app id (e.g. `myplugin`) | Your plugin's identity in the host's registry and the Marketplace catalog. |
| `AinkradDisplayName` | display name (e.g. `My Plugin`) | Shown in the Launcher and Marketplace UI. |
| `AinkradIconSymbol` | an SF Symbol name (e.g. `puzzlepiece.extension`) | Your app's icon. |
| `AinkradAPIVersion` | `1` | The `AinkradAppKit` API generation you built against; the host refuses to load a bundle outside its supported range. |

See `Sources/Plugin/Info.plist` for a filled-in example.

## Conformance

Your app is two Swift types:

```swift
// The app itself.
struct MyApp: AinkradApp {
    static let id = "myplugin"
    static let displayName = "My Plugin"
    static let icon = "puzzlepiece.extension"

    static func makeRootView(host: HostServices) -> AnyView { ... }
    static func makeSettingsView(host: HostServices) -> AnyView { ... }
}

// The bundle's principal class — matches `NSPrincipalClass` in Info.plist.
@objc(MyPluginEntryPoint)
final class MyPluginEntryPoint: NSObject, AinkradPluginEntryPoint {
    static func app() -> any AinkradApp.Type { MyApp.self }
}
```

The host instantiates your `NSPrincipalClass`, casts it to `AinkradPluginEntryPoint`, and calls
`app()` to get your `AinkradApp` type. From there, everything you can do is expressed through
the `HostServices` value passed into `makeRootView`/`makeSettingsView`:

- `host.theme.tokens` — the host's resolved colors. Tint your UI from these, not hardcoded
  colors, so your plugin recolors live on a host theme change (`HostTheme` is `@Observable`).
- `host.documents` — namespaced key→`Data` storage scoped to your app id. Encode your own
  `Codable` state into `Data` yourself.
- `host.secrets` — namespaced key→string secret storage (Keychain-backed in the host).
- `host.log` — structured logging under your app's subsystem.

Never reach for host internals, private APIs, or anything outside `AinkradAppKit`'s public
surface — that surface is the entire contract between your plugin and the host.

## Dependencies

`AinkradAppKit` must be linked **dynamic and `embed: false`**:

```yaml
dependencies:
  - package: AinkradAppKit
    embed: false
```

The host embeds its own single copy of `AinkradAppKit.framework`; a plugin resolves that same
copy at runtime via `@rpath` instead of embedding a second one. This matters because
`AinkradApp`/`HostServices` casts are dynamic (`as?`/protocol dispatch) — if the plugin embedded
its own copy of `AinkradAppKit`, the host and the plugin would have two distinct copies of the
same protocol's type metadata, and casts between them would silently fail.

Every *other* dependency — anything besides `AinkradAppKit` — must be **statically linked**, so
its symbols and Objective-C classes fold into your bundle's own mach-o rather than risking a
second copy loaded elsewhere in the host process.

If your plugin needs such a dependency (e.g. a UI library, a parser, anything beyond the
standard library and system frameworks), split it into two targets — a static library plus the
bundle — the way `AinkradTerminal` does for SwiftTerm:

```yaml
targets:
  # Static library: its objects (and the statically-linked runtime dependency)
  # fold into whatever links it. AinkradAppKit is dynamic and NEVER embedded.
  MyFeature:
    type: library.static
    sources: [Sources/MyFeature]
    dependencies:
      - package: SomeRuntimeDependency
      - package: AinkradAppKit
        embed: false

  MyPlugin:
    type: bundle
    sources: [Sources/MyPlugin]
    dependencies:
      - target: MyFeature
      # Linked again here too — static libraries don't propagate object code
      # by themselves; the dependency's objects still need to land in the
      # bundle's own mach-o.
      - package: SomeRuntimeDependency
      - package: AinkradAppKit
        embed: false
    settings:
      base:
        MACH_O_TYPE: mh_bundle
        OTHER_LDFLAGS: ["-lMyFeature"]
```

This template's `TemplatePlugin` target has no such dependency, so it skips the split — see
`project.yml`.

## Build & sideload

```bash
make build     # xcodegen generate + xcodebuild the Debug bundle
make sideload  # build, then copy the bundle into DevPlugins for local testing
```

`make sideload` copies `TemplatePlugin.bundle` into
`~/Library/Application Support/com.ainkrad.app/Documents/DevPlugins`, where the host picks up
locally sideloaded plugins without going through the Marketplace.

## Publish

```bash
make release V=v1.0.0
```

`scripts/release.sh`:
1. Builds the Release configuration.
2. Zips the built bundle into `dist/<id>.bundle.zip`.
3. Computes its SHA-256 and writes `dist/ainkrad-plugin.json`:
   ```json
   { "id": "myplugin", "name": "My Plugin", "icon": "puzzlepiece.extension",
     "description": "A starter Ainkrad plugin.", "apiVersion": 1, "sha256": "<sha256>" }
   ```
4. Cuts a GitHub release tagged `V` with both files attached.

The Marketplace catalog reads each listed repo's **latest** GitHub release; the release tag is
your plugin's version string. Once your repo has at least one release, ask to have it added to
the catalog's repository list so it shows up in the Marketplace.

## Signing

Plugins are unsigned/ad-hoc today — dev-mode only (`CODE_SIGNING_REQUIRED: NO` in `project.yml`).
Developer-ID signing and host-side trust verification are planned (AIN-135); until that lands,
sideloading and catalog installs both run without a code-signing check.

## Known limitations

Custom SwiftUI `EnvironmentKey`s do **not** cross the host↔plugin boundary. Your plugin's view
hierarchy and the host's are in different modules with independent environment key identities,
even if the key type's name matches — for example, the host's `paneResizesImmediately` hint
(used internally by host panes) is invisible to a plugin's views; reading it inside your
`makeRootView` returns the key's default value, not the host's actual value.

Talk to the host only through `HostServices` (`host.theme`, `host.documents`, `host.secrets`,
`host.log`). If you need a host behavior that isn't exposed there, it needs to be added to
`AinkradAppKit`, not smuggled through SwiftUI environment.

## Reference

For a full, real-world example of a plugin with a runtime dependency (SwiftTerm), the
static-library split, and a working release pipeline, see
[AinkradTerminal](https://github.com/AhmedMElhalaby/AinkradTerminal).
