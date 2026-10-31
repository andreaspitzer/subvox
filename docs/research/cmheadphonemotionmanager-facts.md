# Facts of CMHeadphoneMotionManager on macOS

Research for issue #3 (andreaspitzer/subvox). Date: 2026-10-31. Local SDKs: MacOSX26.5.sdk and MacOSX27.0.sdk (Command Line Tools, Xcode 27.2).

Source tags:

- **[SDK27]**: header or `.swiftinterface` in `MacOSX27.0.sdk`. Primary.
- **[SDK26.5]**: the same in `MacOSX26.5.sdk`. Primary.
- **[Apple doc]**: developer.apple.com documentation or WWDC session. Primary.
- **[Apple support]**: support.apple.com. Primary for consumer features.
- **[Repo]**: code in this repository.
- **[Secondary]**: third-party repository or forum post. Not verified.

## Answer

- **Sample rate: not documented and not settable.** The API has no interval property and no rate parameter [SDK27]. Apple gives no rate for headphone motion. The 100 Hz maximum in WWDC23 applies to `CMMotionManager`, not to headphones [Apple doc]. Third-party reports disagree: about 25 Hz (common, older reports), 50 Hz measured on AirPods Pro 2 with macOS 26, and about 100 Hz (not measured) [Secondary]. Measure it on our hardware.
- **Timestamp: host-boot clock.** `CMLogItem.timestamp` is "seconds since the device booted" [Apple doc]. The macOS 27 header changes the text to "Mach absolute time" [SDK27]. This is the same clock as `ProcessInfo.systemUptime` and `AVAudioTime` host time, so the alignment in `MotionSample` / `SessionRecorder` is correct in principle [Repo]. Apple does not say if the stamp is set on the AirPods (sensor time) or on the Mac (arrival time).
- **Jitter and latency: not documented.** No Apple figure exists. Measure both.
- **HFP (AirPods as microphone): risk of a full stop of motion.** No Apple statement. On iOS, several developers report that motion updates stop when the AirPods mic is active (`.playAndRecord` + `.allowBluetooth`) [Secondary, iOS only]. No report for macOS was found. This is the highest-risk open fact for "motion labelled by audio transcripts": record audio with a different mic until a measurement shows HFP is safe.
- **Supported hardware: AirPods with spatial audio and dynamic head tracking.** Apple names AirPods 3, AirPods 4, AirPods 5, AirPods Pro (all), AirPods Max [Apple support]. WWDC23 says "audio products that support spatial audio with dynamic head tracking" [Apple doc]. Beats Fit Pro is reported to work [Secondary]. AirPods 1 and 2 have no IMU [Secondary].
- **Supported OS: macOS 14.0 or later** (`API_AVAILABLE(macos(14.0))`) [SDK27]. Head tracking on a Mac needs Apple silicon [Apple support]. The AirPods must be the Mac's active audio output, and stay connected to the Mac (not auto-switched to an iPhone) [Secondary].
- **One bud at a time.** Data comes from one bud. `CMDeviceMotion.sensorLocation` tells which bud (left/right), and the other bud takes over when one leaves the ear [Apple doc]. `sensorLocation` is available on macOS [Apple doc]. Subvox does not record it today [Repo].
- **Data is processed, not raw.** The only payload is `CMDeviceMotion`: attitude, rotation rate, gravity, user acceleration (and magnetic field/heading fields that headphones do not fill in a known way) [SDK27]. No raw accelerometer, no raw gyro, no bone-conduction or speech-accelerometer data, no inward mic.
- **No newer API gives more articulation data on macOS.** macOS 26 and 27 add nothing to `CMHeadphoneMotionManager` except availability macros [SDK26.5 vs SDK27 diff]. New iOS-only items: AirPods high-quality Bluetooth recording (iOS 26, `API_UNAVAILABLE(macos)`), SensorKit `headphoneMotion` stream of `CMRecordedDeviceMotion` (iOS 27, `macOS unavailable`, research entitlement), `CMHeadphoneActivityManager` (macOS 15, activity class only) [SDK27].
- **Spatial Audio can move the yaw reference.** A macOS developer reports that yaw re-centres when Spatial Audio playback starts [Secondary]. Use relative attitude per segment (Subvox already does this in `MotionFeatures`) [Repo].

## Details

### 1. API surface on macOS

