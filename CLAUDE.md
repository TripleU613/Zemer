# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Metrolist** is a native Android music streaming application - a YouTube Music client that allows users to stream and download music from YouTube Music without the official app. The project is a fork of InnerTune by Zion Huang and OuterTune by Davide Garberi.

- **Package**: `com.metrolist.music`
- **Min SDK**: 26 (Android 8.0)
- **Target/Compile SDK**: 36
- **License**: GPL v3.0

## Build Commands

The project uses Gradle with Kotlin DSL and requires JDK 21.

### Standard Build Commands

```bash
# Build all variants (5 ABIs × 2 build types = 10 APKs)
./gradlew assemble

# Build specific ABI variants
./gradlew assembleUniversalRelease  # All ABIs in one APK
./gradlew assembleArm64Release      # ARM 64-bit only
./gradlew assembleArmeabiRelease    # ARM 32-bit only
./gradlew assembleX86Release        # x86 32-bit only
./gradlew assembleX86_64Release     # x86 64-bit only

# Build debug variants
./gradlew assembleUniversalDebug
./gradlew assembleArm64Debug
# etc.

# Clean build
./gradlew clean
```

### Linting

```bash
# Run lint on default variant
./gradlew lint

# Run lint on specific variants
./gradlew lintUniversalDebug
./gradlew lintUniversalRelease
./gradlew lintArm64Release

# Run lint and apply safe fixes
./gradlew lintFix
```

### Testing

```bash
# Run all unit tests
./gradlew test

# Run instrumented tests on connected device
./gradlew connectedCheck
./gradlew connectedUniversalDebugAndroidTest

# Run all checks (lint + tests)
./gradlew check
```

### Development Build

For rapid development iteration, use the universal debug variant:

```bash
./gradlew assembleUniversalDebug
```

The debug APK will be at: `app/build/outputs/apk/universal/debug/app-universal-debug.apk`

## Architecture

### Tech Stack

- **Language**: Kotlin with coroutines
- **UI**: Jetpack Compose with Material 3 (declarative UI)
- **Architecture**: MVVM with single-activity navigation
- **DI**: Hilt for dependency injection
- **Database**: Room (schema version 25 with 24 migrations)
- **Media**: Media3 (ExoPlayer) with MediaSession for playback
- **Networking**: Ktor client with OkHttp engine
- **Image Loading**: Coil 3
- **Async**: Kotlin Coroutines + Flow

### Multi-Module Structure

The project is organized into multiple Gradle modules:

1. **app** - Main Android application module
   - All UI components (Compose screens, ViewModels, themes)
   - Room database with DAOs and entities
   - Media playback service
   - Hilt DI modules

2. **innertube** - YouTube InnerTube API client (pure Kotlin/JVM)
   - Handles communication with YouTube Music's private API
   - Independent of Android framework
   - Uses Ktor + OkHttp + custom NewPipe Extractor fork (`com.github.mostafaalagamy:MetrolistExtractor`)

3. **lrclib** - LRCLib lyrics provider (pure Kotlin/JVM)
   - Open-source synced lyrics database
   - Uses Ktor client

**Note**: All non-app modules are pure JVM/Kotlin with no Android dependencies, making them independently testable.

### Key Directories

```
app/src/main/kotlin/com/metrolist/music/
├── App.kt                    # Application class with Hilt setup
├── MainActivity.kt           # Single activity with Compose
├── db/                       # Room database
│   ├── MusicDatabase.kt      # Database definition (25 migrations!)
│   ├── entities/             # Database entities (Song, Artist, Album, Playlist, etc.)
│   └── DatabaseDao.kt        # All database queries
├── di/                       # Hilt dependency injection modules
├── playback/                 # Media3 playback service
│   ├── MusicService.kt       # MediaLibraryService implementation
│   ├── PlayerConnection.kt   # Service binding
│   ├── DownloadUtil.kt       # Download management
│   ├── ExoDownloadService.kt # Media3 download service
│   └── queues/               # Queue management
├── ui/
│   ├── component/            # Reusable Compose components
│   ├── screens/              # Feature screens (Home, Library, Search, etc.)
│   ├── player/               # Music player UI
│   ├── menu/                 # Context menus and dialogs
│   └── theme/                # Material 3 theming
├── viewmodels/               # ViewModels for screen state management
├── models/                   # Data models and DTOs
├── extensions/               # Kotlin extension functions
├── utils/                    # Utility classes
└── constants/                # App constants

app/schemas/                  # Room database schema exports for testing
```

### Database Architecture

The Room database (`song.db`) is the core of the app with **25 schema versions**. It uses:

- **16 Entities**: SongEntity, ArtistEntity, AlbumEntity, PlaylistEntity, LyricsEntity, FormatEntity, Event, PlayCountEntity, SearchHistory, ArtistWhitelistEntity, SetVideoIdEntity, plus 5 mapping tables (SongArtistMap, SongAlbumMap, AlbumArtistMap, PlaylistSongMap, RelatedSongMap)
- **3 Views**: SortedSongArtistMap, SortedSongAlbumMap, PlaylistSongMapPreview
- **24 Migrations**: 1 manual migration (MIGRATION_1_2) and 23 AutoMigrations from version 2 to 25
- **DatabaseDao**: Main DAO with 1601 lines of query methods - handles all database operations
- **Features**: Artist whitelist filtering, format caching, play count tracking, lyrics storage, event tracking

Schema exports are located in `app/schemas/` and configured via KSP:
```kotlin
ksp {
    arg("room.schemaLocation", "$projectDir/schemas")
}
```

### Dependency Injection

Hilt is used throughout the app. Key modules in `app/src/main/kotlin/com/metrolist/music/di/`:
- **AppModule.kt** - Provides MusicDatabase singleton, application scope, player/download caches
- **NetworkModule.kt** - Provides network connectivity observer
- **Qualifiers.kt** - DI qualifiers (@PlayerCache, @DownloadCache, @ApplicationScope)
- **LyricsHelperEntryPoint.kt** - Entry point for lyrics helpers

## Build Configuration

### Build Flavors

The app has 5 ABI-specific product flavors under the "abi" dimension:
- `universal` - All ABIs (armeabi-v7a, arm64-v8a, x86, x86_64)
- `arm64` - ARM 64-bit only
- `armeabi` - ARM 32-bit only
- `x86` - x86 32-bit only
- `x86_64` - x86 64-bit only

Each flavor gets a `BuildConfig.ARCHITECTURE` field with the ABI name.

### Release Builds

Release builds have:
- ProGuard minification enabled (`proguard-rules.pro`)
- Resource shrinking enabled
- PNG crunching disabled for faster builds
- Code signing via GitHub Secrets in CI/CD

### Environment Variables

For CI/CD, the following GitHub Secrets are required for signing and releases:
- `KEYSTORE` (base64-encoded release keystore)
- `KEY_ALIAS` (signing key alias)
- `STORE_PASSWORD` (keystore password)
- `KEY_PASSWORD` (key password)
- `DEBUG_KEYSTORE` (base64-encoded, for persistent debug signing)
- `RELEASE_TOKEN` (GitHub token for creating releases)

### Core Library Desugaring

The app uses Java 8+ APIs on older Android versions via desugaring:
```kotlin
compileOptions {
    isCoreLibraryDesugaringEnabled = true
}
```

## Development Workflow

### CI/CD

GitHub Actions workflows in `.github/workflows/`:

1. **build.yml** - Builds all ABI variants on every push
   - Runs lint checks
   - Signs APKs with GitHub Secrets
   - Uploads artifacts

2. **release.yml** - Automated releases on version bumps
   - Triggered when `versionCode` changes in `app/build.gradle.kts`
   - Creates GitHub releases with all ABI APKs

3. **build_pr.yml** - Pull request validation
   - Builds universal debug APK only
   - Runs lint checks
   - Uploads APK with PR number

4. **build-arm64.yml** - ARM64-specific workflow (manual trigger only)

### Versioning

Version is managed in `app/build.gradle.kts`:
```kotlin
versionCode = 128
versionName = "12.7.0"
```

Release workflow automatically detects version changes and creates releases.

### Translations

The app uses Weblate for translations (60+ languages):
- Translations: https://hosted.weblate.org/projects/Metrolist/
- Auto-generates locale config via `generateLocaleConfig = true`
- Translation status badge in README

### Signing

The app uses different keystores for debug and release builds:

- **Release Builds**: Use production keystore from `KEYSTORE` GitHub Secret (base64-encoded)
- **Debug Builds (Local)**: Use persistent keystore (`app/persistent-debug.keystore`) for consistent signing across builds and devices
- **Debug Builds (PR)**: Use standard Android debug keystore for pull request validation
- **CI/CD**: Persistent debug keystore is restored from `DEBUG_KEYSTORE` secret

The persistent debug keystore ensures consistent app signatures during local development, avoiding reinstallation when switching between machines or builds.

## Important Development Notes

### Media3 Integration

The app uses Media3 (ExoPlayer) for playback with:
- **MusicService.kt** - MediaLibraryService implementation with custom audio processing
- MediaSession for media controls and Android Auto support
- OkHttp integration for network streaming
- Custom audio processors: SonicAudioProcessor (tempo/pitch), SilenceSkippingAudioProcessor, LoudnessEnhancer
- Separate player cache and download cache
- Background playback with notification controls
- Audio offload support for battery efficiency

