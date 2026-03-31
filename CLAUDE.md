# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Run

This is an iOS app built with Xcode. There is no package manager or Makefile.

```bash
# Build (Debug)
xcodebuild -project "baby-rhythm-app/baby-rhythm-app.xcodeproj" -scheme baby-rhythm-app -configuration Debug build

# Build (Release)
xcodebuild -project "baby-rhythm-app/baby-rhythm-app.xcodeproj" -scheme baby-rhythm-app -configuration Release build

# Run tests
xcodebuild -project "baby-rhythm-app/baby-rhythm-app.xcodeproj" -scheme baby-rhythm-app test
```

Open `baby-rhythm-app/baby-rhythm-app.xcodeproj` in Xcode to build and run on a simulator or device.

## Project Info

- **Language**: Swift 5.0
- **Framework**: SwiftUI + SwiftData
- **Min iOS Target**: 26.2 (latest)
- **Bundle ID**: `hippl.baby-rhythm-app`
- **Xcode**: Uses `PBXFileSystemSynchronizedRootGroup` — new Swift files placed in the source folder are automatically picked up without editing `project.pbxproj`

## Architecture

MVVM with service layer. All business logic lives in `Services/`; views call services only through ViewModels.

```
baby-rhythm-app/
  baby_rhythm_appApp.swift   — @main, sets up ModelContainer([Child, SleepSession])
  ContentView.swift          — RootView: routes to OnboardingView or HomeView based on Child records

  Models/
    Child.swift              — @Model: id, name, DOB, estimatedDueDate?, schedule prefs, photoFilename
    SleepSession.swift       — @Model: startTime, endTime, type (nap/nightSleep), pause tracking
    DailyDigest.swift        — Codable struct for AI-generated sleep summary (cached in UserDefaults)
    SleepSound.swift         — SleepSound enum + SoundDuration enum

  Services/
    SleepScheduleService.swift — Age calculation (corrected age for preemies), hardcoded schedule table
    WakeWindowService.swift    — Wake window midpoint + adjustment, RiskLevel, nextNapTime, bedtime (Rule A/B)
    NotificationService.swift  — Schedules 3 local notifications: nap window (−15 min), overtiredness, bedtime
    DigestService.swift        — Claude API call for morning narrative; caches to UserDefaults
    SleepSoundService.swift    — AVAudioPlayer wrapper, fade-out, countdown timer

  ViewModels/
    HomeViewModel.swift               — Active session state, awake timer, endSleep() triggers notifications + digest
    ChildProfileViewModel.swift       — Edit child name/DOB/EDD/photo/schedule prefs
    SchedulePreferencesViewModel.swift — Standalone schedule preference editing
    DigestViewModel.swift             — Privacy gate, generate/refresh digest, isGenerating state
    SleepSoundViewModel.swift         — Mirrors SleepSoundService state for SwiftUI binding

  Views/
    Onboarding/OnboardingView.swift   — Single-page form; saves Child, requests notification permission
    Home/HomeView.swift               — Main screen: header, NextNapCard, sleep controls, sounds card, summary
    Home/NextNapCardView.swift        — Countdown, risk badge with tooltip, bedtime label
    Home/DigestCardView.swift         — Morning AI digest card with goal bars and shimmer loading
    SleepTimer/SleepTimerView.swift   — Sheet modal: monospace timer, pause/resume, end sleep
    History/HistoryView.swift         — Day-by-day timeline with session rows and summary footer
    Profile/ChildProfileView.swift    — Edit child profile (photo, personal info, schedule windows)
    Settings/SchedulePreferencesView.swift — Wake-up / bedtime window pickers
    Sounds/SleepSoundsView.swift      — Sound + duration selector, volume slider, waveform indicator

  Shared/
    DesignSystem.swift       — Color tokens (brPrimary, brSurface…), DS.Spacing/Radius, PrimaryButton, SecondaryButton, cardStyle()

  Config.swift               — Anthropic API key + model name (claude-haiku-4-5-20251001)
```

## Key Patterns

**SwiftData**: `@Query` in views for reactive fetching; ViewModels receive `ModelContext` via `init` and call `modelContext.fetch()` for filtered queries (SwiftData `#Predicate` with captured `UUID` values).

**Timer continuity across background**: `SleepSession` stores `startTime` and accumulated `pausedDuration`. `elapsedLive(now:)` recomputes elapsed from wall time — no background tasks needed.

**Corrected age (preemies)**: `SleepScheduleService.birthReference(for:)` uses EDD as birth reference when `EDD > DOB` and corrected age < 12 months.

**Wake window adjustments**: midpoint of age bracket ± 15 min (short nap < 30 min) / +10 min (long nap > 90 min). Rule A clips first-nap time to preferred wake window; Rule B clips bedtime ±20 min max.

**Notification flow**: after `endSleep()` → cancel all → schedule 3 notifications. On `startSleep()` → cancel nap + overtiredness notifications.

**AI Digest**: `DigestService` calls Claude API between 05:00–11:00, rate-limited to once per 4 hours on manual refresh. Privacy disclosure shown before first generation. Narrative cached in `UserDefaults`.

## Localisation

All user-facing strings use `String(localized:)` or `LocalizedStringKey`. Strings catalog at `Localizable.xcstrings`. English-first; architecture supports UK/EN/FR/DE/IT/ES (never add Russian).
