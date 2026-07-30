# macOS: Linux-like terminal selection and global middle-click paste

This recipe configures a Mac so that:

- selecting text with the mouse in iTerm2 immediately copies it to the normal macOS clipboard;
- iTerm2 handles terminal scripts and terminal-oriented URL schemes;
- iTerm2 uses the active Apple Terminal profile's colors as a persistent dark palette;
- pressing the physical middle mouse button emits `Command-V` in the frontmost app;
- the mouse mapping starts automatically at login.

It was verified on macOS 26.5.2 with iTerm2 3.6.11 and Hammerspoon 1.1.1. The commands intentionally avoid depending on those exact versions.

## Important behavior and permission notes

- The middle button's original behavior is replaced globally. For example, it will no longer open a browser link in a new tab.
- Middle-click paste works wherever `Command-V` works. Secure-input fields and apps with no normal Paste command may ignore it.
- Hammerspoon needs macOS Accessibility permission because it observes and replaces global mouse events. A human must approve that permission in System Settings.
- Do not terminate an existing iTerm2 process just to reload its preferences if it contains active sessions. Restart it only when safe.

## 1. Install the applications

Install Homebrew first if it is not already installed. Then run:

```sh
brew install --cask iterm2 hammerspoon
brew install duti
```

`duti` is used to register and inspect default application handlers.

## 2. Enable copy-on-selection in iTerm2

Run this while iTerm2 is closed, or restart iTerm2 later when doing so will not interrupt active sessions:

```sh
defaults write com.googlecode.iterm2 CopySelection -bool true
defaults read com.googlecode.iterm2 CopySelection
```

The verification command should print `1`.

The same option is available in **iTerm2 → Settings → General → Selection → Copy to clipboard on selection**.

## 3. Make iTerm2 the default terminal handler

Open iTerm2 and select:

**iTerm2 → Make iTerm2 Default Term**

That registers iTerm2 for `.command`, `.tool`, `.zsh`, `.csh`, and `.pl` files.

Also register terminal-oriented URL schemes:

```sh
for url_scheme in ssh telnet x-man-page; do
    duti -s com.googlecode.iterm2 "$url_scheme"
done
```

### Verify file handlers

```sh
for extension_name in command tool zsh csh pl; do
    printf '.%-7s ' "$extension_name"
    duti -x "$extension_name" | sed -n '1p'
done
```

Every row should report `iTerm` or `iTerm2`.

### Verify URL handlers

```sh
swift -e 'import AppKit; import Foundation
for scheme in ["ssh", "telnet", "x-man-page"] {
    let url = URL(string: "\(scheme)://example.invalid")!
    let app = NSWorkspace.shared.urlForApplication(toOpen: url)?.lastPathComponent ?? "none"
    print("\(scheme): \(app)")
}'
```

Every row should report `iTerm.app`.

### macOS 26 fallback if the iTerm2 menu does not register file types

Use the current Launch Services API:

```sh
swift -e 'import AppKit; import UniformTypeIdentifiers; import Foundation
let appURL = URL(fileURLWithPath: "/Applications/iTerm.app")
let extensions = ["command", "tool", "zsh", "csh", "pl"]
let group = DispatchGroup()
for ext in extensions {
    guard let type = UTType(filenameExtension: ext) else {
        print("No content type for .\(ext)")
        continue
    }
    group.enter()
    NSWorkspace.shared.setDefaultApplication(at: appURL, toOpen: type) { error in
        if let error {
            print(".\(ext) [\(type.identifier)]: ERROR \(error)")
        } else {
            print(".\(ext) [\(type.identifier)]: set to iTerm2")
        }
        group.leave()
    }
}
while group.wait(timeout: .now() + 0.1) == .timedOut {
    RunLoop.current.run(until: Date(timeIntervalSinceNow: 0.1))
}'
```

macOS may display a consent prompt. Approve it, wait for the five result lines, and then repeat the handler verification.

## 4. Copy Apple Terminal's active colors into iTerm2

This optional step copies the active Apple Terminal profile's background, foreground, bold, cursor, and selection colors into iTerm2's Default profile. It also disables iTerm2's adaptive Light palette so the terminal remains dark when macOS itself uses Light appearance. Existing iTerm ANSI colors and non-color settings are preserved.

Save the following as `apply-terminal-profile-to-iterm.swift`:

