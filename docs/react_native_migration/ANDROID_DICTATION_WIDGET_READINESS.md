 # Android Dictation Widget Readiness
 
 This document lists the exact work needed to make the Android dictation widget production-ready. The work is divided into phases with step-by-step checklists so a junior developer can follow them without missing critical details.
 
 ## Scope
 
 The widget includes:
 - Android native dictation module (Kotlin): speech recognition + audio capture + audio preservation
 - React Native TypeScript service layer and hooks
 - UI controls + waveform visualization
 - Verification on real devices
 
 References:
 - `docs/react_native_migration/08_ANDROID_IMPLEMENTATION.md`
 - `docs/react_native_migration/06_TYPESCRIPT_SERVICE_LAYER.md`
 - `docs/react_native_migration/07_WAVEFORM_COMPONENTS.md`
 - `docs/react_native_migration/09_TESTING_AND_VALIDATION.md`
 
 ---
 
 ## Phase 0 — Repo + Environment Setup
 
 **Goal:** Ensure the Android project builds locally and has all prerequisites.
 
 Checklist:
 - Confirm you can run the Android app on a device or emulator.
 - Confirm Kotlin + Gradle build succeeds before changes.
 - Note the applicationId/package: `com.reactnativedictation` must match module package names.
 - Ensure React Native auto-linking is working (or plan for manual linking if needed).
 
 Output:
 - Clean Android build with no new warnings or errors.
 
 ---
 
 ## Phase 1 — Native Module Skeleton (Android)
 
 **Goal:** Create a working RN native module entry point.
 
 Files:
 - `android/src/main/java/com/reactnativedictation/DictationPackage.kt`
 - `android/src/main/java/com/reactnativedictation/DictationModule.kt`
 
 Checklist:
 - Create `DictationPackage` that returns `DictationModule`.
 - Ensure `DictationModule`:
   - Extends `ReactContextBaseJavaModule`
   - Exposes `initialize`, `startListening`, `stopListening`, `cancelListening`, `getAudioLevel`, `normalizeAudio`
   - Emits events: `onResult`, `onStatus`, `onAudioLevel`, `onAudioFile`, `onError`
 - Register `DictationPackage` inside the app's `MainApplication` (or equivalent).
 
 Output:
 - JS can call `DictationModule.initialize()` without crashing.
 - Events can be emitted to JS (even if no real audio yet).
 
 ---
 
 ## Phase 2 — Android Permissions + Manifest
 
 **Goal:** Ensure microphone permissions and speech recognition queries are correct.
 
 Files:
 - `android/src/main/AndroidManifest.xml`
 - `DictationModule.kt`
 
 Checklist:
 - Add permissions:
   - `android.permission.RECORD_AUDIO`
   - `android.permission.INTERNET` (needed for speech recognition on most devices)
 - Add `<queries>` section for `android.speech.RecognitionService`.
 - In `DictationModule`, implement permission request flow:
   - If no permission, request it.
   - Store the pending promise + options.
   - Resume after permission granted.
   - If activity is null, reject with `NO_ACTIVITY`.
 
 Output:
 - First call to `startListening()` prompts for mic permission.
 - If user approves, dictation starts and options are preserved.
 - If user denies, JS receives `NOT_AUTHORIZED`.
 
 ---
 
 ## Phase 3 — Dictation Coordinator + Speech Recognition
 
 **Goal:** Start speech recognition with partial and final results.
 
 Files:
 - `android/src/main/java/com/reactnativedictation/DictationCoordinator.kt`
 
 Checklist:
 - Create `DictationCoordinator` with:
   - `SpeechRecognizer` setup
   - `RecognitionListener` implementation
 - Emit:
   - `emitStatus("ready")` after initialization
   - `emitStatus("listening")` when listening starts
   - `emitStatus("stopped")` or `emitStatus("cancelled")` after stop/cancel
 - Emit `onResult` for:
   - Partial results (`isFinal = false`)
   - Final results (`isFinal = true`)
 - Translate `SpeechRecognizer` errors to user-friendly messages and emit via `onError`.
 
 Output:
 - Partial and final dictation strings arrive in JS with correct `isFinal` flag.
 - Errors are surfaced to JS with `RECOGNITION_ERROR` codes.
 
 ---
 
 ## Phase 4 — Audio Engine + Audio Preservation
 
 **Goal:** Capture microphone audio for waveform display and optional file preservation.
 
 Files:
 - `android/src/main/java/com/reactnativedictation/AudioEngineManager.kt`
 - `android/src/main/java/com/reactnativedictation/AudioEncoderManager.kt`
 - `android/src/main/java/com/reactnativedictation/CanonicalAudioStorage.kt`
 
 Checklist:
 - Implement `AudioEngineManager` using `AudioRecord`.
 - Compute smoothed audio levels (0..1 range).
 - When `preserveAudio` is true:
   - Create a canonical file path via `CanonicalAudioStorage`
   - Encode audio to `.m4a` (AAC) via `AudioEncoderManager`
 - On stop:
   - Return duration, size, sample rate, channels
 - On cancel:
   - Delete the file when `deleteAudioIfCancelled = true`
 
 Output:
 - `onAudioLevel` emits at ~30 FPS.
 - `onAudioFile` emits with accurate metadata when preserving audio.
 - `.m4a` file plays back correctly on device.
 
 ---
 
 ## Phase 5 — JS/TS Service Layer + Hooks
 
 **Goal:** Provide a stable JS API for the widget.
 
 Files (expected):
 - `src/services/DictationService.ts`
 - `src/hooks/useDictation.ts`
 - `src/hooks/useWaveform.ts`
 
 Checklist:
 - Ensure native module methods are wrapped in a service:
   - `initialize`, `startListening`, `stopListening`, `cancelListening`, `getAudioLevel`, `normalizeAudio`
 - Ensure event subscriptions are properly set up and cleaned up.
 - Provide hook state:
   - `isListening`, `audioLevel`, `lastPartial`, `lastFinal`, `error`, `status`
 - Map start options:
   - `preserveAudio`
   - `preservedAudioFilePath`
   - `deleteAudioIfCancelled`
 
 Output:
 - Hook can start/stop/cancel and exposes state correctly.
 - Hook receives partial + final text updates.
 
 ---
 
 ## Phase 6 — UI Widget (Waveform + Controls)
 
 **Goal:** Build the visible dictation widget.
 
 Files:
 - `src/components/Waveform.tsx`
 - `src/components/AudioControlsDecorator.tsx`
 - `src/components/index.ts`
 
 Checklist:
 - `Waveform` shows recent audio levels with smooth bars.
 - `AudioControlsDecorator`:
   - Shows mic button when idle
   - Shows cancel + waveform + timer + check button when listening
   - Exposes `onMicPressed` and `onCancelPressed` callbacks
 - Export components from `src/components/index.ts`.
 
 Output:
 - UI matches the widget design and reacts to hook state changes.
 
 ---
 
 ## Phase 7 — Android Integration Validation
 
 **Goal:** Prove end-to-end dictation works on Android devices.
 
 Checklist:
 - Confirm module registered in `MainApplication`.
 - `initialize()` transitions to `ready` state.
 - `startListening()` transitions to `listening` state.
 - Partial and final results appear in UI.
 - Audio waveform responds to voice input.
 - Cancel flow ends session and, if configured, deletes audio file.
 - Stop flow ends session and emits `onAudioFile` if preserving audio.
 
 Output:
 - Dictation works on at least 1 emulator and 1 physical device.
 
 ---
 
 ## Phase 8 — Test Plan + Regression Coverage
 
 **Goal:** Ensure reliability across edge cases.
 
 Checklist:
 - Permissions:
   - First-time allow
   - Deny
   - Deny and "Don't ask again"
 - Error cases:
   - No network
   - Speech recognition unavailable
   - App backgrounded during permission request
 - Audio preservation:
   - `preserveAudio=false` (no files created)
   - `preserveAudio=true` (file created)
   - `deleteAudioIfCancelled=false` (file preserved on cancel)
 - Long session (10+ minutes) for stability.
 
 Output:
 - Any failures logged with steps to reproduce.
 - Issues filed before release.
 
 ---
 
 ## Phase 9 — Release Readiness
 
 **Goal:** Finish with clear ship criteria.
 
 Checklist:
 - All phases above complete.
 - No open critical bugs in dictation widget.
 - Android builds clean on CI.
 - Documentation is updated (link this doc from `docs/react_native_migration/README.md`).
 
 Output:
 - Android dictation widget is ready for production.
