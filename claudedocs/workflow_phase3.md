# Phase 3 Implementation Workflow — Baby Rhythm

**Scope:** Sleep Sounds (Pillar A) + Daily AI Digest (Pillar B)
**Build order:** Models/Config → Audio → Info.plist → Pillar A → Pillar B → HomeViewModel → Navigation → Tests → Acceptance check

---

## Step 1 — Models & Config

**Files to create:**
- `baby-rhythm-app/Models/DailyDigest.swift`
- `baby-rhythm-app/Models/SleepSound.swift` (SleepSound + SoundDuration enums)
- `baby-rhythm-app/Config.swift`

**Files to modify:**
- `.gitignore` (add `Config.swift`)

**DailyDigest.swift**
```swift
import Foundation

struct DailyDigest: Codable {
    let date: Date          // noon of the day this covers
    let narrative: String   // AI-generated text
    let totalSleepSeconds: TimeInterval
    let napCount: Int
    let nightSleepSeconds: TimeInterval
    let sleepGoalSeconds: TimeInterval  // age-appropriate target (total 24h)
    let generatedAt: Date
}
```
- Not a SwiftData `@Model`; stored as JSON in `UserDefaults` (key: `"dailyDigest_YYYY-MM-DD"`).
- Cache invalidation: re-fetch if `generatedAt` < start of today or data is missing.

**SleepSound.swift**
```swift
enum SleepSound: String, CaseIterable, Identifiable {
    case whiteNoise = "white_noise"
    case pinkNoise  = "pink_noise"

    var id: String { rawValue }
    var displayName: String { /* "White Noise", "Pink Noise" */ }
    var fileName: String { rawValue }       // maps to bundle resource name
    var fileExtension: String { "mp3" }
}

enum SoundDuration: Int, CaseIterable, Identifiable {
    case fifteen  = 15
    case thirty   = 30
    case sixty    = 60
    case infinite = 0   // 0 = loop until manually stopped

    var id: Int { rawValue }
    var displayName: String { /* "15 min", "30 min", "60 min", "Continuous" */ }
}
```

**Config.swift**
```swift
enum Config {
    static let anthropicAPIKey = "REPLACE_WITH_REAL_KEY"
    static let digestModel     = "claude-opus-4-5"
    static let digestMaxTokens = 300
}
```
- Add `Config.swift` to `.gitignore` immediately after creating it.
- Add `Config.swift.example` (safe placeholder version) to track in git as a reminder.

**Key notes:**
- `DailyDigest` carries a `sleepGoalSeconds` field derived from age — see Pillar B for colour threshold logic.
- No SwiftData migration needed; new models are value types or UserDefaults-backed.

**Dependencies:** None — this step has no upstream dependencies.

---

## Step 2 — Audio Placeholder Files

**Files to create (binary resources):**
- `baby-rhythm-app/Resources/white_noise.mp3`
- `baby-rhythm-app/Resources/pink_noise.mp3`

**Generate placeholders via Python (run once, then add to Xcode bundle):**
```bash
# Option A — Python (no external deps, produces minimal valid MP3)
python3 -c "
import struct, math

def mp3_silence(path, duration_sec=5, sample_rate=44100):
    # Write a minimal valid MP3 frame header (silence)
    # Frame sync: 0xFF 0xFB, 128kbps, 44.1kHz, stereo
    header = bytes([0xFF, 0xFB, 0x90, 0x00])
    # Each frame = 417 bytes of silence payload at 128kbps/44.1kHz
    frame = header + bytes(413)
    frames = int(duration_sec * 44100 / 1152)
    with open(path, 'wb') as f:
        for _ in range(frames):
            f.write(frame)

mp3_silence('white_noise.mp3')
mp3_silence('pink_noise.mp3')
"

# Option B — ffmpeg (if installed)
ffmpeg -f lavfi -i anullsrc=r=44100:cl=stereo -t 5 -q:a 9 -acodec libmp3lame white_noise.mp3
ffmpeg -f lavfi -i anullsrc=r=44100:cl=stereo -t 5 -q:a 9 -acodec libmp3lame pink_noise.mp3
```

**Add to Xcode project:**
1. Drag both `.mp3` files into `baby-rhythm-app/Resources/` folder in Xcode Project Navigator.
2. In "Add files" dialog: check "Copy items if needed", target membership = `baby-rhythm-app`.
3. Verify both files appear in Build Phase → "Copy Bundle Resources".