### YouTube InnerTube API

The `innertube` module contains the YouTube Music API client. This is a private, undocumented API that may change without notice. The module is pure Kotlin/JVM with no Android dependencies.

### Compose Navigation

The app uses a single-activity architecture with Compose Navigation. All screens are Composables, and navigation is handled declaratively.

### DataStore Preferences

User preferences are stored using Jetpack DataStore (not SharedPreferences). See `app/src/main/kotlin/com/metrolist/music/constants/PreferenceKeys.kt` and related DataStore utilities for preference keys.

### Artist Whitelist System

A critical architectural feature that filters all content by whitelisted artists:

- **Sync on Launch**: Blocks app startup to ensure whitelist is current (`SyncUtils.kt`)
- **Background Sync**: Runs hourly to check for whitelist updates
- **Hash-Based Detection**: Only syncs when remote whitelist changes (efficient)
- **Data Source**: Fetches artist JSON from GitHub via `WhitelistFetcher.kt`
- **Aggressive Cleanup**: Automatically deletes songs, albums, and playlists when artists are removed from whitelist
- **Progress Tracking**: `WhitelistSyncProgress` StateFlow for UI updates
- **Database Integration**: Uses `ArtistWhitelistEntity` with Room

This feature ensures only approved content appears in the app and may impact initial launch time.

### Lyrics Architecture

The app uses a multi-provider lyrics system with automatic fallback:

- **LRCLib Module** (`lrclib/`) - Open-source synced lyrics database (primary source)
- **YouTubeSubtitleLyricsProvider** - Extracts official YouTube captions/subtitles
- **YouTubeLyricsProvider** - Fetches lyrics from YouTube Music's lyrics feature
- **LyricsHelper** (`utils/LyricsHelper.kt`) - Aggregates all providers with priority-based fallback
- **Romanization Support** - Can provide romanized lyrics for non-Latin scripts
- **Database Storage** - Cached in `LyricsEntity` for offline access
- **LyricsEntry Model** - Unified data model for all lyric types (synced/plain)

The LyricsHelper tries providers in order until lyrics are found, ensuring maximum coverage.

### Room Migrations

The database has gone through 25 schema versions. When modifying entities:
1. Increment the version number in `@Database(version = X)`
2. Add an AutoMigration or manual Migration
3. Run the app to generate the new schema in `app/schemas/`
4. Commit the schema JSON file

### Timber Logging

The app uses Timber for logging. Use `Timber.d()`, `Timber.e()`, etc. instead of `Log`.

### Material 3 Dynamic Theming

The app supports Material 3 dynamic color (Material You) on Android 12+, plus custom light/dark/black themes. Uses **materialKolor** library for enhanced dynamic color generation across all Android versions.

### Key Dependencies

Beyond the standard Android libraries, the app uses several specialized dependencies:

- **materialKolor** - Advanced Material 3 dynamic color theming (supports pre-Android 12)
- **squigglyslider** - Custom slider component for UI controls
- **shimmer** - Loading state skeleton animations
- **ucrop** - Image cropping library for cover art editing
- **guava** - Google Core Libraries for Java (used for utilities)
- **concurrent-futures** - Kotlin coroutine integration with Java futures

### Android Auto & TV

The app has Android Auto support via MediaSession and Android TV support via leanback UI. MediaSession configuration is critical for these features.

### Region Restrictions

YouTube Music is not available in all regions. Users may need a VPN/proxy to use the app in unsupported regions.

## Project Documentation

This repository does not have a README.md file with user-facing documentation - all developer documentation is maintained in this CLAUDE.md file. The project is likely used internally or in a private context.

## Common Development Patterns

### Adding a New Screen

1. Create a Composable in `app/src/main/kotlin/com/metrolist/music/ui/screens/`
2. Create a ViewModel in `viewmodels/` if needed
3. Add navigation route in the navigation graph
4. Use Hilt `@Inject` for dependencies

### Adding a Database Entity

1. Create entity in `db/entities/`
2. Add to `@Database(entities = [...])`
3. Add queries to `DatabaseDao`
4. Create a migration in `MusicDatabase.kt`
5. Increment database version
6. Build to generate schema

### Adding a Dependency

Dependencies are managed via version catalog in `gradle/libs.versions.toml`. Add entries there, then reference in `build.gradle.kts`:
```kotlin
implementation(libs.your.dependency)
```

### Working with InnerTube API

The `innertube` module handles YouTube Music API calls. Key classes:
- YouTube client initialization
- Search, browse, playback URL fetching
- Account authentication

When making changes, ensure the module remains Android-independent.
