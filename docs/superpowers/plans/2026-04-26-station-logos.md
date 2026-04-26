# Station Logos Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Show a small station logo in the InfoOverlay to the left of the station name, falling back to a ColorIdentity radial gradient tile when no favicon is available.

**Architecture:** `Station` gains an optional `faviconURL: URL?` decoded from the Radio Browser `favicon` field (validated to http/https only). A new `StationLogoView` always renders the gradient fallback and layers an `AsyncImage` on top that fades in on success. `InfoOverlay` wraps its existing station-info VStack in an HStack with the logo on the left.

**Tech Stack:** Swift, SwiftUI, AsyncImage, ColorIdentity, URLSession (via AsyncImage), XCTest

---

## File Map

- **Modify:** `drifting/drifting/Models/Station.swift` — add `faviconURL: URL?` with http/https validation
- **Modify:** `drifting/driftingTests/StationModelTests.swift` — add 4 favicon decoding tests
- **Create:** `drifting/drifting/UI/StationLogoView.swift` — gradient fallback + AsyncImage overlay
- **Modify:** `drifting/drifting/UI/InfoOverlay.swift` — wrap station info VStack in HStack with logo

---

### Task 1: Write failing tests for faviconURL decoding

**Files:**
- Modify: `drifting/driftingTests/StationModelTests.swift`

- [ ] **Step 1: Add 4 favicon tests to `StationModelTests.swift`**

Open `drifting/driftingTests/StationModelTests.swift`. Add the following four tests after the last existing test (`test_encodesAndDecodesRoundTrip`):

```swift
    func test_decodesValidFaviconURL() throws {
        let json = """
        {
            "stationuuid": "abc-123",
            "name": "Jazz FM",
            "url_resolved": "http://stream.example.com/jazz",
            "bitrate": 128,
            "codec": "MP3",
            "tags": "jazz",
            "language": "english",
            "countrycode": "GB",
            "lastcheckok": 1,
            "favicon": "https://example.com/logo.png"
        }
        """.data(using: .utf8)!
        let station = try JSONDecoder().decode(Station.self, from: json)
        XCTAssertEqual(station.faviconURL, URL(string: "https://example.com/logo.png"))
    }

    func test_emptyFaviconStringDecodesToNil() throws {
        let json = """
        {
            "stationuuid": "abc-123",
            "name": "Jazz FM",
            "url_resolved": "http://stream.example.com/jazz",
            "bitrate": 128,
            "codec": "MP3",
            "tags": "jazz",
            "language": "english",
            "countrycode": "GB",
            "lastcheckok": 1,
            "favicon": ""
        }
        """.data(using: .utf8)!
        let station = try JSONDecoder().decode(Station.self, from: json)
        XCTAssertNil(station.faviconURL)
    }

    func test_missingFaviconKeyDecodesToNil() throws {
        let json = """
        {
            "stationuuid": "abc-123",
            "name": "Jazz FM",
            "url_resolved": "http://stream.example.com/jazz",
            "bitrate": 128,
            "codec": "MP3",
            "tags": "jazz",
            "language": "english",
            "countrycode": "GB",
            "lastcheckok": 1
        }
        """.data(using: .utf8)!
        let station = try JSONDecoder().decode(Station.self, from: json)
        XCTAssertNil(station.faviconURL)
    }

    func test_nonHttpFaviconSchemeDecodesToNil() throws {
        let json = """
        {
            "stationuuid": "abc-123",
            "name": "Jazz FM",
            "url_resolved": "http://stream.example.com/jazz",
            "bitrate": 128,
            "codec": "MP3",
            "tags": "jazz",
            "language": "english",
            "countrycode": "GB",
            "lastcheckok": 1,
            "favicon": "file:///etc/passwd"
        }
        """.data(using: .utf8)!
        let station = try JSONDecoder().decode(Station.self, from: json)
        XCTAssertNil(station.faviconURL)
    }
```

- [ ] **Step 2: Run tests to confirm they fail**

```bash
cd /Users/vitorfreitas/dev/drifting-ios/drifting && xcodebuild test \
  -scheme drifting \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' \
  -only-testing:driftingTests/StationModelTests 2>&1 | tail -15
```

Expected: compile error — `Station` has no member `faviconURL`.

---

### Task 2: Add faviconURL to Station model

**Files:**
- Modify: `drifting/drifting/Models/Station.swift`

- [ ] **Step 1: Add the property, CodingKey, decode, encode, and update init + Equatable**

Replace the entire contents of `drifting/drifting/Models/Station.swift` with:

```swift
import Foundation

struct Station: Codable, Identifiable {
    let id: String
    let name: String
    let streamURL: URL
    let bitrate: Int
    let codec: String
    let tags: [String]
    let language: String
    let countryCode: String
    let state: String
    let isActive: Bool
    let faviconURL: URL?

    enum CodingKeys: String, CodingKey {
        case id = "stationuuid"
        case name
        case streamURL = "url_resolved"
        case bitrate
        case codec
        case tags
        case language
        case countryCode = "countrycode"
        case state
        case isActive = "lastcheckok"
        case faviconURL = "favicon"
    }

    nonisolated init(from decoder: Decoder) throws {
        let c = try decoder.container(keyedBy: CodingKeys.self)
        id = try c.decode(String.self, forKey: .id)
        name = try c.decode(String.self, forKey: .name).sanitizingHTML()
        let urlString = try c.decode(String.self, forKey: .streamURL)
        guard let url = URL(string: urlString) else {
            throw DecodingError.dataCorruptedError(forKey: .streamURL, in: c, debugDescription: "Invalid URL")
        }
        streamURL = url
        bitrate = try c.decode(Int.self, forKey: .bitrate)
        codec = try c.decode(String.self, forKey: .codec)
        let rawTags = try c.decode(String.self, forKey: .tags)
        tags = rawTags.isEmpty ? [] : rawTags.split(separator: ",").map { $0.trimmingCharacters(in: .whitespaces).sanitizingHTML() }
        language = try c.decode(String.self, forKey: .language)
        countryCode = try c.decode(String.self, forKey: .countryCode)
        state = (try? c.decode(String.self, forKey: .state)) ?? ""
        let activeInt = try c.decode(Int.self, forKey: .isActive)
        isActive = activeInt == 1
        let faviconString = try? c.decode(String.self, forKey: .faviconURL)
        faviconURL = faviconString.flatMap { str in
            guard !str.isEmpty,
                  let url = URL(string: str),
                  url.scheme == "http" || url.scheme == "https" else { return nil }
            return url
        }
    }

    nonisolated func encode(to encoder: Encoder) throws {
        var c = encoder.container(keyedBy: CodingKeys.self)
        try c.encode(id, forKey: .id)
        try c.encode(name, forKey: .name)
        try c.encode(streamURL.absoluteString, forKey: .streamURL)
        try c.encode(bitrate, forKey: .bitrate)
        try c.encode(codec, forKey: .codec)
        try c.encode(tags.joined(separator: ","), forKey: .tags)
        try c.encode(language, forKey: .language)
        try c.encode(countryCode, forKey: .countryCode)
        try c.encode(state, forKey: .state)
        try c.encode(isActive ? 1 : 0, forKey: .isActive)
        try c.encodeIfPresent(faviconURL?.absoluteString, forKey: .faviconURL)
    }

    var isTalk: Bool {
        let nonMusic: Set<String> = [
            "talk", "news", "sports", "quran", "spoken word", "podcast",
            "politics", "comedy", "humor", "islamic", "muslim",
            "christian", "religion", "religious"
        ]
        return tags.contains { nonMusic.contains($0.lowercased()) }
    }

    // Convenience init for tests
    init(id: String, name: String, streamURL: URL, bitrate: Int, codec: String = "MP3",
         tags: [String] = [], language: String = "", countryCode: String = "", state: String = "",
         isActive: Bool = true, faviconURL: URL? = nil) {
        self.id = id; self.name = name; self.streamURL = streamURL
        self.bitrate = bitrate; self.codec = codec; self.tags = tags
        self.language = language; self.countryCode = countryCode; self.state = state
        self.isActive = isActive; self.faviconURL = faviconURL
    }
}

extension Station: @unchecked Sendable {}

extension Station: Equatable {
    nonisolated static func == (lhs: Station, rhs: Station) -> Bool {
        lhs.id == rhs.id &&
        lhs.name == rhs.name &&
        lhs.streamURL == rhs.streamURL &&
        lhs.bitrate == rhs.bitrate &&
        lhs.tags == rhs.tags &&
        lhs.language == rhs.language &&
        lhs.countryCode == rhs.countryCode &&
        lhs.state == rhs.state &&
        lhs.isActive == rhs.isActive &&
        lhs.faviconURL == rhs.faviconURL
    }
}
```

- [ ] **Step 2: Run StationModelTests to verify all tests pass**

```bash
cd /Users/vitorfreitas/dev/drifting-ios/drifting && xcodebuild test \
  -scheme drifting \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' \
  -only-testing:driftingTests/StationModelTests 2>&1 | tail -10
```

Expected: `** TEST SUCCEEDED **`

- [ ] **Step 3: Run full test suite to verify no regressions**

```bash
cd /Users/vitorfreitas/dev/drifting-ios/drifting && xcodebuild test \
  -scheme drifting \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' 2>&1 | tail -5
```

Expected: `** TEST SUCCEEDED **`

- [ ] **Step 4: Commit**

```bash
cd /Users/vitorfreitas/dev/drifting-ios && \
  git add drifting/drifting/Models/Station.swift drifting/driftingTests/StationModelTests.swift && \
  git commit -m "feat: add faviconURL to Station model with http/https validation"
```

