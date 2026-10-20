# Audio playback

Kiloria's audio module decodes bounded WAV data and mixes voices into
interleaved stereo floating-point frames. It is independent of a windowing or
device library: an application opens its preferred platform audio device and
calls `AudioMixer::render` from that device's callback. The mixer serializes
control calls with rendering, so game code may play, stop, pause, and update
voices from another thread. That mutex can contend; this mixer is not a
hard-real-time-safe callback implementation, and latency-sensitive applications
should put controls through a nonblocking audio command queue.

```cpp
#include <kiloria.h>

auto decoded = kiloria::audio::WavDecoder::decode(wavBytes);
if (!decoded) {
    // Surface decoded.message in your asset diagnostics.
}
auto clip = std::make_shared<kiloria::audio::AudioClip>(std::move(decoded.clip));

kiloria::audio::AudioMixer mixer;
kiloria::audio::PlayOptions options;
options.spatial = true;
options.position = {4.0, 0.0, 2.0};
options.bus = "effects";
const auto voice = mixer.play(clip, options);

std::vector<float> stereoFrames(512 * 2);
mixer.render(stereoFrames);
```

The decoder supports PCM 8/16/24/32-bit and IEEE float32, mono and stereo,
including canonical PCM/float WAVE_FORMAT_EXTENSIBLE headers. Configurable
limits cap input bytes, decoded samples, and sample rate. Unsupported encodings,
malformed chunk lengths, inconsistent block alignment, and truncated RIFF data
return structured errors. File loading belongs in the asset layer; pass bounded
file bytes to the decoder.

The mixer provides linear sample-rate conversion, pitch, looping, voice limits,
pause/resume, master and named-bus gain, and distance attenuation with equal
power stereo panning. Spatial stereo clips are folded to mono before panning;
non-spatial stereo clips preserve their left/right channels. Output is clipped
to [-1, 1]. This is a straightforward CPU mixer, not a replacement for a
production audio middleware engine: it does not yet decode compressed formats,
stream long clips, convolve reverb, or provide an OS audio device backend. The
`kiloria_audio` example builds a short tone and mixes one spatial voice.

## Scene entities

Attach `AudioListenerComponent` to the camera/listener entity and
`AudioSourceComponent` to an emitter. `SceneAudioSystem::synchronize(scene)`
uses composed world transforms, including parent transforms, to update positions
and starts the source on first sync when `playOnAwake` is enabled. If more than
one listener is enabled, the first entity in scene order wins. Increment
`requestSerial` to restart a one-shot or looping voice. Replacing the clip
restarts a requested source; removing the source component or destroying its
entity stops its voice. Call `stopAll()` when the scene audio system is shut
down. `kiloria_scene_audio` demonstrates a looping emitter attached to a moving
parent object.
