# Financisto Holo

> **Old-school, no-cloud, offline-first personal finance manager for Android.**  
> Everything stays on your device. Cloud backup (Google Drive / Dropbox) is entirely optional.

[![License: GPL v2](https://img.shields.io/badge/License-GPLv2-blue.svg)](license.txt)
[![Android](https://img.shields.io/badge/Android-5.0%2B-green.svg)](https://developer.android.com)
[![minSdk](https://img.shields.io/badge/minSdk-21-brightgreen)](app/build.gradle)
[![targetSdk](https://img.shields.io/badge/targetSdk-36-brightgreen)](app/build.gradle)

**[Get it on Google Play](https://play.google.com/store/apps/details?id=tw.tib.financisto)**

<p>
<img alt="Account list" src="docs/screenshots/accounts.png" width="24%" />
<img alt="Blotter" src="docs/screenshots/blotter.png" width="24%" />
<img alt="Transaction" src="docs/screenshots/transaction.png" width="24%" />
<img alt="Entity Autocomplete" src="docs/screenshots/autocomplete.png" width="24%" />
</p>

---

## Table of Contents

1. [Project Introduction](#project-introduction)
2. [Features](#features)
3. [System Architecture Overview](#system-architecture-overview)
4. [Quick Start](#quick-start)
5. [Configuration](#configuration)
6. [Common Operations](#common-operations)
7. [Troubleshooting](#troubleshooting)
8. [Contributing / Development Guide](#contributing--development-guide)
9. [License](#license)

---

## Project Introduction

**Financisto Holo** is a fork of the original [Financisto](https://github.com/dsolonenko/financisto) app (originally hosted on Launchpad). It started as an interim solution while waiting for a proper v2, and has since accumulated a number of improvements tailored for daily use:

- Holo / partial Material Design theme
- Native Android date/time pickers
- Tweaked text layout with device text scaling support
- Search by memo text and amount (including range search)
- SMS templates evolved into **Notification templates** — parse push notifications from banks and other apps to automatically create transactions
- Backup format fully compatible with Play Store version 1.7.1
- Location and photo features removed (upstream API breakage)

> **⚠️ BE SURE TO BACKUP YOUR DATA REGULARLY!**

Companion scripts for hledger export, Taiwan EasyCard import, and Taiwan Government Unified Invoice import:  
👉 https://github.com/tiberiusteng/financisto-backup-to-hledger

Original author's latest work:  
👉 https://github.com/dsolonenko/financisto

---

## Features

| Feature | Description |
|---|---|
| **Account Management** | Cash, bank, credit card, electronic payment, and custom account types |
| **Transaction Blotter** | Record income / expense / transfer; split transactions; full-text search by memo, payee, amount range |
| **Budget Tracking** | Set budgets per category and time period; visualise remaining balance |
| **Reports** | Pie charts and bar charts by category, payee, project, period, location |
| **Scheduled Transactions** | RFC 2445 iCal recurrence rules, alarm-based reminders |
| **Multi-currency & Exchange Rates** | Auto-download from FreeCurrencyAPI or OpenExchangeRates |
| **Backup & Restore** | Local storage; optional Google Drive and Dropbox backup |
| **Export** | QIF (compatible with desktop finance apps), CSV |
| **Notification Templates** | Parse push notifications to auto-create transactions |
| **Home Screen Widgets** | 2×1, 3×1, 4×1 account balance widgets |
| **Biometric Lock** | Fingerprint / face unlock |
| **Custom Attributes** | Define extra fields per transaction category |

---

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Android Device                          │
│                                                             │
│  ┌──────────────────┐   ┌──────────────────────────────┐   │
│  │  UI Layer        │   │  Background Services          │   │
│  │  (Activities /   │   │  ┌─────────────────────────┐ │   │
│  │   Fragments)     │   │  │ IntentService            │ │   │
│  │  ┌────────────┐  │   │  │ NotificationListener     │ │   │
│  │  │ MainActivity│  │   │  │ RecurrenceScheduler      │ │   │
│  │  │ Blotter    │  │   │  │ DailyAutoBackupScheduler │ │   │
│  │  │ Reports    │  │   │  └─────────────────────────┘ │   │
│  │  │ Budget     │  │   └──────────────────────────────┘   │
│  │  └─────┬──────┘  │                                      │
│  └────────┼─────────┘   ┌──────────────────────────────┐   │
│           │             │  Export / Sync Layer          │   │
│           ▼             │  ┌───────────┐ ┌───────────┐  │   │
│  ┌─────────────────┐    │  │Google Drive│ │  Dropbox  │  │   │
│  │  DatabaseAdapter│    │  └───────────┘ └───────────┘  │   │
│  │  (SQLite CRUD)  │    │  ┌───────────┐ ┌───────────┐  │   │
│  └────────┬────────┘    │  │    QIF    │ │    CSV    │  │   │
│           │             │  └───────────┘ └───────────┘  │   │
│           ▼             └──────────────────────────────┘   │
│  ┌─────────────────┐                                        │
│  │ financisto.db   │  ← SQLite, DB version 231              │
│  │ (local storage) │                                        │
│  └─────────────────┘                                        │
└─────────────────────────────────────────────────────────────┘
```

### Key Packages

```
app/src/main/java/tw/tib/financisto/
├── activity/    UI entry points (Activities & Fragments)
├── adapter/     RecyclerView / ListView adapters
├── backup/      Backup & restore logic
├── blotter/     Transaction list filters & helpers
├── db/          SQLite data access layer
├── export/      QIF, CSV, Dropbox, Google Drive export
├── model/       Data models (Account, Transaction, Budget, …)
├── rates/       Exchange rate providers & downloaders
├── recur/       iCal recurrence rule engine
├── report/      Report data aggregation
├── service/     Background services & schedulers
├── utils/       General-purpose utilities
└── worker/      WorkManager workers (auto-backup)
```

---

## Quick Start

### Prerequisites

| Tool | Version |
|---|---|
| Android Studio | Hedgehog (2023.1) or later |
| JDK | 17 |
| Android SDK Build-Tools | 36 |
| Android device / emulator | API 21+ |

### Method 1 — Build & Run from Android Studio (Recommended)

```bash
# 1. Clone the repository
git clone https://github.com/kedych/financisto1-holo.git
cd financisto1-holo

# 2. Open in Android Studio
#    File → Open → select the cloned directory

# 3. Let Gradle sync complete automatically

# 4. Connect a device (USB debugging enabled) or start an emulator

# 5. Run the app
#    Click the ▶ Run button, or press Shift+F10
```

### Method 2 — Build APK from Command Line

```bash
# Clone
git clone https://github.com/kedych/financisto1-holo.git
cd financisto1-holo

# Build debug APK
./gradlew assembleDebug

# The APK will be at:
# app/build/outputs/apk/debug/app-debug.apk

# Install directly to connected device
./gradlew installDebug
```

### Method 3 — Build Release APK

> **Note:** The current `build.gradle` uses the debug signing config for release builds as a placeholder. For distribution, replace with your own keystore.

```bash
# 1. Create a keystore (one-time setup)
keytool -genkeypair -v \
  -keystore my-release-key.jks \
  -keyalg RSA -keysize 2048 \
  -validity 10000 \
  -alias my-key-alias

# 2. Add signing config to app/build.gradle (see Configuration section)

# 3. Build release APK
./gradlew assembleRelease

# Output: app/build/outputs/apk/release/app-release.apk
```

> **Docker / Docker Compose:** This is an Android app — it must be built with the Android SDK toolchain and run on an Android device or emulator. Docker-based builds are possible using `mobiledevops/android-sdk` or similar images, but are not officially supported by this project. A minimal Dockerfile example is provided below for CI use only.

<details>
<summary>Docker CI Build Example (informational)</summary>

```dockerfile
FROM openjdk:17-jdk-slim

RUN apt-get update && apt-get install -y wget unzip && \
    wget -q https://dl.google.com/android/repository/commandlinetools-linux-9477386_latest.zip \
         -O /tmp/cmdtools.zip && \
    mkdir -p /opt/android/cmdline-tools && \
    unzip /tmp/cmdtools.zip -d /opt/android/cmdline-tools && \
    mv /opt/android/cmdline-tools/cmdline-tools /opt/android/cmdline-tools/latest && \
    yes | /opt/android/cmdline-tools/latest/bin/sdkmanager \
        "platforms;android-36" "build-tools;36.0.0" "platform-tools"

ENV ANDROID_HOME=/opt/android
ENV PATH=$PATH:$ANDROID_HOME/platform-tools:$ANDROID_HOME/cmdline-tools/latest/bin

WORKDIR /app
COPY . .
RUN ./gradlew assembleDebug
```

</details>

---

## Configuration

### App Settings (in-app)

All configuration is done inside the app via **Menu → Settings (Preferences)**:

| Setting | Location | Description |
|---|---|---|
| Default currency | Settings → Currencies | Set your primary currency |
| Google Drive backup | Settings → Backup → Google Drive | Enable & authenticate |
| Dropbox backup | Settings → Backup → Dropbox | Enable & authenticate |
| Auto backup | Settings → Backup → Auto backup | Daily automatic backup |
| Exchange rate provider | Settings → Exchange Rate | Choose FreeCurrency (default, no key needed) or OpenExchangeRates |
| OpenExchangeRates App ID | Settings → Exchange Rate | Required only if using OpenExchangeRates |
| Biometric lock | Settings → Privacy | Enable fingerprint / face unlock |
| Notification listener | Settings → Notification Templates | Grant notification access permission |

### Exchange Rate Provider Configuration

**FreeCurrencyAPI** (default) — no API key required:

```
Settings → Exchange Rate → Provider → FreeCurrencyAPI
```

**OpenExchangeRates** — requires a free App ID:

```
1. Sign up at https://openexchangerates.org (free tier available)
2. Copy your App ID
3. Settings → Exchange Rate → Provider → OpenExchangeRates
4. Enter your App ID
```

### Signing Config for Release Builds

Add to `app/build.gradle` before building a release APK for distribution:

```groovy
android {
    signingConfigs {
        release {
            storeFile file("path/to/my-release-key.jks")
            storePassword System.getenv("KEYSTORE_PASSWORD")   // use env vars, never hardcode
            keyAlias "my-key-alias"
            keyPassword System.getenv("KEY_PASSWORD")
        }
    }
    buildTypes {
        release {
            signingConfig signingConfigs.release
            // ... existing config
        }
    }
}
```

---

## Common Operations

### Build

```bash
# Debug build
./gradlew assembleDebug

# Release build
./gradlew assembleRelease

# Clean build
./gradlew clean assembleDebug
```

### Install / Run

```bash
# Install debug APK on connected device
./gradlew installDebug

# Uninstall
./gradlew uninstallDebug

# Launch via ADB
adb shell am start -n tw.tib.financisto/.activity.MainActivity
```

### View Logs

```bash
# All app logs
adb logcat -s financisto

# Filter by tag
adb logcat | grep tw.tib.financisto

# Save to file
adb logcat -d > financisto_logcat.txt
```

### Run Tests

```bash
# Instrumented tests (requires connected device/emulator)
./gradlew connectedAndroidTest

# View test results
# app/build/reports/androidTests/connected/index.html
```

### Lint

```bash
# Run Android Lint
./gradlew lint

# View report
# app/build/reports/lint-results-debug.html
```

### Database — Schema Migration

Database migrations are handled automatically by `DatabaseSchemaEvolution` on app startup using SQL scripts located in:

```
app/src/main/assets/db/
```

Script naming convention: `update_<from_version>_to_<to_version>.sql`

No manual migration steps are required for end users. For developers adding schema changes:

1. Increment `DATABASE_VERSION` in `app/src/main/java/tw/tib/financisto/db/Database.java`
2. Add a new SQL script in `app/src/main/assets/db/`
3. Update `DatabaseSchemaEvolution` if new view definitions are needed

### Backup & Restore (Manual)

```bash
# Pull database from device (requires adb root or backup permission)
adb exec-out run-as tw.tib.financisto cat databases/financisto.db > financisto_backup.db

# In-app backup (recommended):
# Menu → Backup/Restore → Backup to Storage
```

---

## Troubleshooting

### 1. Gradle sync fails with "SDK location not found"

**Cause:** `local.properties` missing or `sdk.dir` not set.

**Fix:**
```bash
# Create/edit local.properties in project root
echo "sdk.dir=/path/to/your/Android/sdk" > local.properties
# Example on macOS: sdk.dir=/Users/username/Library/Android/sdk
# Example on Linux: sdk.dir=/home/username/Android/Sdk
```

---

### 2. Build fails: "Unsupported class file major version"

**Cause:** Wrong JDK version. The project requires JDK 17.

**Fix:**
```bash
# Check current Java version
java -version

# In Android Studio: File → Project Structure → SDK Location → JDK Location
# Set to JDK 17 path
```

---

### 3. App crashes on launch: "Database version X is newer than Y"

**Cause:** Installing an older APK on a device that already has a newer database version.

**Fix:** Uninstall the existing app first (or use a fresh emulator), then install:
```bash
adb uninstall tw.tib.financisto
./gradlew installDebug
```

---

### 4. Google Drive backup fails / "Sign-in required"

**Cause:** OAuth token expired or Google Play Services not available on the emulator.

**Fix:**
- Use a physical device with Google Play Services for Drive testing
- In the app: Settings → Backup → Google Drive → Disconnect, then reconnect
- Ensure the device has an active internet connection

---

### 5. Exchange rates not updating

**Cause:** Network error, or OpenExchangeRates App ID is missing/invalid.

**Fix:**
- Check internet connectivity on the device
- If using OpenExchangeRates: Settings → Exchange Rate → verify App ID is correct
- Switch to FreeCurrencyAPI (default, no key needed): Settings → Exchange Rate → FreeCurrencyAPI

---

### 6. Notification templates not detecting transactions

**Cause:** Notification Listener permission not granted.

**Fix:**
1. On the device: Settings → Apps → Special App Access → Notification Access
2. Enable **Financisto** in the list
3. In Financisto: Settings → Notification Templates → verify templates are configured

---

### 7. Widgets not updating / showing wrong balance

**Cause:** App background process killed by battery optimiser.

**Fix:**
1. Device Settings → Battery → App Battery Usage → Financisto → set to "Unrestricted"
2. Long-press the widget → Remove, then re-add it from the widget picker
3. Or send a manual update: `adb shell am broadcast -a tw.tib.financisto.UPDATE_WIDGET`

---

### 8. Backup file not found after export

**Cause:** On Android 10+, scoped storage restrictions may prevent writing to expected directories.

**Fix:**
- Grant "All files access" (MANAGE_EXTERNAL_STORAGE) if prompted
- Use the in-app file picker to select a destination folder
- Use Google Drive or Dropbox backup as an alternative

---

### 9. "MultiDex" error on old devices (API 21)

**Cause:** App exceeds 65,536 method limit — handled by MultiDex.

**Fix:** Ensure `multiDexEnabled = true` is set in `app/build.gradle` (it already is). If issues persist, add to your `Application` class:
```java
@Override
protected void attachBaseContext(Context base) {
    super.attachBaseContext(base);
    MultiDex.install(this);
}
```

---

### 10. `annotationProcessor` error during build

**Cause:** AndroidAnnotations or EventBus annotation processor misconfiguration.

**Fix:**
```bash
./gradlew clean
# Then rebuild. If the issue persists, invalidate caches in Android Studio:
# File → Invalidate Caches → Invalidate and Restart
```

---

## Contributing / Development Guide

### Prerequisites

- Android Studio (latest stable)
- JDK 17
- Android SDK API 36

### Branch Strategy

```
main          ← stable, production-ready
feature/*     ← new features (e.g. feature/add-dark-mode)
fix/*         ← bug fixes (e.g. fix/widget-refresh-crash)
```

### Commit Message Convention

Follow the format:
```
<type>: <short description>

[optional body]
```

Types: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`

Examples:
```
feat: add range search for transaction amounts
fix: correct widget balance display after currency change
docs: update README with Docker build instructions
```

### Code Style

- Follow standard Android / Java code style
- Use 4-space indentation (no tabs)
- Max line length: 120 characters
- Add Javadoc to public methods
- Prefer AndroidAnnotations conventions for DI (`@EBean`, `@EActivity`, etc.)

### Making Changes

```bash
# 1. Fork the repository and clone your fork
git clone https://github.com/<your-username>/financisto1-holo.git
cd financisto1-holo

# 2. Create a feature branch
git checkout -b feature/your-feature-name

# 3. Make your changes and build
./gradlew assembleDebug

# 4. Run lint and tests
./gradlew lint
./gradlew connectedAndroidTest   # requires device/emulator

# 5. Commit
git add .
git commit -m "feat: describe your change"

# 6. Push and open a Pull Request
git push origin feature/your-feature-name
```

### Adding a Database Migration

1. Edit `Database.java` — increment `DATABASE_VERSION`
2. Create `app/src/main/assets/db/update_<old>_to_<new>.sql`
3. Test on a device that already has the previous DB version

### Reporting Issues

Please include:
- Android version and device model
- App version (visible in Settings → About)
- Steps to reproduce
- Logcat output (`adb logcat -d | grep tw.tib.financisto`)

---

## License

This project is licensed under the **GNU General Public License v2.0**.  
See [license.txt](license.txt) for the full text.

Original work Copyright © 2010 Denis Solonenko.  
Modifications Copyright © tiberiusteng and contributors.

---

*README last updated: 2026-03-02*
