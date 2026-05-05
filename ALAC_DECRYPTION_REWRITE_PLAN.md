# Streaming-First ALAC Stability Plan

## Hard Rules

- Do not download the full song before playback.
- Do not do full-track remuxing before playback.
- Do not touch frontend/player UI unless a later task explicitly asks for it.
- Keep package name, signing, version, and release metadata unchanged.
- Keep provider fallback behavior: Apple Music ALAC first, then Qobuz, then YouTube, unless user settings say otherwise.

## Goal

Make Apple Music ALAC reliable enough for normal playback:

- Fast initial start.
- No 13-second or mid-song freeze.
- No malformed MP4 atom errors.
- No FFmpeg ALAC decode failures from damaged samples.
- Seeking works without scanning or downloading the whole track.
- Old decrypt work cancels immediately when skipping tracks.

## What The Deep Dive Found

AppleMusicDecrypt, wrapper-manager, wrapper, and zhaarey/apple-music-downloader point to three main lessons:

1. AppleMusicDecrypt is reliable because it uses a long-lived wrapper-manager decrypt stream and tracks every sample result with futures/retries.
2. zhaarey/apple-music-downloader has an `alacfix` pass that patches malformed ALAC packet tails without changing packet sizes.
3. zhaarey's streaming decrypt path only decrypts protected AES block-aligned bytes, not necessarily every byte in a sample.
4. wrapper-manager keeps one TCP decrypt connection per wrapper instance and switches context by adamId/key.
5. Our app currently pretends decrypted ALAC is one progressive fMP4 stream, which makes seeking and parser recovery harder than real segment-based HLS playback.

## Current Suspected Problems

- Whole-sample decrypting can damage ALAC packet tails when only the block-aligned protected prefix should be decrypted.
- Decrypt responses can be same-size but still unchanged or invalid.
- Segment validation catches some malformed MP4 now, but it does not prove ALAC packets inside are decodable.
- The fake progressive stream makes ExoPlayer seek by byte offset, forcing us to estimate segment/sample indexes manually.
- If a later segment decrypts slowly, Exo can run dry and playback freezes even though the player itself is fine.

## Phase 1 - Instrument Before Changing Behavior

Add detailed ALAC logs with one stable `sessionId` per playback attempt:

- wrapper host selected
- M3U8 fetch time
- init fetch time
- selected ALAC playlist quality
- segment download time
- segment sample parse time
- per-key decrypt time
- retry count
- unchanged sample count
- invalid packet count
- segment validation result
- bytes ready ahead of playback
- Exo buffering/rebuffer events
- fallback reason

Acceptance:

- A single log sequence can explain why a song froze, fell back, or decoded badly.
- Logs include enough data to compare `wm.wol.moe` vs `wm1.wol.moe`.

## Phase 2 - Fix Sample Decrypt Correctness

Implement CBCS-aware sample preparation before sending samples to wrapper-manager.

For each encrypted sample:

1. Parse encryption metadata from init/moof/traf/senc/saiz/saio/sgpd/sbgp where possible.
2. Determine protected ranges.
3. For ALAC full-sample encryption, decrypt only the largest multiple-of-16 protected prefix.
4. Preserve non-protected or trailing bytes exactly.
5. Reassemble the sample in place.

Add validation:

- decrypted size must equal original size
- decrypted output must not equal encrypted input for samples large enough to compare
- any unchanged suspicious sample is retried
- repeated unchanged failures mark the segment invalid

Acceptance:

- No full-song download.
- Segment output remains the same size/layout as input.
- Wrapper returns that are technically same-size but useless no longer reach Exo.

## Phase 3 - Add Streaming-Safe ALAC Packet Repair

Port the useful part of zhaarey's `alacfix` to Kotlin as a small packet-level repair helper.

Do it per decrypted sample, not on a full file:

1. Read ALAC decoder config from the init segment.
2. Parse each decrypted ALAC packet enough to find the true element body end.
3. If the packet tail does not begin with ALAC TYPE_END, patch those 3 bits to `111`.
4. Zero-pad the remaining tail bits.
5. Never change sample size.