`CMHeadphoneMotionManager` is `API_AVAILABLE(macos(14.0), ios(14.0), watchos(7.0))`, unavailable on visionOS and tvOS [SDK27, `CMHeadphoneMotionManager.h`]. The public members are:

- `+authorizationStatus`, `delegate`, `connectionStatusActive`, `deviceMotionAvailable`, `deviceMotionActive`, `deviceMotion` (pull).
- `-startDeviceMotionUpdates`, `-startDeviceMotionUpdatesToQueue:withHandler:` (push), `-stopDeviceMotionUpdates`.
- `-startConnectionStatusUpdates`, `-stopConnectionStatusUpdates`.

There is no `deviceMotionUpdateInterval` (which `CMMotionManager` has) and no reference-frame parameter [SDK27]. Thus the app cannot set the rate. AWARE says "the update rate is controlled by the AirPods firmware and cannot be configured programmatically" [Secondary: github.com/awareframework/com.awareframework.ios.sensor.airpods].

Delegate callbacks run "on the operation queue used to deliver the device motion updates. The main thread is used if the queue was not specified" [SDK27, `CMHeadphoneMotionManagerDelegate.h`].

The header diff from SDK 26.5 to SDK 27.0 for the two headphone headers is only `API_UNAVAILABLE(tvos)` and blank lines [SDK26.5 vs SDK27].

WWDC23 "What's new in Core Motion" (session 10179) [Apple doc]:

- "starting this year, it's also coming to macOS 14."
- It streams device motion "from audio products that support spatial audio with dynamic head tracking, like AirPods Pro".
- With Automatic Ear Detection, a bud out of ear sends a disconnect event; in ear sends connect.
- "motion data is delivered to you from one bud at a time." `SensorLocation` identifies the bud; the other bud takes over when one is removed.
- The 100 Hz figure in the session is for `CMMotionManager` only. The session gives no rate for headphone motion.

### 2. Sample rate

| Value | Device / OS | How found | Source |
|---|---|---|---|
| not stated | — | — | [Apple doc], [SDK27] |
| ~25 Hz | AirPods on macOS (model not given) | not stated | [Secondary: github.com/crippler95/airtracker] |
| ~25 Hz | this repo's README "Known limits" | not stated | [Repo: README.md] |
| 50.0 Hz, "steady 20.0 ms between samples" | AirPods Pro 2, macOS 26 | measured (method not given) | [Secondary: github.com/changtimwu/airpod-pose-estimate] |
| "25 Hz on older hardware" | not given | not measured | [Secondary: same repo] |
| "approximately 100 Hz" | AirPods Pro / Max, iOS | not stated | [Secondary: AWARE repo] |

Conclusion: the rate probably depends on the AirPods model, firmware or OS. It is not a contract. The decoder spec must read the rate from the data and must not hard-code 25 Hz.

Check against the code: `HeadphoneMotionService` computes `sampleRate` from `sample.t` (the `CMDeviceMotion.timestamp`) over a 1 s window [Repo]. This measures the rate of the timestamps, not the rate of arrival. It cannot show delivery jitter or bursts.

### 3. Timestamp source

- `CMLogItem.timestamp`: "The timestamp is the amount of time in seconds since the device booted." [Apple doc: developer.apple.com/documentation/coremotion/cmlogitem/timestamp]
- macOS 27 SDK header: "Mach absolute time at which the item is valid." The 26.5 header says only "Time at which the item is valid." [SDK27, SDK26.5, `CMLogItem.h`]
- The type is `NSTimeInterval` (seconds), so "Mach absolute time" means the Mach clock converted to seconds. On macOS this clock does not count system sleep, the same as `ProcessInfo.systemUptime` and `AVAudioTime.seconds(forHostTime:)`.
- [Repo] `MotionSample.t = m.timestamp`; `SessionRecorder` stores `ProcessInfo.systemUptime` at start and `AVAudioTime.seconds(forHostTime:)` of the first audio buffer. The three use one clock, so the alignment design is correct.

Not documented: whether the AirPods stamp each sample on their own clock and CoreMotion maps it to host time, or whether the Mac stamps the sample on arrival. One third-party note says "Bluetooth delivery jitter lives in arrival time, not in `t`", which suggests a sensor-side stamp [Secondary: airpod-pose-estimate]. This decides whether `t` holds the true articulation time or the time plus a variable Bluetooth delay.

