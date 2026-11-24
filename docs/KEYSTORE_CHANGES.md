# KEYSTORE changes

Date: 2026-10-14

## Architecture summary

KEYSTORE is a single-module Kotlin/Compose Android app. Its custom `InputMethodService` and companion app share one `KeystoreApp`-owned Room database and `ClipboardRepository`; saved clipboard text is encrypted by `ContentCipher` before it is stored. The keyboard is a custom canvas view, not an XML layout.

## Confirmed by inspection

- The original clipboard toolbar path opened the companion activity instead of inserting saved text in the active editor (`keyboard/KeystoreInputMethodService.kt`).
- Clipboard capture paths did not reject `ClipDescription.EXTRA_IS_SENSITIVE` clips.
- `onStartInput` always reset to alphabetic mode; numeric/phone editors were not recognized.
- Enter only handled search directly; it did not consistently honour configured send/go/done/next/previous actions or `IME_FLAG_NO_ENTER_ACTION`.
- Room destructive-migration fallback risked erasing an unsupported older database.
- Re-copying an existing clip updated `lastCopiedAt`, but the list was ordered by immutable `createdAt`, so the active/recent clip could remain far down the archive.

## Implemented

- Added an in-keyboard clipboard panel. It observes the existing `ClipboardRepository`, uses the existing Room ordering, shows an explicit empty state, and commits the complete selected text through the current `InputConnection`. It neither adds a database nor copies data to a second store. The panel is unavailable in sensitive fields.
- Rejects API-33+ clipboard clips marked `ClipDescription.EXTRA_IS_SENSITIVE` in the IME, foreground monitor and dormant capture-service path.
- Starts number, phone and date/time fields in the numeric layout through the new small, pure `KeyboardInputMode` mapper.
- Uses `EditorInfo` actions directly for search/send/go/done/next/previous. `IME_FLAG_NO_ENTER_ACTION` now inserts a newline instead. Cursor-selection updates finish composing text and refresh suggestions.
- Suppresses delimiter-triggered autocorrection for URL, email and path-like tokens.
- Removed Room's `fallbackToDestructiveMigration(true)`. No schema changed and no migration was added; unsupported old schemas now fail safely instead of silently deleting data.
- Orders clipboard rows by `lastCopiedAt` so the existing deduplication behaviour correctly keeps a re-copied clip at the top of the archive and IME panel.
- Added JVM tests for editor-action selection and input-mode selection.
- Corrected README feature/limitation statements to reflect existing emoji and basic local suggestions.
- Kept the 205,000-word offline asset but caps the live prediction dictionary at 40,000 entries (32,000 bundled plus 8,000 custom/downloaded), caches the personal word/bigram model in memory, removes whole-bucket completion sorting, and halves the typo-correction search set.
- Made autocorrect conservative: learned/known words are preserved, and an unknown word must be at least four letters and one edit from a candidate before it is changed automatically.
- Fits action labels to their actual key width, including browser `search` on narrow or landscape displays.

## Files changed

- `keyboard/ModernKeystoreKeyboardView.kt` — clipboard panel and numeric initial-state support.
- `keyboard/KeystoreInputMethodService.kt` — repository observation, insertion, editor actions, selection handling, sensitive clip exclusion, and autocorrect guard.
- `keyboard/KeyboardEnterBehavior.kt`, `keyboard/KeyboardInputMode.kt` — small pure policy helpers.
- `ClipboardMonitor.kt`, `ClipboardCaptureService.kt` — sensitive clip exclusion.
- `data/AppDatabase.kt` — removed destructive fallback.
- `KeyboardEnterBehaviorTest.kt`, `KeyboardInputModeTest.kt` — policy tests.
- `README.md`, `KEYSTORE_AUDIT.md` — documentation.

## Tests and build results

- **UNVERIFIED:** `./gradlew testDebugUnitTest` could not run because this repository has no `gradlew`/`gradlew.bat` script and neither `gradle` nor `java` is installed on PATH.
- **UNVERIFIED:** A temporary Gradle 8.11.1 download was attempted solely for verification, but the downloaded archive was zero bytes in this environment; Gradle could not start.
- **UNVERIFIED:** Lint and APK build were not run for the same missing-toolchain reason.
- Source inspection only: no emulator/device was available.

## Database, permissions and privacy

- Database migration: none. This change removes a destructive fallback; it does not alter schema or stored rows.
- Manifest/permission changes: none.
- `INTERNET` remains present for the pre-existing optional dictionary-pack downloader. This change adds no network request and sends no clipboard contents over the network.

## Incomplete / requires further product work