---

### Task 3: Create StationLogoView

**Files:**
- Create: `drifting/drifting/UI/StationLogoView.swift`

No unit tests — pure SwiftUI view with no logic.

- [ ] **Step 1: Create `StationLogoView.swift`**

Create the file `drifting/drifting/UI/StationLogoView.swift` with this exact content:

```swift
import SwiftUI

struct StationLogoView: View {
    let station: Station
    var size: CGFloat = 32

    var body: some View {
        let identity = ColorIdentity(stationID: station.id)
        ZStack {
            RadialGradient(
                colors: [identity.secondaryColor, identity.primaryColor],
                center: UnitPoint(x: 0.42, y: 0.38),
                startRadius: 0,
                endRadius: size * 0.55
            )
            if let url = station.faviconURL {
                AsyncImage(url: url) { phase in
                    Group {
                        if let image = phase.image {
                            image
                                .resizable()
                                .aspectRatio(contentMode: .fill)
                        } else {
                            Color.clear
                        }
                    }
                    .animation(.easeIn(duration: 0.25), value: phase.image != nil)
                }
            }
        }
        .frame(width: size, height: size)
        .clipShape(RoundedRectangle(cornerRadius: 6))
    }
}
```

- [ ] **Step 2: Build to confirm it compiles**

```bash
cd /Users/vitorfreitas/dev/drifting-ios/drifting && xcodebuild build \
  -scheme drifting \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' 2>&1 | tail -5
```

Expected: `** BUILD SUCCEEDED **`

- [ ] **Step 3: Commit**

```bash
cd /Users/vitorfreitas/dev/drifting-ios && \
  git add drifting/drifting/UI/StationLogoView.swift && \
  git commit -m "feat: add StationLogoView with gradient fallback and AsyncImage"
```

---

### Task 4: Integrate StationLogoView into InfoOverlay

**Files:**
- Modify: `drifting/drifting/UI/InfoOverlay.swift`

- [ ] **Step 1: Wrap the station info VStack in an HStack with the logo**

In `drifting/drifting/UI/InfoOverlay.swift`, find:

```swift
            VStack(alignment: .leading, spacing: 2) {
                if let track = currentTrack {
                    Text(track)
                        .font(.caption)
                        .foregroundColor(.white.opacity(0.85))
                        .lineLimit(1)
                }
                HStack(spacing: 5) {
                    Text(station.name)
                        .font(.subheadline).fontWeight(.semibold).foregroundColor(.white)
                        .lineLimit(1)
```

Replace with:

```swift
            HStack(alignment: .top, spacing: 8) {
                StationLogoView(station: station)
            VStack(alignment: .leading, spacing: 2) {
                if let track = currentTrack {
                    Text(track)
                        .font(.caption)
                        .foregroundColor(.white.opacity(0.85))
                        .lineLimit(1)
                }
                HStack(spacing: 5) {
                    Text(station.name)
                        .font(.subheadline).fontWeight(.semibold).foregroundColor(.white)
                        .lineLimit(1)
```

Then find the closing `}` of the VStack (the one before `.padding(.horizontal, 24)`):

```swift
            }
            .padding(.horizontal, 24)
            .padding(.bottom, 48)
```

Replace with:

```swift
            }
            }
            .padding(.horizontal, 24)
            .padding(.bottom, 48)
```

- [ ] **Step 2: Run the full test suite**

```bash
cd /Users/vitorfreitas/dev/drifting-ios/drifting && xcodebuild test \
  -scheme drifting \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' 2>&1 | tail -5
```

Expected: `** TEST SUCCEEDED **`

- [ ] **Step 3: Commit**

```bash
cd /Users/vitorfreitas/dev/drifting-ios && \
  git add drifting/drifting/UI/InfoOverlay.swift && \
  git commit -m "feat: add station logo to InfoOverlay"
```

---

### Task 5: Update design spec

**Files:**
- Modify: `/Users/vitorfreitas/dev/drifting/docs/design-spec.md`

- [ ] **Step 1: Mark the v1.1 item as implemented**

In `/Users/vitorfreitas/dev/drifting/docs/design-spec.md`, find:

```
- **Station logos / artwork** — Radio Browser provides favicon/logo URLs for many stations; show small station logo in info overlay; makes the app feel more complete alongside other radio apps
```

Replace with:

```
- **Station logos / artwork** ✅ — 32pt rounded tile to the left of the station name in InfoOverlay; loads favicon from Radio Browser `favicon` field (http/https only, validated at decode time); falls back to a radial gradient derived from the station's `ColorIdentity` (same visual as lock screen / Dynamic Island artwork); gradient shown immediately, favicon fades in once loaded
```

- [ ] **Step 2: Commit**

```bash
cd /Users/vitorfreitas/dev/drifting && \
  git add docs/design-spec.md && \
  git commit -m "docs: mark station logos as implemented in design spec"
```
