# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

KeyboardShortcuts is a Swift Package for macOS that provides user-customizable global keyboard shortcuts. It's fully sandboxed and Mac App Store compatible.

**Key capabilities:**
- Global keyboard shortcut registration and listening
- SwiftUI (`Recorder`) and Cocoa (`RecorderCocoa`) UI components for recording shortcuts
- Automatic storage in UserDefaults
- Validation against system shortcuts and menu bar conflicts
- Works even when NSMenu is open (for menu bar apps)

## Build & Test Commands

### Build
```bash
swift build
```

### Run Tests
```bash
swift test
```

### Run Specific Test
```bash
swift test --filter <TestName>
```

### Lint
```bash
swiftlint
```

The project uses an extensive SwiftLint configuration (see `.swiftlint.yml`) with custom rules including:
- Use `CGRect`/`CGSize`/`CGPoint` instead of NS equivalents
- Use `Double` instead of `CGFloat`
- SwiftUI properties should be private
- Classes must be marked as `final`

### Example App
Open and run the example app in Xcode:
```bash
open Example/KeyboardShortcutsExample.xcodeproj
```

## Architecture

### Core Components

**KeyboardShortcuts (main enum)** - Central API with static methods
- Handler registration: `onKeyDown()`, `onKeyUp()`, `events()` (async stream)
- Shortcut management: `getShortcut()`, `setShortcut()`, `reset()`, `enable()`, `disable()`
- Storage handling via UserDefaults with prefix `KeyboardShortcuts_`

**Name** - Strongly-typed shortcut identifiers
- Created via extensions: `extension KeyboardShortcuts.Name { static let myShortcut = Self("myShortcut") }`
- Can specify default shortcuts (though discouraged for public apps)
- Stored in UserDefaults when assigned

**Shortcut** - Represents key + modifiers combination
- Contains `carbonKeyCode` and `carbonModifiers` (low-level Carbon API)
- Provides `key` (strongly-typed) and `modifiers` (NSEvent.ModifierFlags)
- Can check if taken by system or main menu
- Handles sandboxing restrictions (macOS 15.0-15.1: Option key requires Command/Control)

**Recorder (SwiftUI)** - UI component for recording shortcuts
- Wraps `RecorderCocoa` in NSViewRepresentable
- Automatically validates and stores shortcuts
- Shows alerts for conflicts with system or menu shortcuts

**RecorderCocoa (AppKit)** - Native NSView recorder component
- Used directly in Cocoa apps or wrapped by SwiftUI Recorder
- Handles keyboard events, visual feedback, and validation

**CarbonKeyboardShortcuts** - Low-level Carbon API wrapper
- Registers/unregisters EventHotKeyRef with Carbon event handler
- Switches between hot key events and raw key events when menu is open
- Uses RunLoopLocalEventMonitor on macOS 14+ for menu tracking mode

### Event Flow

1. User defines `KeyboardShortcuts.Name` extension
2. UI presents `Recorder` for user to record shortcut
3. Shortcut is JSON-encoded and stored in UserDefaults
4. App registers handler via `onKeyDown()`/`onKeyUp()` or `events()`
5. `CarbonKeyboardShortcuts.register()` creates EventHotKeyRef
6. When shortcut pressed, Carbon event handler calls registered closures
7. Handlers track both legacy callbacks and async stream continuations

### Key Files

- `KeyboardShortcuts.swift` - Main API and handler management
- `Name.swift` - Shortcut name type definition
- `Shortcut.swift` - Shortcut data structure and validation
- `Recorder.swift` - SwiftUI recording interface
- `RecorderCocoa.swift` - AppKit recording interface  
- `CarbonKeyboardShortcuts.swift` - Carbon API integration
- `Key.swift` - Enum of keyboard keys with Carbon key codes
- `NSMenuItem++.swift` - Extension to set menu item shortcuts
- `Utilities.swift` - Helper extensions and utilities
- `ViewModifiers.swift` - SwiftUI view modifiers

### Testing

Tests are located in `Tests/KeyboardShortcutsTests/`:
- `KeyboardShortcutsTests.swift` - Core functionality tests
- `RecorderLayoutTests.swift` - UI layout tests
- `Utilities.swift` - Test helpers

### Localization

Localizations are in `Sources/KeyboardShortcuts/Localization/` with `.lproj` directories. Add new localizations by copying `en.lproj/Localizable.strings`.

## Important Constraints

**macOS 15.0-15.1 Sandboxing Issue:** For sandboxed apps, shortcuts with Option key must also include Command or Control (no Option+Shift only). The `Shortcut.isDisallowed` property checks this.

**Carbon APIs:** Uses deprecated Carbon APIs for global shortcut registration - this is currently the only way to register global shortcuts. Apple has not provided modern replacements.

**UserDefaults Storage:** Shortcuts are automatically stored with the prefix `KeyboardShortcuts_` + the name's raw value.

## Development Patterns

**Dynamic shortcut names:** Names don't have to be static. Create `KeyboardShortcuts.Name` instances dynamically and store them for user-defined actions.

**Handler cleanup:** Use `.removeHandler(for:)` or `.removeAllHandlers()` to prevent duplicate handlers. Async stream `events()` automatically cleans up on sequence termination.

**Menu integration:** Use `NSMenuItem.setShortcut(_:)` extension to sync menu items with recorded shortcuts.

**Concurrent handlers:** Multiple handlers can listen to the same shortcut name. All are invoked when triggered.
