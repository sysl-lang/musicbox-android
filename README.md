# musicbox-android

**A music box on an Android phone, written in sysl.** One button and one built-in tune: the tune is
text in [musicbox](https://github.com/sysl-lang/musicbox)'s LilyPond-like notation, the synthesizer
renders it, and [musicbox-miniaudio](https://github.com/sysl-lang/musicbox-miniaudio) plays it through
[miniaudio](https://github.com/sysl-lang/miniaudio) — which on Android opens an AAudio stream (or
OpenSL ES on a phone without one).

The project is [skitter-app](https://github.com/sysl-lang/skitter-app) with two lines of
`gradle.properties` and the program changed, so everything skitter-app's README says about building
holds here. The whole program is [`program/main.sysl`](program/main.sysl).

## The tune

Twinkle, Twinkle, Little Star — a bell over a plucked bass:

```
[tempo 100]
<< [instrument bell] [octave 5]
   c4 c g g a a g2 f4 f e e d d c2
 | [instrument pluck] [octave 3] [volume 0.7]
   c2 e f e d g c1 >>
```

Changing the music is changing that text in `program/main.sysl` and nothing else; musicbox's README
has the notation.

Tap the green play button and the tune plays from the top; the button turns red with a stop square.
Tap it again and it stops. When the last note has died away the device is closed and the button
goes back to play.

## How it is put together

**The synthesizer runs on the program's thread, not the audio thread.** miniaudio asks for samples on
a thread of its own; `Player.pump` renders ahead into a ring that thread reads, and once a frame is
plenty — so the SDL frame loop that draws the button is also the loop that keeps the music fed.
Dropping the last `&Player` closes the device, which is how both "stop" and "finished" end.

**No C, and no audio library to link.** miniaudio is vendored in its package and `dlopen`s
`libaaudio.so` at run time, so the APK carries no audio library and the CMake names none; `libmain.so`
needs only `libSDL3.so`, `liblog`, `libdl`, `libc` and `libm`.

**It logs to logcat.** On a phone standard output and standard error go nowhere, so the first line of
`SDL_main` is skitter's `logcat("musicbox")`, which turns every `sysl.log` record into a logcat entry
under the tag `musicbox`. The program logs `start` (with its `version`), `play`, `stop` and `finished`
(both with the number of `samples` the device played). Follow them with

```
adb logcat -s musicbox
```

That needs `log` in `target_link_libraries` in `app/src/main/cpp/CMakeLists.txt`: Gradle's CMake owns
the link and never sees the `@link("log")` the sysl module declares.

## Building it

You need Android Studio's SDK with the **NDK** and **CMake**, **`sbt`**, a **JDK between 17 and 25**,
and **sysl 0.0.161 or newer** (skitter 0.3.0's floor, which `logcat` needs).

```
skitter run
```

with [the `skitter` command](https://github.com/sysl-lang/skitter-cli), or by hand:

```
export ANDROID_HOME=~/Library/Android/sdk
./fetch-sdl3.sh
./gradlew assembleDebug
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

An emulator works, and it has to be an **arm64** one: sysl has one Android target, `aarch64-android`.

## Dependencies

`program/package.hocon` names three coordinates:

```hocon
dependencies {
  skitter            { git = "github.com/sysl-lang/skitter", version = "0.3.1" }
  musicbox           { git = "github.com/sysl-lang/musicbox", version = "0.1.3" }
  musicbox-miniaudio { git = "github.com/sysl-lang/musicbox-miniaudio", version = "0.1.0" }
}
```

Skitter brings SDL3 (0.3.3) and toml (0.1.1), and musicbox-miniaudio brings miniaudio (0.1.0). musicbox-miniaudio asks
for musicbox 0.1.0; naming 0.1.3 here lifts it, and `program/sysl.sum` records one musicbox, 0.1.3.

## What was verified

On the `Medium_Phone_API_36.1` emulator (arm64, Android 16, 1080×2400), with sysl 0.0.161. The button
is tapped with `adb shell input tap 540 1200`, the middle of the screen.

- **The program's own log.** `adb logcat -s musicbox` over a launch, a play to the end, then a play
  stopped two seconds in:

  ```
  I musicbox: start version=0.1.1
  I musicbox: play
  I musicbox: finished samples=469048
  I musicbox: play
  I musicbox: stop samples=104640
  ```

  469,048 samples at 48 kHz is 9.8 s, the tune with its release.
- **The tune plays to the end and closes the device.** A tap on the button: logcat shows
  `AAudioStreamBuilder_openStream() … returns 0 = AAUDIO_OK`, `AAudioStream_requestStart returned 0`
  and the stream reaching the started state, then `AAudioStream_close` about 10 s later, on the same
  millisecond as the `finished` line — the length of the tune, release included (the same score on a
  Mac plays 469,024 samples at 48 kHz, 9.8 s).
- **Frames reach the mixer.** `dumpsys media.audio_flinger` during playback lists the app's track as
  active on the speaker output — float, stereo, 48 kHz, **0 underruns** — and the output's signal-power
  history shows a note-by-note attack and decay.
- **Stop stops.** Play, then a second tap two seconds later: the stream is opened and
  `AAudioStream_close`d 2.0 s apart, and the `stop` line follows the close.
- No `AndroidRuntime` or native crash in the log.

Hearing it was not part of this; the log says the samples reached the speaker path.

## License

ISC — see [LICENSE](LICENSE).
