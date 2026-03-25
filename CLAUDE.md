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
- **Framework**: SwiftUI
- **Min iOS Target**: 26.2
- **Bundle ID**: `hippl.baby-rhythm-app`

## Architecture

Minimal SwiftUI app with two source files:

- `baby_rhythm_appApp.swift` — `@main` entry point, declares `WindowGroup { ContentView() }`
- `ContentView.swift` — root view; all UI starts here

Assets live in `Assets.xcassets` (app icon + accent color).