Acceptance:

- FFmpeg ALAC decode failures caused by malformed packet tails are reduced or gone.
- Repair is only applied to ALAC samples.
- Invalid/unparseable packets are logged and retried/fallbacked, not silently ignored.

## Phase 4 - Improve Segment Integrity And Retry Routing

Before returning a segment to Exo:

- validate top-level MP4 box layout
- require `moof` + `mdat`
- verify child box layout
- verify encryption boxes are removed or neutralized
- verify sample count matches
- verify repaired ALAC packets are sane enough to decode

Retry strategy:

1. Retry same wrapper once for transient misses.
2. Retry a different wrapper-manager host for integrity failures.
3. Mark host/session as degraded if it returns repeated invalid segments.
4. Fallback provider only after bounded retries fail.

Acceptance:

- `Skipping atom with length > 2147483647` should not reach ExoPlayer.
- A bad wrapper response retries elsewhere before falling back to Qobuz.

## Phase 5 - Replace Fake Progressive ALAC With Virtual HLS

This is the biggest structural improvement.

Instead of exposing one fake continuous fMP4 stream, expose a virtual local ALAC HLS source:

- `/alac/{session}/master.m3u8`
- `/alac/{session}/media.m3u8`
- `/alac/{session}/init.mp4`
- `/alac/{session}/seg-{index}.m4s`

The media playlist should use Apple segment durations and stable segment indexes.

When Exo requests a segment:

1. Return cached decrypted segment if ready.
2. Otherwise fetch/decrypt/repair/validate that segment.
3. Prefetch the next 2-4 segments in the background.
4. Keep memory bounded around 24-48 MB.
5. Cancel old sessions on skip.

Why this helps:

- ExoPlayer can seek by HLS segment/time instead of fake byte offsets.
- The app no longer has to precompute all previous segment lengths/sample counts for seeks.
- Parser errors are isolated to one segment.
- Rebuffering becomes normal HLS behavior instead of a broken progressive read.

Acceptance:

- Seeking works by segment index and does not scan/download the whole song.
- Mid-song segment failure can retry that segment.
- Playback no longer depends on a guessed continuous byte stream.

## Phase 6 - Persistent Decrypt Client

Implement a real long-lived decrypt client like AppleMusicDecrypt's wrapper-manager flow.

Create:

- `AppleMusicDecryptClient`
- `StreamingGrpcDecryptClient`
- `BatchGrpcDecryptClient` fallback

Streaming client behavior:

- open one gRPC `Decrypt` stream per ALAC session
- keep it alive
- queue samples from segments N, N+1, N+2
- map replies by `sampleIndex`
- complete per-sample futures
- retry failed samples
- close immediately on track skip

Fallback:

- if streaming gRPC fails to initialize or drops repeatedly, use current batch decrypt path.

Acceptance:

- While Exo consumes segment N, samples for N+1 are already decrypting.
- One slow sample does not block unrelated queued samples forever.
- Closing/skipping cancels futures and the network call.

## Phase 7 - ExoPlayer Buffer Tuning

Only tune Exo after segment correctness is fixed.

Use Media3 `DefaultLoadControl.Builder` for ALAC playback:

- increase streaming `minBufferMs`
- increase streaming `maxBufferMs`
- increase `bufferForPlaybackAfterRebufferMs`
- consider `setPrioritizeTimeOverSizeThresholdsForStreaming(true)`
- keep normal providers on current/default settings if possible

This is meant to smooth wrapper latency. It will not fix corrupted samples by itself.

Acceptance:

- Normal ALAC startup remains reasonable.
- Rebuffers are less frequent during slow decrypt periods.
- Memory remains bounded.

## Phase 8 - Audio Offload Eligibility

Audio offload is worth fixing, but it is not the primary ALAC stability fix.

Expected benefit:

- lower CPU usage
- lower battery drain
- less thermal throttling during long ALAC playback
- slightly more headroom for decrypt/prefetch work