**Key notes:**
- These are placeholders only; the real audio files will be dropped in by the user and must have the exact same filenames.
- `AVAudioPlayer` will silently fail (no crash) if a file is malformed — test with a real .mp3 before release.

**Dependencies:** Step 1 (SleepSound enum defines the filenames).

---

## Step 3 — Info.plist (Background Audio)

**Files to create:**
- `baby-rhythm-app/Info.plist`

**Files to modify:**
- `baby-rhythm-app.xcodeproj/project.pbxproj` — set `INFOPLIST_FILE` build setting to point at the custom plist.

**Minimal Info.plist content:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>UIBackgroundModes</key>
    <array>
        <string>audio</string>
    </array>
    <key>NSMicrophoneUsageDescription</key>
    <string>Baby Rhythm does not use the microphone.</string>
</dict>
</plist>
```

**Xcode project.pbxproj change:**
- In the Debug + Release build settings for the `baby-rhythm-app` target, set:
  `INFOPLIST_FILE = baby-rhythm-app/Info.plist`
- Remove the `GENERATE_INFOPLIST_FILE = YES` setting if present (they conflict).

**Key notes:**
- `UIBackgroundModes: audio` allows `AVAudioSession` to keep playing when the app backgrounds.
- The `AVAudioSession` category must be set to `.playback` in `SleepSoundService` (Step 4) — the plist entry alone is not sufficient.
- Do NOT add `NSPrivacyTrackingUsageDescription` or any other keys not required; Apple reviews flag unnecessary entries.

**Dependencies:** Step 2 (conceptually tied to audio feature existence).

---

## Step 4 — Pillar A Services

**Files to create:**
- `baby-rhythm-app/Services/SleepSoundService.swift`
- `baby-rhythm-app/ViewModels/SleepSoundViewModel.swift`

**SleepSoundService.swift**
```swift
import AVFoundation

final class SleepSoundService {
    static let shared = SleepSoundService()

    private var player: AVAudioPlayer?
    private var stopTimer: Timer?

    // MARK: - Playback

    func play(sound: SleepSound, duration: SoundDuration) throws {
        stop()
        try AVAudioSession.sharedInstance().setCategory(.playback, mode: .default)
        try AVAudioSession.sharedInstance().setActive(true)

        guard let url = Bundle.main.url(forResource: sound.fileName, withExtension: sound.fileExtension) else {
            throw SleepSoundError.fileNotFound(sound.fileName)
        }
        player = try AVAudioPlayer(contentsOf: url)
        player?.numberOfLoops = -1   // always loop; timer handles finite durations
        player?.play()

        if duration != .infinite {
            stopTimer = Timer.scheduledTimer(withTimeInterval: Double(duration.rawValue) * 60,
                                             repeats: false) { [weak self] _ in
                self?.stop()
            }
        }
    }

    func stop() {
        stopTimer?.invalidate()
        stopTimer = nil
        player?.stop()
        player = nil
        try? AVAudioSession.sharedInstance().setActive(false, options: .notifyOthersOnDeactivation)
    }

    var isPlaying: Bool { player?.isPlaying ?? false }
}

enum SleepSoundError: LocalizedError {
    case fileNotFound(String)
    var errorDescription: String? {
        switch self {
        case .fileNotFound(let name): return "Audio file '\(name).mp3' not found in bundle."
        }
    }
}
```

**SleepSoundViewModel.swift**
```swift
import Foundation

@Observable
final class SleepSoundViewModel {
    private let service: SleepSoundService

    var selectedSound: SleepSound = .whiteNoise
    var selectedDuration: SoundDuration = .thirty
    var isPlaying: Bool = false
    var errorMessage: String?

    init(service: SleepSoundService = .shared) {
        self.service = service
    }

    func togglePlayback() {
        if isPlaying {
            service.stop()
            isPlaying = false
        } else {
            do {
                try service.play(sound: selectedSound, duration: selectedDuration)
                isPlaying = true
                errorMessage = nil
            } catch {
                errorMessage = error.localizedDescription
            }
        }
    }