### 4. Latency and jitter

No Apple source gives latency or jitter for headphone motion. AirTracker calls itself "low-latency" but gives no number [Secondary]. Measure (see open facts).

Note on the code: `startDeviceMotionUpdates(to: .main)` delivers on the main queue [Repo]. UI work on the main thread adds delay to the handler. This does not change `t`, but it does change arrival time and any live trigger (`CaptureController`) that runs on arrival.

### 5. HFP profile (AirPods as microphone)

- Apple says nothing about motion while the AirPods mic is active. The support guide does not mention calls or the mic for head tracking [Apple support].
- iOS developers report: with `setCategory(.playAndRecord, mode: .default, options: .allowBluetooth)` motion updates stop; without `.allowBluetooth` (iPhone mic) motion works. Reconnecting the AirPods sometimes restores motion. Reports from May 2021 to Feb 2024, no Apple reply [Secondary: developer.apple.com/forums/thread/681301].
- No report for macOS was found. macOS has no `AVAudioSession`; the switch to HFP happens when an app selects the AirPods as the input device. The README already notes this switch [Repo: README.md "Known limits"].
- A macOS developer reports that an audio tap stops when the AirPods change from A2DP to HFP during a recording (input at 24 kHz, output still at 48 kHz) [Secondary: github.com/scosman/Biscotti/issues/88]. This is about audio, not motion, but shows that the profile change is disruptive.
- AirPods high-quality Bluetooth recording (full-band input instead of HFP) exists only on iOS 26+: `AVAudioSessionCategoryOptionBluetoothHighQualityRecording` and `AVCaptureSession.configuresApplicationAudioSessionForBluetoothHighQualityRecording` are `API_UNAVAILABLE(macos)` [SDK27, `AVAudioSessionTypes.h`, `AVCaptureSession.h`]. The header warns "This option may increase input latency" [SDK27]. So on macOS the AirPods mic means HFP.

Consequence for the spec: for passive sessions labelled by audio, use the Mac's built-in mic (or another mic) until a measurement shows that motion rate and quality do not change in HFP.

### 6. Supported AirPods models and OS versions

- OS: macOS 14.0+ for the API [SDK27]. Head tracking on a Mac: "Mac computers with Apple silicon and macOS 12.3 or later" [Apple support]. Combined: Apple-silicon Mac with macOS 14+. Not documented: whether the API works on Intel Macs; Apple ties the head-tracking feature to Apple silicon.
- Models: "AirPods 3, AirPods 4, AirPods 5, AirPods Pro, and AirPods Max use Spatial Audio and head tracking" [Apple support: support.apple.com/guide/airpods/dev00eb7e0a3/web]. Apple's sample code names "AirPods Pro 2 or AirPods 4" as examples [Apple doc: getting-motion-activity-data-from-headphones]. AirPods Max over USB-C cable also does head tracking with macOS 15.4+ [Apple support]; not documented if `CMHeadphoneMotionManager` delivers data over the cable.
- Beats: AirTracker lists Beats Fit Pro [Secondary]. Not confirmed by Apple for this API.
- Firmware: an early iOS 14 beta did not deliver data until an AirPods firmware update (3A283) [Secondary: developer.apple.com/forums/thread/650051]. Firmware can change behaviour, so record the firmware version with each session.
- Runtime check: `isDeviceMotionAvailable` [Apple doc]. Subvox uses it [Repo].

### 7. Newer APIs (macOS 26 / 27, iOS 26 / 27)

