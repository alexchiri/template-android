# Android Template Repository

This is a template repository for Android projects with Claude Code skills.

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
- Package name (e.g., `com.example.myapp`)
- App name (e.g., `My App`)
