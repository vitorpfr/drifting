# Saved Stations Limit Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Limit free users to 10 saved stations, show a tappable "Save limit reached" toast on the 11th attempt, and update the Plus upgrade card to mention unlimited saves.

**Architecture:** Two static pure functions on `AppState` handle the testable logic — `canSave(isPlusUnlocked:isGrandfathered:favoritesCount:)` and `detectAndWriteGrandfatherFlag(favoritesFileExists:)`. The grandfathering check runs at the top of `bootstrap()` before any save can occur. `favoriteStation()` gates on `canSave`. The toast follows the existing `stationNotFoundToast` pattern in `MainView`.

**Tech Stack:** Swift, SwiftUI, UserDefaults, XCTest

---

## File Map

- **Modify:** `drifting/drifting/App/AppState.swift`
  - Add `@Published var saveLimitReachedToast: Bool = false`
  - Add `var canSave: Bool` computed property
  - Add `static func canSave(isPlusUnlocked:isGrandfathered:favoritesCount:) -> Bool`
  - Add `static func detectAndWriteGrandfatherFlag(favoritesFileExists:)`
  - Update `favoriteStation()` to check `canSave`
  - Call `detectAndWriteGrandfatherFlag` as first line of `bootstrap()`
- **Modify:** `drifting/drifting/UI/MainView.swift`
  - Add `@State private var showSaveLimitToast = false`
  - Add toast view in ZStack
  - Add `onChange(of: appState.saveLimitReachedToast)` handler
- **Modify:** `drifting/drifting/UI/SettingsView.swift`
  - Add `Label("Unlimited saved stations", systemImage: "bookmark")` as first item in the Plus features list
- **Create:** `drifting/driftingTests/SaveLimitTests.swift`

---

### Task 1: Create feature branch

- [ ] **Step 1: Create and switch to branch**

```bash
git checkout -b feature/saved-stations-limit
```

Expected: `Switched to a new branch 'feature/saved-stations-limit'`

---

### Task 2: Write failing tests

- [ ] **Step 1: Create test file**

Create `drifting/driftingTests/SaveLimitTests.swift` with this content:

```swift
import XCTest
@testable import drifting

final class SaveLimitTests: XCTestCase {
    private let key = "hasGrandfatheredUnlimitedSaves"

    override func setUp() {
        UserDefaults.standard.removeObject(forKey: key)
    }

    override func tearDown() {
        UserDefaults.standard.removeObject(forKey: key)
    }

    // MARK: - canSave

    func test_canSave_freeUserWith9Favorites_allowed() {
        XCTAssertTrue(AppState.canSave(isPlusUnlocked: false, isGrandfathered: false, favoritesCount: 9))
    }

    func test_canSave_freeUserWith10Favorites_blocked() {
        XCTAssertFalse(AppState.canSave(isPlusUnlocked: false, isGrandfathered: false, favoritesCount: 10))
    }

    func test_canSave_plusUserWith10Favorites_allowed() {
        XCTAssertTrue(AppState.canSave(isPlusUnlocked: true, isGrandfathered: false, favoritesCount: 10))
    }

    func test_canSave_grandfatheredUserWith10Favorites_allowed() {
        XCTAssertTrue(AppState.canSave(isPlusUnlocked: false, isGrandfathered: true, favoritesCount: 10))
    }

    // MARK: - detectAndWriteGrandfatherFlag

    func test_grandfathering_existingFile_writesTrue() {
        AppState.detectAndWriteGrandfatherFlag(favoritesFileExists: true)
        XCTAssertTrue(UserDefaults.standard.bool(forKey: key))
    }

    func test_grandfathering_noFile_writesFalse() {
        AppState.detectAndWriteGrandfatherFlag(favoritesFileExists: false)
        XCTAssertFalse(UserDefaults.standard.bool(forKey: key))
    }

    func test_grandfathering_flagAlreadyPresent_notOverwritten() {
        UserDefaults.standard.set(true, forKey: key)
        AppState.detectAndWriteGrandfatherFlag(favoritesFileExists: false)
        XCTAssertTrue(UserDefaults.standard.bool(forKey: key))
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
xcodebuild test \
  -scheme drifting \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' \
  -only-testing:driftingTests/SaveLimitTests 2>&1 | tail -20
```

Expected: compile error — `AppState.canSave` and `AppState.detectAndWriteGrandfatherFlag` do not exist yet.

---

### Task 3: Add static methods to AppState

- [ ] **Step 1: Add the two static methods to `AppState.swift`**

