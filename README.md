# Playout Statistics API for WebAudio explainer

This is the repository for Playout Statistics API for WebAudio. You're welcome to
[contribute](CONTRIBUTING.md)!

## Authors:

- Fredrik Hernqvist
- Palak Agarwal

## Participate
- https://github.com/WICG/proposals/issues/142

## Introduction

test commit please ignore

There is currently no way to detect whether WebAudio playout has glitches (gaps in the played audio, which typically happens due to underperformance in the audio pipeline). There is an existing way to measure the instantaneous playout latency using [AudioContext.outputLatency](https://developer.mozilla.org/en-US/docs/Web/API/AudioContext/outputLatency), but no simple way to measure average/minimum/maximum latency over time. With this API, we want to propose a way to be able to measure the delay of that audio and the glitchiness of the audio.


## Motivating Use Cases

Glitches and high latency are bad for the user experience, so if any of these occur it can be useful for the application to be able to detect this and possibly take some action to improve the playout.

With the API, we want to be able to calculate the following things:

- The fraction of audio that was made up from fallback frames (which in chromium is silence) due to audio glitches over a certain time interval
- The number of audio glitches over a certain time interval
- The average audio playout delay over a certain time interval, and the interval that the delay moves in

## Proposed Solution

We can do this by adding a new attribute playoutStats, of type AudioPlayoutStats, to AudioContext interface. This new interface exposes multiple values that measure the glitches and latency in the audio playout.


```js
partial interface AudioContext {
[SameObject] readonly attribute AudioPlayoutStats playoutStats;
}

/**
Fallback frames: Frames played by the output device that were not provided
by the playout path. This happens when the playout path fails to provide audio frames
to the output device on time, in which case fallback frames will be played. This typically
only happens if the pipeline is underperforming. In the Chromium implementation, fallback
frames are always silence. This includes underrun situations that happen for reasons unrelated to WebAudio/AudioWorklets.
*/

[Exposed=Window, SecureContext]
interface AudioPlayoutStats {
readonly attribute DOMHighResTimeStamp fallbackFramesDuration;
readonly attribute unsigned long fallbackFramesEvents;
readonly attribute DOMHighResTimeStamp totalFramesDuration;
readonly attribute DOMHighResTimeStamp averageLatency;
readonly attribute DOMHighResTimeStamp minimumLatency;
readonly attribute DOMHighResTimeStamp maximumLatency;
undefined resetLatency();
[Default] object toJSON();
}

```


## Key scenarios

This is an example of how the API can be used to calculate the following stats over a time interval:
- Fraction of fallback frames
- Frequency of fallback frames events (glitches)
- Average playout delay

```js
var oldTotalFramesDuration = audioContext.playoutStats.totalFramesDuration;
var oldFallbackFramesDuration = audioContext.playoutStats.fallbackFramesDuration;
var oldFallbackFramesEvents = audioContext.playoutStats.fallbackFramesEvents;
audioContext.playoutStats.resetLatency();

// Wait while playing audio
...

// the number of seconds that were covered by the frames played by the output device between the two executions.
let deltaTotalFramesDuration = (audioContext.playoutStats.totalFramesDuration - oldTotalFramesDuration) / 1000;
let deltaFallbackFramesDuration = (audioContext.playoutStats.fallbackFramesDuration - oldFallbackFramesDuration) / 1000;
let deltaFallbackFramesEvents = audioContext.playoutStats.fallbackFramesEvents - oldFallbackFramesEvents;

// fallback frames fraction stat over the last deltaTotalFramesDuration seconds
let fallbackFramesFraction = deltaFallbackFramesDuration / deltaTotalFramesDuration;
// fallback event frequency stat over the last deltaTotalFramesDuration seconds
let fallbackEventFrequency = deltaFallbackFramesEvents / deltaTotalFramesDuration;
// Average playout delay stat during the last deltaTotalFramesDuration seconds
let playoutDelay = audioContext.playoutStats.averageLatency / 1000;
```

## Detailed design discussion

The AudioPlayoutStats under AudioContext will be a dedicated object for statistics reporting; Similar to [RTCAudioPlayoutStats](https://w3c.github.io/webrtc-stats/#dom-rtcaudioplayoutstats) but it is for the playout path via AudioDestinationNode and the associated output device. This will allow us to measure glitches occurring due to underperforming AudioWorklets as well as glitches and delay occurring in the playout path between the AudioContext and the output device.

The stats are kept in sync with each other in accordance with run-to-completion semantics. If one of the stats has been observed, we consider all of the other stats to have been observed simultaneously (i.e. they won't change). 

- fallbackFrameDuration: Measured in milliseconds and is incremented each time a fallback frame is played by the output device at the end of the playout path. This metric can be used together with totalFramesDuration to calculate the percentage of played out media that was not provided by the AudioContext.
- fallbackFramesEvents: Number of synthesized fallback frames events. This counter increases every time a fallback frame is played after a non-fallback frame. That is, multiple consecutive fallback frames will increase fallbackFramesDuration multiple times but is a single fallback frames event.
- totalFramesDuration: Total duration, in milliseconds, of all audio frames that have been played by the audio device. Includes both fallback and non-fallback frames.
- The following latency-related properties and methods are similar to the MediaStreamTrackAudioStats interface.
    - averageLatency: The average latency for the frames played since the last call to resetLatency(), or since the creation of the AudioContext if resetLatency() has not been called.
    - minimumLatency: The minimum latency for the frames played since the last call to resetLatency(), or since the creation of the AudioContext if resetLatency() has not been called.
    - maximumLatency: The maximum latency for the frames played since the last call to resetLatency(), or since the creation of the AudioContext if resetLatency() has not been called.
    - resetLatency(): Resets the latency counters. Note that it does not remove latency information that has accrued but not yet been exposed through the API.

## Security and Privacy Considerations

A more detailed discussion about security and privacy can be found in the [Security and Privacy self review](https://docs.google.com/document/d/1wGv_mr7Lgg2w-6PuKDrcScoa8IvYAOW3PMTFW85O3Gw/edit#heading=h.d6giaiodx3q3).

### Cross-site covert channeling

In Chromium, Safari and Firefox it is already possible to intentionally communicate between different origins using audio glitches ([see explanation here](https://github.com/w3ctag/design-reviews/issues/939#issuecomment-2022954199)). This side channel has a very high latency, is audible to the user, and degrades system performance, so it is unlikely to be useful to attackers. 

It would be good if the introduction of the Playout Statistics API for WebAudio does not reduce the latency of this side channel. We could do this by limiting the rate at which the information provided by the API is updated, similar to the [Rate limitation used for the ComputePressure API](https://www.w3.org/TR/compute-pressure/#rate-limiting-change-notifications). This would increase the latency of any attempts to communicate using the glitch information, ensuring that the audio glitch-based side channel is not made more useful.

## Comparison to [WebAudio RenderCapacity API](https://github.com/w3ctag/design-reviews/issues/843)

### Usage
The proposed WebAudio RenderCapacity API gives information about how much load the audio rendering thread is under, and the ratio of glitches that are occurring if that load exceeds 100%. This is intended to allow applications to react when the rendering thread is approaching maximum load, in which case the application can proactively reduce the complexity of the audio computations to avoid audio glitches occurring.

The Playout Statistics API for WebAudio that we are proposing in this explainer gives audio glitch and delay information about the user-experienced audio playout, but no information about the usage of the rendering thread. This is intended to measure user experience across the population, and can also be used to react to when glitches occur, in which case the application can reduce complexity of audio computations or prompt the user to take some action, such as closing unused applications.

### Security risks

The RenderCapacity API gives information about the usage of the rendering thread even when there are no audio glitches, while the Playout Statistics API never gives any direct information about the rendering thread. Both APIs report audio glitches, the presence of which could mean that the rendering thread is overloaded but could also have a number of other explanations, as described in [the Side-channeling section of the Security and Privacy self review](https://docs.google.com/document/d/1wGv_mr7Lgg2w-6PuKDrcScoa8IvYAOW3PMTFW85O3Gw/edit#heading=h.14monskf6i0b). This means that the RenderCapacity API gives more information about CPU usage than the Playout Statistics API. Therefore we believe that the Playout Statistics API is safer than the RenderCapacity API.

## Considered alternatives

### Alternative delay definition

Alternatively, we can just define a totalPlayoutDelay similar to https://w3c.github.io/webrtc-stats/#dom-rtcaudioplayoutstats-totalplayoutdelay.

In this case we also need to add a total frames count attribute: When audio frames are pulled by the playout device, this counter is incremented with the number of frames emitted for playout. Note that we won't be able to calculate minimum/maximum latency with this approach.

```js
readonly attribute double totalPlayoutDelay;
readonly attribute unsigned long long totalFramesCount; 
```