    func stopIfNeeded() {
        if isPlaying { service.stop(); isPlaying = false }
    }
}
```

**Key notes:**
- `SleepSoundService` is a singleton accessed by the ViewModel — matches the stateless `NotificationService` pattern already in the project.
- `@Observable` macro follows the exact pattern of `HomeViewModel`.
- AVAudioSession category must be set before each `play()` call (it may be reset by other audio).
- `numberOfLoops = -1` means infinite loop; the `Timer` provides finite-duration auto-stop.

**Dependencies:** Steps 1, 2, 3.

---

## Step 5 — Pillar A UI

**Files to create:**
- `baby-rhythm-app/Views/SleepSounds/SleepSoundsView.swift`
- `baby-rhythm-app/Views/Home/SleepSoundsCardView.swift` (compact home card)

**SleepSoundsView.swift — full-screen pushed view**
Layout (top → bottom):
1. Section header: "Sleep Sounds"
2. Sound picker: `Picker` with `.segmented` style showing all `SleepSound.allCases`
3. Duration picker: `Picker` with `.segmented` style showing all `SoundDuration.allCases`
4. Play/Stop button: `PrimaryButton` (reuse from DesignSystem), title toggles "Play" / "Stop"
5. Error inline alert (per confirmed decision: inline, not modal):
   ```swift
   if let msg = viewModel.errorMessage {
       Text(msg)
           .font(.caption)
           .foregroundStyle(Color.brRiskHigh)
           .padding(.horizontal, DS.Spacing.lg)
   }
   ```
6. Privacy disclosure (inline, below controls):
   ```swift
   Text("Audio plays in background. No data is recorded.")
       .font(.caption)
       .foregroundStyle(Color.brMuted)
       .multilineTextAlignment(.center)
       .padding(.horizontal, DS.Spacing.xl)
   ```

**SleepSoundsCardView.swift — compact home card**
- Shows: sound name (if playing) or "Sleep Sounds" label, play/stop icon button, chevron for navigation.
- Tapping the card body navigates to `SleepSoundsView` via the `NavigationStack` destination (Step 9).
- Uses `.cardStyle()` modifier from DesignSystem.
- Takes `SleepSoundViewModel` as a parameter (passed from `HomeViewModel`).

**Key notes:**
- Both views receive `SleepSoundViewModel` — no `@State` allocation in the card; the VM is owned by `HomeViewModel`.
- The compact card is added to `HomeView`'s `VStack` between `NextNapCardView` and `notificationBanner`.
- Use `DS.Spacing.*` and `.cardStyle()` consistently with existing cards.

**Dependencies:** Steps 1, 4.

---

## Step 6 — Pillar B Services

**Files to create:**
- `baby-rhythm-app/Services/DigestService.swift`
- `baby-rhythm-app/ViewModels/DigestViewModel.swift`

**DigestService.swift**

Responsibilities:
1. Compute today's sleep totals from a `[SleepSession]` array passed in.
2. Check cache (`UserDefaults` key `"digest_YYYY-MM-DD"`): return cached `DailyDigest` if present and generated today.
3. Rate-limit guard: if a request was made in the last 60 seconds (key `"digestLastRequestDate"`), return cached value without hitting the API.
4. Build the Anthropic API prompt and call `https://api.anthropic.com/v1/messages`.
5. Decode the response, construct `DailyDigest`, cache it, return it.