Find the `// MARK: - Private` comment at line 441 of `drifting/drifting/App/AppState.swift`. Add before it:

```swift
    // MARK: - Save limit

    static func canSave(isPlusUnlocked: Bool, isGrandfathered: Bool, favoritesCount: Int) -> Bool {
        isPlusUnlocked || isGrandfathered || favoritesCount < 10
    }

    static func detectAndWriteGrandfatherFlag(favoritesFileExists: Bool) {
        guard UserDefaults.standard.object(forKey: "hasGrandfatheredUnlimitedSaves") == nil else { return }
        UserDefaults.standard.set(favoritesFileExists, forKey: "hasGrandfatheredUnlimitedSaves")
    }

```

- [ ] **Step 2: Run tests to verify they pass**

```bash
xcodebuild test \
  -scheme drifting \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' \
  -only-testing:driftingTests/SaveLimitTests 2>&1 | tail -20
```

Expected: `** TEST SUCCEEDED **` — all 7 tests pass.

- [ ] **Step 3: Commit**

```bash
git add drifting/driftingTests/SaveLimitTests.swift drifting/drifting/App/AppState.swift
git commit -m "feat: add canSave and grandfathering detection logic with tests"
```

---

### Task 4: Wire save limit into AppState instance

- [ ] **Step 1: Add `saveLimitReachedToast` published var**

In `drifting/drifting/App/AppState.swift`, find:

```swift
    @Published var stationNotFoundToast: Bool = false
```

Add immediately after it:

```swift
    @Published var saveLimitReachedToast: Bool = false
```

- [ ] **Step 2: Add `canSave` computed property**

In `drifting/drifting/App/AppState.swift`, find the `// MARK: - Save limit` block you added in Task 3. Add a computed property before the static methods:

```swift
    var canSave: Bool {
        AppState.canSave(
            isPlusUnlocked: store.isPlusUnlocked,
            isGrandfathered: UserDefaults.standard.bool(forKey: "hasGrandfatheredUnlimitedSaves"),
            favoritesCount: favorites.count
        )
    }
```

- [ ] **Step 3: Call `detectAndWriteGrandfatherFlag` at top of `bootstrap()`**

In `drifting/drifting/App/AppState.swift`, find:

```swift
    func bootstrap() async {
        haptics.prepare()
```

Replace with:

```swift
    func bootstrap() async {
        AppState.detectAndWriteGrandfatherFlag(
            favoritesFileExists: FileManager.default.fileExists(atPath: FavoritesStore.defaultURL.path)
        )
        haptics.prepare()
```

- [ ] **Step 4: Update `favoriteStation()` to gate on `canSave`**

In `drifting/drifting/App/AppState.swift`, find:

```swift
    func favoriteStation() {
        guard let station = currentStation else { return }
        guard !favorites.contains(station) else {
            haptics.playTap()
            alreadySavedFlashID = UUID()
            return
        }
        haptics.playFavorite()
```

Replace with:

```swift
    func favoriteStation() {
        guard let station = currentStation else { return }
        guard !favorites.contains(station) else {
            haptics.playTap()
            alreadySavedFlashID = UUID()
            return
        }
        guard canSave else {
            haptics.playTap()
            saveLimitReachedToast = true
            return
        }
        haptics.playFavorite()
```

- [ ] **Step 5: Run the full test suite**

```bash
xcodebuild test \
  -scheme drifting \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' 2>&1 | tail -20
```

Expected: `** TEST SUCCEEDED **`

- [ ] **Step 6: Commit**

```bash
git add drifting/drifting/App/AppState.swift
git commit -m "feat: gate favoriteStation on canSave, run grandfathering check on bootstrap"
```

---

### Task 5: Add "Save limit reached" toast to MainView

- [ ] **Step 1: Add state variable**

In `drifting/drifting/UI/MainView.swift`, find:

```swift
    @State private var showStationNotFoundToast = false
```

Add immediately after it:

```swift
    @State private var showSaveLimitToast = false
```

- [ ] **Step 2: Add toast view in ZStack**

In `drifting/drifting/UI/MainView.swift`, find:

```swift
            if showStationNotFoundToast {
                VStack {
                    Spacer()
                    Text("Station not found")
                        .font(.subheadline).fontWeight(.medium)
                        .foregroundColor(.white.opacity(0.7))
                        .padding(.horizontal, 18).padding(.vertical, 10)
                        .background(.ultraThinMaterial, in: Capsule())
                        .padding(.bottom, 120)
                        .transition(.opacity)
                }
            }
```

Add immediately after the closing brace of that block:

```swift
            if showSaveLimitToast {
                VStack {
                    Spacer()
                    Text("Save limit reached")
                        .font(.subheadline).fontWeight(.medium)
                        .foregroundColor(.white.opacity(0.7))
                        .padding(.horizontal, 18).padding(.vertical, 10)
                        .background(.ultraThinMaterial, in: Capsule())
                        .padding(.bottom, 120)
                        .transition(.opacity)
                }
                .onTapGesture {
                    withAnimation { showSaveLimitToast = false }
                    appState.saveLimitReachedToast = false
                    showSettings = true
                }
            }
```

- [ ] **Step 3: Add onChange handler**

In `drifting/drifting/UI/MainView.swift`, find:

```swift
        .onChange(of: appState.stationNotFoundToast) { _, isNotFound in
            guard isNotFound else { return }
            withAnimation { showStationNotFoundToast = true }
            DispatchQueue.main.asyncAfter(deadline: .now() + 2.5) {
                withAnimation { showStationNotFoundToast = false }
                appState.stationNotFoundToast = false
            }
        }
```

Add immediately after it:

```swift
        .onChange(of: appState.saveLimitReachedToast) { _, isLimited in
            guard isLimited else { return }
            withAnimation { showSaveLimitToast = true }
            DispatchQueue.main.asyncAfter(deadline: .now() + 3) {
                withAnimation { showSaveLimitToast = false }
                appState.saveLimitReachedToast = false
            }
        }
```

- [ ] **Step 4: Run full test suite**

```bash
xcodebuild test \
  -scheme drifting \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' 2>&1 | tail -20
```

Expected: `** TEST SUCCEEDED **`

- [ ] **Step 5: Commit**

```bash
git add drifting/drifting/UI/MainView.swift
git commit -m "feat: add Save limit reached toast in MainView"
```

---

### Task 6: Update Plus upgrade card in SettingsView

- [ ] **Step 1: Add "Unlimited saved stations" to the features list**

In `drifting/drifting/UI/SettingsView.swift`, find:

```swift
                            VStack(alignment: .leading, spacing: 6) {
                                Label("320kbps+ stream quality", systemImage: "waveform")
```

Replace with:

```swift
                            VStack(alignment: .leading, spacing: 6) {
                                Label("Unlimited saved stations", systemImage: "bookmark")
                                Label("320kbps+ stream quality", systemImage: "waveform")
```

- [ ] **Step 2: Run full test suite**

```bash
xcodebuild test \
  -scheme drifting \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' 2>&1 | tail -20
```

Expected: `** TEST SUCCEEDED **`

- [ ] **Step 3: Commit**

```bash
git add drifting/drifting/UI/SettingsView.swift
git commit -m "feat: add Unlimited saved stations to Plus upgrade card"
```

---

### Task 7: Update design spec

- [ ] **Step 1: Mark feature as implemented in design-spec.md**

In `/Users/vitorfreitas/dev/drifting/docs/design-spec.md`, find:

```
- **Saved stations limit** — free users can save up to 10 stations; Plus users get unlimited; when the limit is hit, saving is blocked and a sheet prompts upgrading; doesn't affect the core discovery loop — free users can still play and explore freely; chosen limit should feel generous enough not to frustrate casual users but meaningful enough to convert regular ones; **grandfathering**: on first launch of the v1.1 build, check for a `hasGrandfatheredUnlimitedSaves` UserDefaults flag — if absent (i.e. user installed before v1.1), write it as `true` and treat them as Plus for save limits only; this prevents backlash from early adopters who had unlimited saves under v1.0
```

Replace with:

```
- **Saved stations limit** ✅ — free users can save up to 10 stations; Plus users get unlimited; when the limit is hit, a light haptic fires and a tappable "Save limit reached" toast appears (tapping opens Settings to the Plus section); grandfathering: on first launch of v1.1, if `favorites.json` exists on disk the user is an upgrade from v1.0 and `hasGrandfatheredUnlimitedSaves` is written as `true` (treated as Plus for save limits, invisibly); fresh installs get `false`
```

Also update the feature split table in Section 13. Find:

```
| All gestures, saving, favorites | ✅ | ✅ |
```

Replace with:

```
| All gestures, saving, favorites | ✅ (up to 10 saves) | ✅ Unlimited saves |
```

- [ ] **Step 2: Commit**

```bash
cd /Users/vitorfreitas/dev/drifting && git add docs/design-spec.md && git commit -m "docs: mark saved stations limit as implemented in design spec"
cd /Users/vitorfreitas/dev/drifting-ios
```