- There is no floating keyboard implementation or control.
- Resize persists a clamped height, but there is no explicit reset-to-default control.
- Suggestions are basic offline English/personal-model suggestions. There is no user autocorrect toggle, and URL/email protection is heuristic.
- A clip from a password field cannot always be identified by Android APIs. The implementation rejects explicit API-33+ sensitive-clip metadata and never exposes the archive in sensitive input fields, but app/OEM behaviour must be tested on devices.
- Existing global hash deduplication updates any matching saved item rather than only the newest matching entry; this was not changed because changing it needs product confirmation and a migration/data-behaviour decision.

## Manual verification required

On a physical Android 10+ device, verify mixed typing; selection replacement and long-press delete; letters/numbers/symbols/emoji; search/send/go/done/next and multiline Enter; copy/open/insert long multiline clips; empty clipboard panel; search/star/unstar/delete/counts; app/IME restart persistence; password and API-33 sensitive clipboard exclusion; small/large portrait/landscape resize; and network/log inspection. Test messaging, browser search, notes, numeric, and password fields separately and record device, Android version and target apps.

## Project status log — 24 November 2026

This status snapshot is based on source and project documents inspected on this date; it is not a live task tracker. “Implemented” does not mean every feature has passed device or release testing.

### Done / implemented

- Built the Android companion app and custom Keystore Keyboard.
- Added clipboard capture through app and keyboard paths, subject to Android clipboard-access rules.
- Added a Room database and clipboard archive with search, editing, favorites, trash, and insertion from the keyboard.
- Added duplicate detection: copying identical text updates the existing clipboard entry and its copy count.
- Added AES-GCM encryption for the saved clipboard content field using a key held by Android Keystore. Archive previews and metadata are also stored in the local database.
- Added local keyboard suggestions and settings, including themes and resizing.
- Added a passphrase-based backup encryption codec. Complete in-app backup export and restore are still listed as unfinished in project documentation.
- Replaced the root privacy-policy redirect with a responsive product website at `index.html`. It includes the supplied privacy policy URL and an evidence-based project status section.

### Being verified / release preparation

These are documented outstanding checks, not claims about someone's current activity:

- Physical-device testing of clipboard capture, insertion, persistence, duplicate handling, and sensitive-field behavior is still called for in the project notes. No device-test results were found in this inspection.
- Build/test evidence is mixed: `build-keystore-final.out.log` reports `assembleDebug` and `testDebugUnitTest` succeeded. The 14 October 2026 change note says build and test commands could not be run in that environment. No Android build or tests were run for the website task. Verify current source with a fresh build and test run.
- `docs/PLAY_STORE_RELEASE_CHECKLIST.md` lists store declarations, screenshots, reviewer notes, and device testing as release preparation.
- Project documents need reconciliation: the README describes writing/typing capture and destructive migration as limitations, while current app UI labels the typing archive disabled and the current Room builder has no destructive-migration fallback. The README says Android 10+ although current `minSdk` is 23.
- `docs/PRIVACY_POLICY.md` still contains a placeholder asking for a monitored developer contact email. Verify its other statements against current app behavior.
- The product page is in the repository, but public hosting and deployed links were not verified.

### Next

1. Run the current Android build and unit tests; record date, commands, and results.
2. Test clipboard capture, keyboard insertion, persistence, duplicate handling, and sensitive-field behavior on physical Android devices and relevant apps.
3. Align the README, privacy policy, in-app disclosures, and release checklist with current code. Resolve differences about writing archive, minimum Android version, database migration, and capture.
4. Finish and verify in-app backup export/restore and the database migration strategy.
5. Complete accessibility/tablet and localization checks, Play listing materials, privacy disclosures, and the independent security review called out in project notes.
6. Configure website hosting and verify the live site and its assets.

### Evidence checked

- `app/build.gradle.kts` — app version 0.3.8 and SDK configuration.
- `app/src/main/AndroidManifest.xml` — keyboard service, permissions, and Android backup setting.
- `app/src/main/java/com/keystore/clipboard/MainActivity.kt` — current screens and disclosure text, including disabled typing archive and clipboard limitations.
- `app/src/main/java/com/keystore/clipboard/data/ClipboardRepository.kt` and `data/AppDatabase.kt` — clipboard save/deduplication behavior and database queries.
- `app/src/main/java/com/keystore/clipboard/security/ContentCipher.kt` — saved-content encryption.
- `app/src/main/java/com/keystore/clipboard/BackgroundCaptureController.kt` — background capture is not automatically started.
- `app/src/main/java/com/keystore/clipboard/backup/BackupCodec.kt` — backup encryption codec.
- `README.md`, `docs/KEYSTORE_CHANGES.md`, `docs/KEYSTORE_AUDIT.md`, `docs/PLAY_STORE_RELEASE_CHECKLIST.md`, and `docs/PRIVACY_POLICY.md` — project claims and documented release work.
