# Android Template Repository

This is a template repository for Android projects with Claude Code skills.

## Package Naming Convention

All Android apps created from this template must use package names starting with `com.kroslabs`. For example:
- `com.kroslabs.myapp`
- `com.kroslabs.apps.calculator`
- `com.kroslabs.tools.filemanager`

## App Conventions

### Settings Screen

All apps must include a settings screen that displays:
- **App version** - Show the `versionName` from `build.gradle.kts` (e.g., "Version 1.2.3"). This is the version that gets bumped by the `/build-workflow` GitHub Action.

To retrieve the version programmatically:
```kotlin
val versionName = packageManager.getPackageInfo(packageName, 0).versionName
```

The settings screen should also provide access to the Debug Logs screen (see `/debug-logs` skill).

### AI Integration

When implementing AI functionality in the app, use **Claude Sonnet 4.5** as the default model:
- **Model ID:** `claude-sonnet-4-5-20250929`
- **API Base URL:** `https://api.anthropic.com/v1/messages`

Example API request:
```kotlin
val requestBody = """
{
    "model": "claude-sonnet-4-5-20250929",
    "max_tokens": 1024,
    "messages": [
        {"role": "user", "content": "Hello, Claude!"}
    ]
}
""".trimIndent()
```

Store the API key securely (e.g., in encrypted SharedPreferences or Android Keystore) and never hardcode it in the source code.

## Spec ID Extraction

Before running the `/build-workflow` skill, search for a `SPEC.md` file in the project folder. If found, look for a spec ID in an HTML comment at the very top of the file:

```html
<!-- SPEC-ID: my-fitness-app-clx2ab3c -->
```

If a spec ID is found:
1. Extract the slug value (e.g., `my-fitness-app-clx2ab3c`)
2. Use this slug when running the `/build-workflow` skill - it will be passed as the `slug` parameter in the upload call

The spec ID format is: `<!-- SPEC-ID: <slug> -->` where `<slug>` is typically a kebab-case identifier.

## Available Skills

### /build-workflow

Creates a GitHub Actions workflow for building and uploading Android APKs. The workflow includes:

- Manual trigger with version bump options (patch, minor, major)
- Optional release notes
- JDK 17 setup with Temurin distribution
- Gradle caching
- Automatic semantic version bumping
- Git tagging and committing version changes
- Keystore-based APK signing
- Upload to app hosting service (includes optional `slug` parameter from SPEC.md)

**Required Secrets:**
- `KEYSTORE_BASE64` - Base64 encoded keystore file
- `KEYSTORE_PASSWORD` - Keystore password
- `KEY_ALIAS` - Signing key alias
- `KEY_PASSWORD` - Signing key password
- `APP_HOSTING_API_KEY` - API key for app hosting service
- `APP_HOSTING_URL` - URL of the app hosting service

**Usage:**
Run `/build-workflow` and provide:
- Package name (must start with `com.kroslabs`, e.g., `com.kroslabs.myapp`)
- App name (e.g., `My App`)
- Slug (optional) - If a `SPEC.md` file exists with a spec ID, extract and pass it as the `slug` parameter in the upload call

### /debug-logs

Creates a debug logs screen and adds comprehensive debug logging throughout the app. Features include:

- In-app log viewer with color-coded log levels (verbose, debug, info, warn, error)
- Timestamps for each log entry
- Log lines wrap for easier reading (no horizontal scrolling)
- Copy logs to clipboard functionality
- Clear logs functionality
- Auto-scroll to latest logs
- Support for both Jetpack Compose and XML Views
- Thread-safe logging with a DebugLogger singleton
- **Automatic logging integration across the entire app**
- **1-hour log retention** - Logs older than 1 hour are automatically pruned to prevent filling device storage

**Components Created:**
- `DebugLogger` - Singleton wrapper around `android.util.Log` that captures all logs
- `DebugLogsScreen` (Compose) or `DebugLogsActivity` (XML) - UI for viewing logs
- Required layout and menu resources (XML Views only)

**Logging Added To:**
- Activity/Fragment lifecycle methods (onCreate, onResume, onPause, onDestroy)
- ViewModel initialization and data loading
- Network/API calls (request start, success, and failures)
- User interactions (button clicks, form submissions)
- Data operations (database reads/writes)
- Repository methods (cache hits/misses, network fetches)
- Navigation events

**Usage:**
Run `/debug-logs` and provide:
- Package name (must start with `com.kroslabs`, e.g., `com.kroslabs.myapp`)
- UI framework preference (Jetpack Compose or XML Views)

The skill will create the debug logging infrastructure, replace existing `Log.*` calls with `DebugLogger.*`, and add comprehensive logging throughout the app's functionality.
