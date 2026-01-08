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
- Upload to app hosting service

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

### /debug-logs

Creates a debug logs screen accessible from the app's settings screen where all application logs can be viewed. Features include:

- In-app log viewer with color-coded log levels (verbose, debug, info, warn, error)
- Timestamps for each log entry
- Copy logs to clipboard functionality
- Clear logs functionality
- Auto-scroll to latest logs
- Support for both Jetpack Compose and XML Views
- Thread-safe logging with a DebugLogger singleton

**Components Created:**
- `DebugLogger` - Singleton wrapper around `android.util.Log` that captures all logs
- `DebugLogsScreen` (Compose) or `DebugLogsActivity` (XML) - UI for viewing logs
- Required layout and menu resources (XML Views only)

**Usage:**
Run `/debug-logs` and provide:
- Package name (must start with `com.kroslabs`, e.g., `com.kroslabs.myapp`)
- UI framework preference (Jetpack Compose or XML Views)

After running the skill, add a "Debug Logs" option to your settings screen and replace `Log.*` calls with `DebugLogger.*` to capture logs in the viewer.