Important caveat:

- Audio offload will not fix malformed MP4 segments or damaged ALAC packets.
- Offload usually cannot run when the app is using custom audio processing.
- Current playback always builds an audio processor chain with EQ, silence detection, silence skipping, and Sonic speed/pitch support, so real offload may be blocked even when the setting is enabled.

Plan:

1. Detect when playback is offload-eligible:
   - crossfade off
   - EQ/effects off
   - tempo/pitch unchanged
   - skip silence off
   - instant silence detector not needed
   - codec/device supports offload
2. Build a lighter audio sink/renderer path when eligible:
   - no custom audio processors
   - enable Media3 offload preferences
   - keep the existing processor chain for normal/effects playback
3. Recreate or safely update the player when eligibility changes.
4. Log whether offload was enabled, disabled by settings, or blocked by effects/device support.

Acceptance:

- Offload actually becomes active on supported devices when effects are disabled.
- Turning on EQ/crossfade/speed/pitch/silence features cleanly disables offload.
- ALAC playback behavior is not made harder to debug by hidden processor changes.

## Test Matrix

Use at least these cases:

- fast ALAC track
- track that freezes around 13 seconds
- track that freezes around 1-2 minutes
- track that previously threw `PARSING_CONTAINER_UNSUPPORTED`
- track that previously threw `DECODING_FAILED`
- seek to 30%, 60%, and 90%
- skip rapidly across several ALAC tracks
- wrapper host failover test
- fallback to Qobuz test

Build check:

```powershell
.\gradlew.bat :app:assembleFossDebug
```

Release test build should keep the existing MetroApple package/signature:

- package: `com.metroapple.music`
- ABI: `arm64-v8a`
- non-debug release build
- native libs must remain installable/uncompressed as needed

## Suggested Implementation Order

1. Add better timing and failure logs.
2. Implement protected-range / multiple-of-16 decrypt handling.
3. Add unchanged-output detection and retry.
4. Port per-sample ALAC packet repair.
5. Retry failed segments on alternate wrapper-manager host.
6. Build virtual HLS DataSource/MediaSource path.
7. Add persistent streaming gRPC decrypt client.
8. Tune ExoPlayer buffer/load control for ALAC only.
9. Fix audio offload eligibility and logging.

## Files Likely To Touch

- `app/src/main/kotlin/com/metrolist/music/apple/AppleMusicDecryptPipeline.kt`
- `app/src/main/kotlin/com/metrolist/music/apple/AppleMusicWrapperManagerProvider.kt`
- `app/src/main/kotlin/com/metrolist/music/apple/AppleMusicWrapperDataSource.kt`
- `app/src/main/kotlin/com/metrolist/music/apple/AppleMusicSongResolver.kt`
- `app/src/main/kotlin/com/metrolist/music/playback/MusicService.kt`
- `app/src/main/kotlin/com/metrolist/music/eq/audio/CustomEqualizerAudioProcessor.kt`
- `app/src/main/kotlin/com/metrolist/music/playback/audio/SilenceDetectorAudioProcessor.kt`

Potential new files:

- `AppleMusicAlacPacketRepair.kt`
- `AppleMusicVirtualHlsDataSource.kt`
- `AppleMusicVirtualHlsSession.kt`
- `AppleMusicStreamingDecryptClient.kt`

## Stop Conditions

Do not keep stacking timeout tweaks if malformed samples or malformed segments still occur.

If segment integrity fails repeatedly:

- log the host, segment index, sample index range, and failure type
- retry another wrapper host
- fallback for the current playback attempt
- do not permanently disable Apple Music ALAC

## References

- `WorldObservationLog/AppleMusicDecrypt`
- `WorldObservationLog/wrapper-manager`
- `WorldObservationLog/wrapper`
- `zhaarey/apple-music-downloader`
- AndroidX Media3 `DefaultLoadControl.Builder`
- AndroidX Media3 audio offload preferences
- AndroidX Media3 `FragmentedMp4Extractor`