```swift
struct DigestService {

    // MARK: - Cache

    static func cachedDigest(for date: Date) -> DailyDigest? {
        let key = cacheKey(for: date)
        guard let data = UserDefaults.standard.data(forKey: key) else { return nil }
        let digest = try? JSONDecoder().decode(DailyDigest.self, from: data)
        // Invalidate if not generated today
        guard let d = digest,
              Calendar.current.isDate(d.generatedAt, inSameDayAs: date) else { return nil }
        return digest
    }

    static func cache(_ digest: DailyDigest, for date: Date) {
        if let data = try? JSONEncoder().encode(digest) {
            UserDefaults.standard.set(data, forKey: cacheKey(for: date))
        }
    }

    // MARK: - Fetch

    static func fetchDigest(sessions: [SleepSession], child: Child) async throws -> DailyDigest {
        let today = Date()

        if let cached = cachedDigest(for: today) { return cached }

        // Rate-limit check
        if let last = UserDefaults.standard.object(forKey: "digestLastRequestDate") as? Date,
           Date().timeIntervalSince(last) < 60 {
            throw DigestError.rateLimited
        }
        UserDefaults.standard.set(Date(), forKey: "digestLastRequestDate")

        // Compute totals
        let completed = sessions.filter { $0.endTime != nil }
        let naps = completed.filter { $0.type == .nap }
        let nights = completed.filter { $0.type == .nightSleep }
        let napTotal = naps.compactMap(\.duration).reduce(0, +)
        let nightTotal = nights.compactMap(\.duration).reduce(0, +)
        let totalSleep = napTotal + nightTotal

        let age = SleepScheduleService.ageInMonths(for: child)
        let goal = sleepGoalSeconds(forAgeInMonths: age)

        // Build prompt
        let prompt = buildPrompt(child: child, age: age, napTotal: napTotal,
                                  nightTotal: nightTotal, totalSleep: totalSleep,
                                  goal: goal, napCount: naps.count)

        // API call
        let narrative = try await callAnthropicAPI(prompt: prompt)

        let digest = DailyDigest(
            date: today,
            narrative: narrative,
            totalSleepSeconds: totalSleep,
            napCount: naps.count,
            nightSleepSeconds: nightTotal,
            sleepGoalSeconds: goal,
            generatedAt: Date()
        )
        cache(digest, for: today)
        return digest
    }

    // MARK: - Sleep Goal Table (total 24h, midpoints)
    // Age 0-2m: 16h, 3-5m: 15h, 6-8m: 14h, 9-11m: 14h, 12-17m: 13h, 18-23m: 13h, 24-36m: 12h

    static func sleepGoalSeconds(forAgeInMonths age: Int) -> TimeInterval {
        switch age {
        case 0...2:  return 16 * 3600
        case 3...5:  return 15 * 3600
        case 6...11: return 14 * 3600
        case 12...23: return 13 * 3600
        default:     return 12 * 3600
        }
    }

    // MARK: - Prompt

    private static func buildPrompt(child: Child, age: Int, napTotal: TimeInterval,
                                     nightTotal: TimeInterval, totalSleep: TimeInterval,
                                     goal: TimeInterval, napCount: Int) -> String {
        let fmt: (TimeInterval) -> String = { s in
            let h = Int(s) / 3600; let m = (Int(s) % 3600) / 60
            return h > 0 ? "\(h)h \(m)m" : "\(m)m"
        }
        return """
        You are a friendly baby sleep coach summarising one day of sleep data for a parent.
        Baby: \(child.name), age \(age) months.
        Night sleep: \(fmt(nightTotal)). Daytime naps: \(napCount) nap(s), \(fmt(napTotal)).
        Total sleep: \(fmt(totalSleep)) (age-appropriate goal ~\(fmt(goal))).
        Write 2–3 upbeat sentences. Focus on patterns and one actionable tip. Be concise.
        """
    }

    // MARK: - Anthropic API

    private static func callAnthropicAPI(prompt: String) async throws -> String {
        guard let url = URL(string: "https://api.anthropic.com/v1/messages") else {
            throw DigestError.invalidURL
        }
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.setValue(Config.anthropicAPIKey, forHTTPHeaderField: "x-api-key")
        request.setValue("2023-06-01", forHTTPHeaderField: "anthropic-version")

        let body: [String: Any] = [
            "model": Config.digestModel,
            "max_tokens": Config.digestMaxTokens,
            "messages": [["role": "user", "content": prompt]]
        ]
        request.httpBody = try JSONSerialization.data(withJSONObject: body)

        let (data, response) = try await URLSession.shared.data(for: request)
        guard let http = response as? HTTPURLResponse, http.statusCode == 200 else {
            throw DigestError.apiError((response as? HTTPURLResponse)?.statusCode ?? 0)
        }

        // Decode: {"content": [{"type": "text", "text": "..."}]}
        struct APIResponse: Decodable {
            struct Content: Decodable { let type: String; let text: String }
            let content: [Content]
        }
        let decoded = try JSONDecoder().decode(APIResponse.self, from: data)
        return decoded.content.first(where: { $0.type == "text" })?.text ?? ""
    }

    // MARK: - Helpers

    private static func cacheKey(for date: Date) -> String {
        let fmt = DateFormatter(); fmt.dateFormat = "yyyy-MM-dd"
        return "digest_\(fmt.string(from: date))"
    }
}

enum DigestError: LocalizedError {
    case rateLimited, invalidURL, apiError(Int)
    var errorDescription: String? {
        switch self {
        case .rateLimited: return "Please wait a moment before refreshing."
        case .invalidURL:  return "Internal error: invalid API URL."
        case .apiError(let code): return "API error (HTTP \(code)). Check your API key."
        }
    }
}
```