```swift
import AppKit
import Foundation

enum ConversionError: Error, CustomStringConvertible {
    case missing(String)
    case invalid(String)

    var description: String {
        switch self {
        case .missing(let value): return "Missing \(value)"
        case .invalid(let value): return "Invalid \(value)"
        }
    }
}

let fileManager = FileManager.default
let terminalPreferencesURL = fileManager.homeDirectoryForCurrentUser
    .appendingPathComponent("Library/Preferences/com.apple.Terminal.plist")
let iTermPreferencesURL = fileManager.homeDirectoryForCurrentUser
    .appendingPathComponent("Library/Preferences/com.googlecode.iterm2.plist")

guard CommandLine.arguments.count == 2 else {
    throw ConversionError.missing("output plist argument")
}
let outputURL = URL(fileURLWithPath: CommandLine.arguments[1])

func decodedColor(from value: Any?, named name: String) throws -> NSColor {
    guard let archived = value as? Data else {
        throw ConversionError.missing("Terminal color \(name)")
    }
    guard let color = try NSKeyedUnarchiver.unarchivedObject(
        ofClass: NSColor.self,
        from: archived
    ),
          let rgb = color.usingColorSpace(.sRGB) else {
        throw ConversionError.invalid("Terminal color \(name)")
    }
    return rgb
}

func iTermColor(_ color: NSColor, forceOpaque: Bool = false) -> [String: Any] {
    [
        "Alpha Component": forceOpaque ? 1.0 : color.alphaComponent,
        "Red Component": color.redComponent,
        "Green Component": color.greenComponent,
        "Blue Component": color.blueComponent,
        "Color Space": "sRGB",
    ]
}

let terminalData = try Data(contentsOf: terminalPreferencesURL)
guard let terminalRoot = try PropertyListSerialization.propertyList(
    from: terminalData,
    options: [],
    format: nil
) as? [String: Any] else {
    throw ConversionError.invalid("Terminal preferences")
}

guard let activeTerminalProfile = terminalRoot["Default Window Settings"] as? String else {
    throw ConversionError.missing("Terminal default profile name")
}
guard let terminalProfiles = terminalRoot["Window Settings"] as? [String: Any],
      let terminalProfile = terminalProfiles[activeTerminalProfile] as? [String: Any] else {
    throw ConversionError.missing("Terminal profile \(activeTerminalProfile)")
}

let mappings = [
    (terminal: "BackgroundColor", iTerm: "Background Color"),
    (terminal: "TextColor", iTerm: "Foreground Color"),
    (terminal: "TextBoldColor", iTerm: "Bold Color"),
    (terminal: "CursorColor", iTerm: "Cursor Color"),
    (terminal: "SelectionColor", iTerm: "Selection Color"),
]

var convertedColors: [String: NSColor] = [:]
for mapping in mappings {
    convertedColors[mapping.iTerm] = try decodedColor(
        from: terminalProfile[mapping.terminal],
        named: mapping.terminal
    )
}

let iTermData = try Data(contentsOf: iTermPreferencesURL)
guard var iTermPreferences = try PropertyListSerialization.propertyList(
    from: iTermData,
    options: [],
    format: nil
) as? [String: Any] else {
    throw ConversionError.invalid("iTerm preferences")
}
guard let defaultGuid = iTermPreferences["Default Bookmark Guid"] as? String else {
    throw ConversionError.missing("iTerm default profile GUID")
}
guard var iTermProfiles = iTermPreferences["New Bookmarks"] as? [[String: Any]],
      let defaultIndex = iTermProfiles.firstIndex(where: { $0["Guid"] as? String == defaultGuid }) else {
    throw ConversionError.missing("iTerm default profile")
}

var iTermProfile = iTermProfiles[defaultIndex]
iTermProfile["Use Separate Colors for Light and Dark Mode"] = false

for (key, color) in convertedColors {
    let value = iTermColor(color, forceOpaque: key == "Background Color")
    iTermProfile[key] = value
    iTermProfile["\(key) (Dark)"] = value
    iTermProfile["\(key) (Light)"] = value
}

if let background = convertedColors["Background Color"] {
    iTermProfile["Transparency"] = 1.0 - background.alphaComponent
    iTermProfile["Blur"] = false
}

iTermProfiles[defaultIndex] = iTermProfile
iTermPreferences["New Bookmarks"] = iTermProfiles
let convertedPreferences = try PropertyListSerialization.data(
    fromPropertyList: iTermPreferences,
    format: .xml,
    options: 0
)
try convertedPreferences.write(to: outputURL, options: .atomic)

print("Converted Terminal profile '\(activeTerminalProfile)' into iTerm profile '\(iTermProfile["Name"] as? String ?? "Default")'.")
print("Forced a dark-only palette and preserved iTerm ANSI colors and non-color settings.")
print("Wrote converted preferences to \(outputURL.path).")
```

Ensure no foreground job is running in iTerm2, close iTerm2, and keep its background helper stopped while applying the preferences. Then run:

```sh
backup_dir="$HOME/Library/Application Support/iTerm2/Preference Backups"
mkdir -p "$backup_dir"

backup_file="$backup_dir/com.googlecode.iterm2-before-color-conversion-$(date +%Y%m%d-%H%M%S).plist"
converted_file="$backup_dir/com.googlecode.iterm2-colors-converted-$(date +%Y%m%d-%H%M%S).plist"

defaults export com.googlecode.iterm2 "$backup_file"
swiftc -typecheck apply-terminal-profile-to-iterm.swift
swift apply-terminal-profile-to-iterm.swift "$converted_file"
plutil -lint "$converted_file"
defaults import com.googlecode.iterm2 "$converted_file"
open -a iTerm
```

For Terminal's built-in **Pro** profile, the expected iTerm values include:

- `Use Separate Colors for Light and Dark Mode`: `false`
- background red component: `0`
- foreground red component: approximately `0.960185`
- transparency: `0.15`

The backup plist can be restored with `defaults import com.googlecode.iterm2 <backup-file>` while iTerm2 is closed.

## 5. Configure global middle-click paste

Create `~/.hammerspoon/init.lua` with the following content. If the file already exists, merge this with the existing configuration instead of replacing it.

```lua
-- Make the physical middle mouse button paste from the macOS clipboard.
-- Keep a global reference so the event tap is not garbage-collected.
local middleButton = 2
local mouseEvent = hs.eventtap.event

-- Enable the local `hs` command for health checks and configuration reloads.
require("hs.ipc")

middleClickPaste = hs.eventtap.new({
    mouseEvent.types.otherMouseDown,
    mouseEvent.types.otherMouseUp,
    mouseEvent.types.otherMouseDragged,
}, function(event)
    local button = event:getProperty(mouseEvent.properties.mouseEventButtonNumber)
    if button ~= middleButton then
        return false
    end

    if event:getType() == mouseEvent.types.otherMouseUp then
        -- Run after this callback so the consumed mouse-up finishes first.
        hs.timer.doAfter(0, function()
            hs.eventtap.keyStroke({ "cmd" }, "v", 0)
        end)
    end

    -- Suppress the original middle-button event in the frontmost app.
    return true
end)

middleClickPaste:start()
hs.autoLaunch(true)
```

Launch Hammerspoon:

```sh
open -a Hammerspoon
```

On first launch:

1. Open **Hammerspoon → Settings/Preferences**.
2. Select **Enable Accessibility**.
3. In **System Settings → Privacy & Security → Accessibility**, enable Hammerspoon.
4. Quit and relaunch Hammerspoon if its permission warning does not clear immediately.

If Homebrew manages Hammerspoon updates, its separate automatic updater can be disabled to avoid duplicate prompts.

## 6. Verify the mouse mapping

Run:

```sh
HS_BIN="$(command -v hs)"
"$HS_BIN" -c 'print(
  "middle-click paste " .. (middleClickPaste:isEnabled() and "enabled" or "disabled")
  .. "; launch-at-login " .. (hs.autoLaunch() and "enabled" or "disabled")
  .. "; accessibility " .. (hs.accessibilityState() and "granted" or "missing")
)'
```

Expected result:

```text
middle-click paste enabled; launch-at-login enabled; accessibility granted
```

Finally, select a harmless piece of text in iTerm2 and middle-click in a temporary editable document. The selected text should paste.

## Rollback

Disable iTerm2 copy-on-selection:

```sh
defaults write com.googlecode.iterm2 CopySelection -bool false
```

Restore the pre-conversion iTerm profile by closing iTerm2 and importing the saved backup plist:

```sh
defaults import com.googlecode.iterm2 "/path/to/com.googlecode.iterm2-before-color-conversion-TIMESTAMP.plist"
open -a iTerm
```

Disable the global mouse mapping by quitting Hammerspoon, turning off **Launch Hammerspoon at login**, and removing or commenting out the `middleClickPaste` block in `~/.hammerspoon/init.lua`.

To restore Apple Terminal's file associations, hold **Option** while opening the iTerm2 application menu, then select **Make Terminal Default Term**.

To revoke the global-input permission, turn Hammerspoon off in **System Settings → Privacy & Security → Accessibility**.

## References

- [iTerm2 General preferences: Selection](https://iterm2.com/documentation-preferences-general.html)
- [iTerm2 menu item: Make iTerm2 Default Term](https://stage.iterm2.com/documentation-menu-items.html)
- [Apple NSWorkspace default-application API](https://developer.apple.com/documentation/appkit/nsworkspace/setdefaultapplication%28at%3Atoopen%3Acompletion%3A%29)
- [Hammerspoon event taps](https://www.hammerspoon.org/docs/hs.eventtap.html)
- [Hammerspoon launch-at-login API](https://www.hammerspoon.org/docs/hs#autoLaunch)