| API | Platforms | Gives more articulation data? | Source |
|---|---|---|---|
| `CMHeadphoneMotionManager` changes 26.5 → 27 | macOS | No. Only availability macros. | [SDK diff] |
| `CMLogItem.timestamp` doc text "Mach absolute time" | all | No. Clarifies clock. | [SDK27] |
| `CMHeadphoneActivityManager` | macOS 15, iOS 18, watchOS 11 | No. Activity class (walking, running, …) and connect status only. | [SDK27] |
| `CMRecordedDeviceMotion` (subclass of `CMDeviceMotion`, adds `identifier`, `startDate`) | macOS 27, iOS 27 | No new fields. "passively collected motion data, including data from accessories such as headphones." | [SDK27, `CMRecordedDeviceMotion.h`] |
| SensorKit `SRSensorHeadphoneMotion` / `SRHeadphoneMotionSensor` (returns `[CMRecordedDeviceMotion]`) | iOS 27 only, `macOS unavailable` | No. Historical, batched; needs the SensorKit research entitlement. | [SDK27, `SRSensors.h`, `SensorKit.swiftinterface`] |
| SensorKit `SRSensorHeadphoneSettings` | iOS 27 only | No. Settings only. | [SDK27] |
| `CMBodyIdentifiable`, `CMMotionManager.deviceMotionBody` | `deviceMotionBody` is `API_UNAVAILABLE(macos)` | No. Reference body for device motion. | [SDK27, `CMBody.h`, `CMMotionManager.h`] |
| `CMDeviceMotion.headingAccuracy` | iOS 27 | No. | [SDK27] |
| AirPods high-quality recording (`bluetoothHighQualityRecording`) | iOS 26 only | Audio, not motion. Unavailable on macOS. | [SDK27], [Apple doc: WWDC25 session 251] |
| `AVAudioApplication` input-mute handler (AirPods stem gesture) | macOS 14 | No. | [SDK27, `AVAudioApplication.h`] |
| `kAudioUnitProperty_SpatialMixerEnableHeadTracking` | macOS | No. Uses AirPods motion inside the spatial mixer; no data out. | [SDK27, `AudioUnitProperties.h`] |
| `CMHighFrequencyHeartRateData` | iOS 17, watchOS 10, `API_UNAVAILABLE(macos)` | No. | [SDK27] |

No public API on any platform exposes the AirPods speech-detect accelerometer, bone-conduction signal, inward mic, raw IMU, or both buds at once. A full-text search of `MacOSX27.0.sdk` headers for "AirPods" finds only the items above [SDK27].

### 8. Findings in Subvox/Motion

- `HeadphoneMotionService.sampleRate` uses timestamps, not arrival times. Add an arrival-time stamp (`ProcessInfo.systemUptime` in the handler) to measure jitter and latency [Repo].
- `MotionSample` does not store `sensorLocation`. A bud switch changes the frame and the signal; without this field the data cannot show it [Repo, Apple doc].
- Updates go to `.main`. A dedicated `OperationQueue` removes UI-caused delay for live triggering [Repo, SDK27].
- `connectionChanged(false)` resets the rate but keeps `history`. Because one bud can take over from the other, a disconnect/connect pair can mark a bud switch; log these events into `events.csv` [Repo, Apple doc].

## Open facts that need an on-device measurement

Record AirPods model, firmware version, macOS version and Mac model with every run.

1. **Real sample rate per model.** Stream 5 min while still and 5 min while mouthing. Report the median and the 1st/99th percentile of `diff(t)`. Repeat for each AirPods model we own. Check if the rate changes with motion, with Spatial Audio on/off, and with audio playing/not playing.
2. **Timestamp source and jitter.** In the handler, also store `arrival = ProcessInfo.processInfo.systemUptime`. Compare `diff(t)` with `diff(arrival)`. If `diff(t)` is near-constant and `diff(arrival)` is bursty, the stamp is sensor-side. Report `arrival − t` (median, spread).
3. **End-to-end latency.** Tap the AirPods stem area (or knock the jaw) 30 times while the Mac's built-in mic records. Find the audio onset (host time) and the motion onset in `t` and in `arrival`. Latency = motion onset − audio onset. This also gives the audio-to-motion alignment error that matters for transcript labels.
4. **HFP effect on macOS.** Run 3 min with built-in mic, then select the AirPods as input (HFP) for 3 min, then switch back. Report: does motion stop, does the rate change, does noise (std of rotation rate while still) change, does `sensorLocation` change, does motion recover after switching back.
5. **Bud switching.** Log `sensorLocation` and connect/disconnect events. Take one bud out and in. Check the step in attitude and gravity at the switch.
6. **Spatial Audio yaw re-centre.** Start and stop music playback with Spatial Audio "Head Tracked", "Fixed" and "Off". Check for yaw steps.
7. **Multiple clients.** Run Subvox and a second app with `CMHeadphoneMotionManager` together. Check rate and reference frame in both.
8. **Magnetic field / heading fields.** Log `magneticField.accuracy` and `heading` once; confirm they are empty or invalid for headphones, so the spec can ignore them.
9. **Intel Mac and USB-C AirPods Max.** Only if such hardware is in scope: check `isDeviceMotionAvailable` and data flow.