**DigestViewModel.swift**
```swift
import Foundation

@Observable
final class DigestViewModel {
    var digest: DailyDigest?
    var isLoading: Bool = false
    var errorMessage: String?

    func load(sessions: [SleepSession], child: Child) {
        // Show cached immediately if available
        digest = DigestService.cachedDigest(for: Date())
        guard !isLoading else { return }
        isLoading = true
        Task { @MainActor in
            defer { isLoading = false }
            do {
                digest = try await DigestService.fetchDigest(sessions: sessions, child: child)
                errorMessage = nil
            } catch DigestError.rateLimited {
                // Silently ignore — cached value already shown
            } catch {
                errorMessage = error.localizedDescription
            }
        }
    }
}
```

**Colour threshold logic (confirmed: green >= 90% of target midpoint, amber 70–89%, red < 70%):**
```swift
// In DigestService or a shared helper
static func sleepColour(totalSleepSeconds: TimeInterval, goalSeconds: TimeInterval) -> Color {
    guard goalSeconds > 0 else { return .brMuted }
    let ratio = totalSleepSeconds / goalSeconds
    if ratio >= 0.90 { return .brRiskLow }    // green
    if ratio >= 0.70 { return .brRiskMedium } // amber
    return .brRiskHigh                         // red
}
```

**Key notes:**
- `DigestService` is a pure `struct` with static methods — matches the `NotificationService` / `WakeWindowService` pattern.
- Cache is keyed by date string so yesterday's digest is not shown as today's.
- The rate-limit guard prevents double-taps from burning API credits.
- `DigestViewModel` mirrors the `@Observable` + `Task { @MainActor }` pattern seen in `HomeViewModel`.

**Dependencies:** Step 1 (DailyDigest model, Config).

---

## Step 7 — Pillar B UI

**Files to create:**
- `baby-rhythm-app/Views/Home/DigestCardView.swift`

**DigestCardView layout:**

```
┌────────────────────────────────────────────────────────┐
│  "Today's Sleep Summary"  [Refresh icon button]         │
│                                                         │
│  SHIMMER PLACEHOLDER (while isLoading && digest == nil) │
│  ─ or ─                                                 │
│  Totals row:                                            │
│    [Night icon]  Xh Ym    [Nap icon]  N naps  Xh Ym   │
│                                                         │
│  Goal bar:                                              │
│    ████████░░░░  85% of goal  (coloured per threshold)  │
│                                                         │
│  Narrative text (2–3 sentences, .body font, .brInk)    │
│                                                         │
│  Error inline: red caption text (if errorMessage set)  │
└────────────────────────────────────────────────────────┘
```

**Shimmer effect** — use a simple `redacted(reason: .placeholder)` modifier on placeholder `Text` views while loading, or implement a gradient shimmer:
```swift
struct ShimmerModifier: ViewModifier {
    @State private var phase: CGFloat = 0
    func body(content: Content) -> some View {
        content
            .overlay(
                LinearGradient(colors: [.clear, .white.opacity(0.6), .clear],
                               startPoint: .leading, endPoint: .trailing)
                    .rotationEffect(.degrees(30))
                    .offset(x: phase * 400 - 200)
                    .animation(.linear(duration: 1.2).repeatForever(autoreverses: false), value: phase)
            )
            .onAppear { phase = 1 }
            .clipped()
    }
}
```

**Goal bar:**
```swift
GeometryReader { geo in
    ZStack(alignment: .leading) {
        RoundedRectangle(cornerRadius: 4).fill(Color.brBorder).frame(height: 8)
        RoundedRectangle(cornerRadius: 4)
            .fill(DigestService.sleepColour(totalSleepSeconds: digest.totalSleepSeconds,
                                             goalSeconds: digest.sleepGoalSeconds))
            .frame(width: geo.size.width * min(ratio, 1.0), height: 8)
    }
}
.frame(height: 8)
```

**Refresh button:** `Button { viewModel.load(sessions: sessions, child: child) }` with `Image(systemName: "arrow.clockwise")`. Show `.disabled(viewModel.isLoading)`.

