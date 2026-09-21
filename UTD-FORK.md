# UTD fork of flutter_webrtc 1.4.0

One change, in `android/.../MethodCallHandlerImpl.java`: a new engine option,
`softwareVoiceProcessing`.

## Why it exists

Upstream exposes exactly one switch that reaches the Android audio device
module — `bypassVoiceProcessing` — and it bundles five decisions:

    hardware AEC off · hardware NS off · stereo input · stereo output
    · MediaRecorder.AudioSource.MIC instead of VOICE_COMMUNICATION

We need the first two and none of the last three. The capture source is the
part that breaks: WebRTC's software canceller (AEC3) aligns the microphone
against what was played, inside a bounded delay window, and the
voice-communication path is the low-latency one Android tunes for exactly that.
Handed the raw MIC, it cannot track the delay.

Measured on 2026-09-21, one live room, three handsets on one build:

| device | Android | ERLE (dB removed) |
|---|---|---|
| Samsung SM-A175F | 16 | 8 – 12 |
| Xiaomi M2004J7AC (Redmi Note 9) | 12 | **0.18 – 1.3** (nothing) |
| BRP-NX1 | 16 | reports no stats at all |

The two failing devices also never stopped transmitting while their user was
silent — 15-58 kbps of background noise into the room, continuously — because
the noise suppressor was not running either. Both live in the same audio
processing module; when it does not run, both symptoms appear together.

Hardware cancellation is not a safe answer either: it is broken on a long tail
of devices that advertise it (Signal ships a device blacklist for this), and
under a media audio profile it does not engage at all.

## What the option does

    } else if (softwareVoiceProcessing) {
      setUseHardwareAcousticEchoCanceler(false)
      setUseHardwareNoiseSuppressor(false)
      setUseLowLatency(SDK_INT >= O)
      // capture source, channel count and sample rate left at their defaults:
      // VOICE_COMMUNICATION, mono, device rate
    }

So libwebrtc's APM is allowed to run, and it gets the call audio path it needs
to converge. This is the arrangement the commercial SDKs ship as their own 3A —
ZEGO's default `ModeGeneral` uses its own canceller, and their own docs note the
system one is weaker on some Android devices.

Unknown option keys are ignored by upstream, so a build without this fork simply
falls back to the platform canceller.
