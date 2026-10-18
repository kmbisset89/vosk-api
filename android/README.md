# Vosk library for Android

Offline speech recognition for Android. The Java library in `lib/` is a JNA binding to `libvosk.so`. Recognition runs on device. Audio never leaves the phone.

Build the native library, then the AAR:

```sh
export ANDROID_NDK_HOME="$HOME/Library/Android/sdk/ndk/28.2.13676358"
export VOSK_ANDROID_ABIS="arm64-v8a"
cd lib
bash ./build-vosk.sh
cd ..
./gradlew :lib:assembleRelease :model-en:assembleRelease
```

`VOSK_ANDROID_ABIS` is a space-separated list. The default is `armeabi-v7a arm64-v8a x86_64 x86`. NDK `28.2.13676358` is the version pinned in `lib/build.gradle`. `build-vosk.sh` clones and compiles OpenBLAS, CLAPACK, OpenFST, and Kaldi, then copies `libvosk.so` into `lib/src/main/jniLibs/<abi>/`. The host needs a working `make` and `gcc`. On macOS that means the Xcode license has been accepted.

`:lib:assembleRelease` produces `lib/build/outputs/aar/vosk-android-release.aar` (`io.github.kmbisset:vosk-android`). It depends on JNA `net.java.dev.jna:jna:5.18.1` with type `aar`, which supplies `libjnidispatch.so`. Use Gradle 8.13 (`./gradlew` in this directory).

`:model-en:assembleRelease` publishes as `io.github.kmbisset:vosk-model-en`.

## Maven Central

`.github/workflows/android-maven-central.yml` builds `libvosk.so` and publishes both AARs. Pull requests build `arm64-v8a` and assemble only. A `v*` tag or a manual run builds `armeabi-v7a`, `arm64-v8a`, `x86_64`, and `x86`, then publishes and releases the version. The manual run takes the version without a leading `v`.

Add these repository secrets before the first release:

- `MAVEN_CENTRAL_USERNAME` and `MAVEN_CENTRAL_PASSWORD`: a Central Portal user token, not the portal login.
- `SIGNING_IN_MEMORY_KEY`: the ASCII-armored private key from `gpg --export-secret-keys --armor <key id>`.
- `SIGNING_PASSWORD`: the passphrase for that key. Leave it empty when the key has none.

The public key must be on a keyserver Central can see. The namespace `io.github.kmbisset` has to be verified on the Central Portal for the account that owns the token.

`:model-en:assembleRelease` downloads `vosk-model-small-en-us-0.15`, checks that it contains `HCLr.fst` and `Gr.fst`, and packs the files as assets under `model-en-us/` with a `uuid` file. That archive is `model-en/build/outputs/aar/vosk-model-en-release.aar`. Grammar commands require those lookahead graphs. A model that only ships `HCLG.fst` ignores the phrase list.

## Command recognition

Depend on both AARs. Request `RECORD_AUDIO` before creating `SpeechService`.

`StorageService` copies `model-en-us` out of assets because Kaldi opens a filesystem path. The `uuid` file is `vosk-model-small-en-us-0.15`, so a later launch skips the copy while that model is already unpacked.

```java
StorageService.unpack(context, "model-en-us", "models",
    model -> startCommands(model),
    error -> { /* model copy or load failed */ });
```

Create the recognizer with a JSON array of phrases. The sample rate must be `16000` to match this model. `SpeechService` records 16 kHz mono PCM 16-bit from `VOICE_RECOGNITION` and feeds 200 ms chunks to that recognizer.

```java
String grammar = "[\"call mom\", \"set a timer\", \"[unk]\"]";
Recognizer recognizer = new Recognizer(model, 16000.0f, grammar);
recognizer.setEndpointerMode(Recognizer.EndpointerMode.SHORT);

SpeechService speechService = new SpeechService(recognizer, 16000.0f);
speechService.startListening(new RecognitionListener() {
    @Override
    public void onPartialResult(String hypothesis) {
        // {"partial" : "call"}
    }

    @Override
    public void onResult(String hypothesis) {
        // {"text" : "call mom"}  — silence closed the utterance
    }

    @Override
    public void onFinalResult(String hypothesis) {
        // same shape as onResult, after stop()
    }

    @Override
    public void onError(Exception exception) { }

    @Override
    public void onTimeout() { }
});
```

`onResult` is the command event. Read `text` from the JSON. Treat `""` and `"[unk]"` as no command: `[unk]` means the utterance was not in the phrase list. Anything else is the matched phrase.

Each phrase is split on spaces and each word must appear in the model vocabulary. This model stores that vocabulary in the output symbols of `graph/Gr.fst`; it does not ship a separate `words.txt`. Words that are missing are dropped, so a phrase can silently recognize as nothing. Tokens are lowercase.

`setGrammar("[]")` restores the model's default large graph. Call it only while the recognizer is idle. Calling it while audio is being decoded throws. The recognizer is not thread-safe; only the `SpeechService` thread should call it.

`startListening(listener, timeoutMs)` stops on its own and then calls `onTimeout()` instead of `onFinalResult()`. `SHORT` scales the model's trailing-silence rules by 0.75 so a short command endpoints sooner. `setEndpointerDelays` takes those silences in seconds.

When the screen is finished:

```java
speechService.stop();
speechService.shutdown();
recognizer.close();
model.close();
```

`stop()` delivers `onFinalResult` for audio still in the pipeline. `shutdown()` releases the microphone.