**Key notes:**
- `DigestCardView` takes `DigestViewModel`, `[SleepSession]`, and `Child` as parameters.
- The card is inserted into `HomeView`'s `VStack` below the `SleepSoundsCardView`.
- Uses `.cardStyle()` from DesignSystem.

**Dependencies:** Steps 1, 6.

---

## Step 8 — HomeViewModel Updates

**Files to modify:**
- `baby-rhythm-app/ViewModels/HomeViewModel.swift`

**Changes:**
1. Add stored properties:
   ```swift
   private(set) var sleepSoundViewModel = SleepSoundViewModel()
   private(set) var digestViewModel = DigestViewModel()
   ```
2. In `refresh()`, after fetching sessions, trigger digest load:
   ```swift
   let sessions = todaySessions()
   digestViewModel.load(sessions: sessions, child: child)
   ```
3. In `HomeView.onChange(of: scenePhase)` for `.active` phase, also call `viewModel.refreshDigest()` — add a dedicated method:
   ```swift
   func refreshDigest() {
       digestViewModel.load(sessions: todaySessions(), child: child)
   }
   ```
4. Stop sound when app backgrounds (scenePhase → `.background`):
   ```swift
   // In HomeView.onChange(of: scenePhase):
   if newPhase == .background { viewModel.sleepSoundViewModel.stopIfNeeded() }
   ```
   Or handle this inside `SleepSoundViewModel` via a scenePhase observation.

**Key notes:**
- The two new child ViewModels are owned by `HomeViewModel` — this matches the existing pattern where `HomeViewModel` is the single source of truth for the home screen.
- `DigestViewModel.load()` is idempotent: it checks the cache first and does nothing if already loaded today.
- `SleepSoundViewModel` state persists across navigations because it lives on `HomeViewModel`.

**Dependencies:** Steps 4, 6.

---

## Step 9 — Navigation

**Files to modify:**
- `baby-rhythm-app/Views/Home/HomeView.swift`
- `baby-rhythm-app/baby_rhythm_appApp.swift`

**HomeView.swift changes:**
1. Add `SleepSoundsCardView(viewModel: viewModel.sleepSoundViewModel)` to the main `VStack`.
2. Add `DigestCardView(viewModel: viewModel.digestViewModel, sessions: viewModel.todaySessions(), child: viewModel.child)` below it.
3. Extend `navigationDestination(for: String.self)`:
   ```swift
   case "sleepSounds":
       SleepSoundsView(viewModel: viewModel.sleepSoundViewModel)
   ```
4. Add scenePhase handler for `.active` to trigger digest refresh (see Step 8).

**baby_rhythm_appApp.swift — no schema change needed:**
- `DailyDigest` is not a SwiftData model, so `.modelContainer(for: [Child.self, SleepSession.self])` remains unchanged.
- If `SleepSoundService.shared` needs early setup (e.g., AVAudioSession category), add an `init()` to `BabyRhythmApp` that calls it:
  ```swift
  init() {
      try? AVAudioSession.sharedInstance().setCategory(.playback, mode: .default)
  }
  ```

**Key notes:**
- Navigation uses the existing `String`-keyed `NavigationStack` pattern — no new `NavigationPath` or enum type required.
- The compact `SleepSoundsCardView` navigates via `NavigationLink(value: "sleepSounds")`.

**Dependencies:** Steps 5, 7, 8.

---

## Step 10 — Tests

**Files to create:**
- `baby-rhythm-appTests/SleepSoundServiceTests.swift`
- `baby-rhythm-appTests/DigestServiceTests.swift`

**Pattern** — follow the exact XCTest structure from `WakeWindowServiceTests.swift`:
```swift
import XCTest
@testable import baby_rhythm_app

final class SleepSoundServiceTests: XCTestCase { ... }
final class DigestServiceTests: XCTestCase { ... }
```

**SleepSoundServiceTests — test cases:**
```
test_play_whiteNoise_setsIsPlayingTrue
test_stop_afterPlay_setsIsPlayingFalse
test_play_unknownFile_throwsFileNotFound      // use a SleepSound with a bad fileName
test_soundDuration_infinite_rawValueIsZero
test_sleepSound_allCases_haveUniqueFileNames
```

**DigestServiceTests — test cases:**
```
test_sleepGoalSeconds_age0_returns16h
test_sleepGoalSeconds_age6_returns14h
test_sleepGoalSeconds_age24_returns12h
test_sleepColour_above90percent_isGreen
test_sleepColour_80percent_isAmber
test_sleepColour_60percent_isRed
test_cachedDigest_returnsNil_whenNothingCached
test_cachedDigest_returnsNil_whenGeneratedYesterday
test_cacheRoundTrip_encodesAndDecodesDigest
test_buildPrompt_containsChildName            // via a testable internal prompt helper
```

**Key notes:**
- `SleepSoundService.play()` requires a real `.mp3` file in the bundle; in tests, use `XCTSkipIf` if running in CI without resources, or inject a mock URL.
- `DigestService.fetchDigest()` hits the real API — mark these tests as requiring an API key with `XCTSkipUnless(!Config.anthropicAPIKey.hasPrefix("REPLACE"))`.
- All pure logic tests (goal table, colour thresholds, cache encode/decode) run without any mocking.

**Dependencies:** All prior steps.

---

## Step 11 — Final Checklist

**Acceptance criteria to verify manually or via XCTest:**

**Pillar A — Sleep Sounds:**
- [ ] Tapping Play starts audio; indicator shows "Playing".
- [ ] Audio continues when app goes to background (screen locked).
- [ ] Tapping Stop ends audio immediately.
- [ ] Timer auto-stops audio after selected duration (verify with 1-minute test duration if possible).
- [ ] Continuous mode loops indefinitely until manual stop.
- [ ] Sound picker selection persists within the session (state on ViewModel).
- [ ] If `.mp3` file is missing, an inline error message appears (no crash).
- [ ] Privacy disclosure line is visible below controls.

**Pillar B — Daily AI Digest:**
- [ ] On first app open of the day, digest card shows shimmer then populates.
- [ ] Narrative is 2–3 sentences, readable, baby name visible.
- [ ] Night sleep, nap count, and total sleep figures are accurate vs. logged sessions.
- [ ] Goal bar colour: green when total >= 90% of goal, amber 70–89%, red below 70%.
- [ ] Tapping refresh re-fetches; rate-limit message shown if tapped twice within 60 s.
- [ ] Cached digest survives app restart within the same calendar day.
- [ ] With a placeholder API key, an inline error message appears (no crash or empty card).

**Phase 1 + 2 regression:**
- [ ] All 11 Phase 1 acceptance criteria still pass.
- [ ] All Phase 2 acceptance criteria still pass (notifications, risk badge, preferences, bedtime).
- [ ] `xcodebuild test` runs clean (zero failures).

---

## File Map Summary

```
baby-rhythm-app/
  Models/
    DailyDigest.swift          [NEW — Step 1]
    SleepSound.swift           [NEW — Step 1]
    Child.swift                [unchanged]
    SleepSession.swift         [unchanged]
  Services/
    SleepSoundService.swift    [NEW — Step 4]
    DigestService.swift        [NEW — Step 6]
    NotificationService.swift  [unchanged]
    SleepScheduleService.swift [unchanged]
    WakeWindowService.swift    [unchanged]
  ViewModels/
    SleepSoundViewModel.swift  [NEW — Step 4]
    DigestViewModel.swift      [NEW — Step 6]
    HomeViewModel.swift        [MODIFIED — Step 8]
  Views/
    Home/
      HomeView.swift           [MODIFIED — Step 9]
      NextNapCardView.swift    [unchanged]
      SleepSoundsCardView.swift [NEW — Step 5]
      DigestCardView.swift     [NEW — Step 7]
    SleepSounds/
      SleepSoundsView.swift    [NEW — Step 5]
  Shared/
    DesignSystem.swift         [unchanged]
  Resources/
    white_noise.mp3            [NEW — Step 2]
    pink_noise.mp3             [NEW — Step 2]
  Config.swift                 [NEW — Step 1, git-ignored]
  Info.plist                   [NEW — Step 3]
  baby_rhythm_appApp.swift     [MODIFIED — Step 9]

baby-rhythm-appTests/
  SleepSoundServiceTests.swift [NEW — Step 10]
  DigestServiceTests.swift     [NEW — Step 10]

.gitignore                     [MODIFIED — Step 1, add Config.swift]
Config.swift.example           [NEW — Step 1, tracked in git]
```
