# react-native-video fork — changes to port upstream

These changes currently live only as a local `patch-package` patch
(`patches/react-native-video+6.19.2.patch`) applied on top of
`git+https://github.com/logicwind/react-native-video.git#ott_sideload_text_tracks_6.19.2`
(pinned in `package.json`). They were made while adding support for merging
backend-sideloaded external subtitles with subtitle tracks already
incorporated in the video's own manifest (embedded HLS text tracks).

**Goal of this doc:** let you port each change into the actual fork repo
(same branch, or a new one) so the local patch can eventually be retired.

Once ported upstream and `package.json`'s git ref is bumped to the new
commit, delete `patches/react-native-video+6.19.2.patch` and this file (or
keep this file as changelog — your call).

---

## Summary

| # | File | Platform | Bug type | One-liner |
|---|------|----------|----------|-----------|
| 1 | `android/.../exoplayer/ReactExoplayerView.java` | Android | Silent data loss | Sideloaded subtitles never reached the player at all |
| 2 | `android/.../exoplayer/ReactExoplayerView.java` | Android | Crash / rejected samples | `TextRenderer` rejected sideloaded VTT samples |
| 3 | `ios/Video/RCTVideo.swift` | iOS/tvOS | Silent data loss | `onLoad`/`onTextTracks` dropped manifest-incorporated tracks whenever any sideloaded track existed |
| 4 | `ios/Video/RCTVideo.swift` | iOS/tvOS | Wrong routing | Selecting an incorporated track was unreachable whenever any sideloaded track existed |
| 5 | `ios/Video/RCTVideo.swift` | iOS/tvOS | Silent data loss | Incorporated-track cues were delivered natively but never rendered on screen |
| 6 | `ios/Video/Features/RCTPlayerOperations.swift` | iOS/tvOS | Crash risk | `commonMetadata.map(\.value)[0]` force-indexes a possibly-empty array |

Items 1–5 are functional bugs found while wiring up incorporated-subtitle
support. Item 6 is an unrelated crash risk spotted along the way. All debug
`print`/`console.log` lines added for diagnosis are called out separately at
the bottom — strip those (or keep them behind a debug flag) before merging
upstream, your call.

---

## 1. Android — sideloaded subtitles silently dropped

**File:** `android/src/main/java/com/brentvatne/exoplayer/ReactExoplayerView.java`
**Method:** `buildMediaSource()`

### Bug

`buildSubtitleConfigurations()` correctly builds `MediaItem.SubtitleConfiguration`
objects for sideloaded tracks and attaches them via
`mediaItemBuilder.setSubtitleConfigurations(...)`. But the resulting
`MediaItem` is then converted to a `MediaSource` via the type-specific
factory directly (`HlsMediaSource.Factory`, `DashMediaSource.Factory`, etc.):

```java
MediaSource mediaSource = mediaSourceFactory
        .setDrmSessionManagerProvider(drmProvider)
        .setLoadErrorHandlingPolicy(...)
        .createMediaSource(mediaItem);
```

Unlike `DefaultMediaSourceFactory`, these type-specific factories **do not
read `MediaItem.subtitleConfigurations` at all** — that merge-in step (wrap
the base source + a `SingleSampleMediaSource` per subtitle config in a
`MergingMediaSource`) only exists inside `DefaultMediaSourceFactory`. The
sideloaded subtitle configs were silently dropped between `MediaItem` and
`MediaSource`. Confirmed via native track dump: only the manifest's own
embedded subtitle groups ever appeared in ExoPlayer's real track list; the
sideloaded ones never did.

### Fix

After the base `mediaSource` is built, manually replicate what
`DefaultMediaSourceFactory` does — wrap it in a `MergingMediaSource` with a
`SingleSampleMediaSource` per subtitle configuration:

```java
MediaSource mediaSource = mediaSourceFactory
        .setDrmSessionManagerProvider(drmProvider)
        .setLoadErrorHandlingPolicy(
                config.buildLoadErrorHandlingPolicy(source.getMinLoadRetryCount())
        )
        .createMediaSource(mediaItem);

// NOTE: HlsMediaSource/DashMediaSource/etc. (unlike DefaultMediaSourceFactory) do not read
// MediaItem.subtitleConfigurations themselves, so external subtitles set on the MediaItem
// above would otherwise be silently dropped here. Manually merge them in, matching what
// DefaultMediaSourceFactory does internally for sideloaded/external text tracks.
if (subtitleConfigurations != null && !subtitleConfigurations.isEmpty()) {
    MediaSource[] mediaSourcesWithSubtitles = new MediaSource[subtitleConfigurations.size() + 1];
    mediaSourcesWithSubtitles[0] = mediaSource;
    SingleSampleMediaSource.Factory subtitleSourceFactory =
            new SingleSampleMediaSource.Factory(mediaDataSourceFactory);
    for (int i = 0; i < subtitleConfigurations.size(); i++) {
        mediaSourcesWithSubtitles[i + 1] =
                subtitleSourceFactory.createMediaSource(subtitleConfigurations.get(i), C.TIME_UNSET);
    }
    mediaSource = new MergingMediaSource(mediaSourcesWithSubtitles);
}

if (cropStartMs >= 0 && cropEndMs >= 0) {
    ...
```

**New imports needed:** `androidx.media3.exoplayer.source.SingleSampleMediaSource`
(`MergingMediaSource` was already imported).

---

## 2. Android — TextRenderer rejects sideloaded VTT samples

**File:** `android/src/main/java/com/brentvatne/exoplayer/ReactExoplayerView.java`
**Method:** `initializePlayerCore()`

### Bug

Once fix #1 lands, sideloaded subtitles reach the player but playback
crashes:

```
androidx.media3.exoplayer.ExoPlaybackException: Unexpected runtime error
Caused by: java.lang.IllegalStateException: Legacy decoding is disabled,
can't handle text/vtt samples (expected application/x-media3-cues).
    at androidx.media3.exoplayer.text.TextRenderer.assertLegacyDecodingEnabledIfRequired
    at androidx.media3.exoplayer.text.TextRenderer.onStreamChanged
```

`SingleSampleMediaSource` (Media3 1.4.1, confirmed via `javap` on the actual
`.aar` — its `Factory` has no `setSubtitleParserFactory`/equivalent) always
emits the legacy raw subtitle format (e.g. `text/vtt`), never the newer
`application/x-media3-cues` format. `TextRenderer` rejects legacy samples
unless legacy decoding is explicitly enabled.

### Fix

Override `DefaultRenderersFactory.buildTextRenderers()` at player
initialization to call `TextRenderer.experimentalSetLegacyDecodingEnabled(true)`
on the renderer it builds:

```java
DefaultRenderersFactory renderersFactory =
        new DefaultRenderersFactory(getContext()) {
            // SingleSampleMediaSource (used for our sideloaded/external text tracks, see
            // buildSubtitleConfigurations()) always emits the legacy raw subtitle format
            // (e.g. text/vtt) rather than the newer application/x-media3-cues format, and
            // TextRenderer rejects legacy samples unless explicitly opted in.
            @Override
            protected void buildTextRenderers(
                    Context context,
                    TextOutput output,
                    android.os.Looper outputLooper,
                    int extensionRendererMode,
                    ArrayList<Renderer> out) {
                super.buildTextRenderers(context, output, outputLooper, extensionRendererMode, out);
                for (Renderer renderer : out) {
                    if (renderer instanceof TextRenderer) {
                        ((TextRenderer) renderer).experimentalSetLegacyDecodingEnabled(true);
                    }
                }
            }
        }
                .setExtensionRendererMode(DefaultRenderersFactory.EXTENSION_RENDERER_MODE_OFF)
                .setEnableDecoderFallback(true)
                .forceEnableMediaCodecAsynchronousQueueing();
```

**New imports needed:** `androidx.media3.exoplayer.Renderer`,
`androidx.media3.exoplayer.text.TextOutput`,
`androidx.media3.exoplayer.text.TextRenderer`.

---

## 3. iOS/tvOS — `onLoad`/`onTextTracks` drop manifest-incorporated tracks

**File:** `ios/Video/RCTVideo.swift`
**Methods:** `handleReadyToPlay()` (the `onLoad` event) and
`handleTracksChange()` (the `onTextTracks` event)

### Bug

Both events build their `textTracks` JSON payload the same way:

```swift
"textTracks": extractJsonWithIndex(from: source.textTracks) ?? textTracks.map(\.json),
```

`extractJsonWithIndex()` returns `nil` **only** when its input array is
empty. `source.textTracks` is the JS-supplied sideloaded list. So the moment
there's *any* sideloaded track, this `??` fallback to the real
manifest-incorporated tracks (`textTracks`, from `getTextTrackInfo()` /
`AVMediaSelectionGroup(for: .legible)`) never runs — incorporated tracks are
silently absent from the JS-visible list whenever any sideloaded track
exists, which in this app is effectively "always."

### Fix

Concatenate both lists (both are `[TextTrack]`) before extracting, instead of
picking one or the other:

```swift
// handleReadyToPlay():
"audioTracks": audioTracks,
// Merge sideloaded (source.textTracks) with tracks incorporated in the
// manifest itself (textTracks, from AVMediaSelectionGroup) - using ??
// here previously meant sideloading anything at all silently hid every
// embedded/manifest track from this event.
"textTracks": extractJsonWithIndex(from: source.textTracks + textTracks) ?? [],
"target": self.reactTag as Any])
```

```swift
// handleTracksChange():
let textTracks = await RCTVideoUtils.getTextTrackInfo(self._player)
// See handleReadyToPlay() - merge sideloaded + manifest-incorporated tracks rather
// than picking one or the other.
self.onTextTracks?(["textTracks": extractJsonWithIndex(from: source.textTracks + textTracks) ?? []])
```

Note the fallback changed from `textTracks.map(\.json)` /
`textTracks.compactMap(\.json)` to plain `[]` — `extractJsonWithIndex` already
returns `nil` only for a truly-empty combined list, so `[]` is the correct
"nothing to report" value in both places now.

---

## 4. iOS/tvOS — selecting an incorporated track was unreachable

**File:** `ios/Video/RCTVideo.swift`
**Method:** `setSelectedTextTrack(_ selectedTextTrack: SelectedTrackCriteria?)`

### Bug

```swift
func setSelectedTextTrack(_ selectedTextTrack: SelectedTrackCriteria?) {
    _selectedTextTrackCriteria = selectedTextTrack ?? SelectedTrackCriteria.none()
    guard let source = _source else { return }
    if !source.textTracks.isEmpty { // sideloaded text tracks
        ... RCTPlayerOperations.setSideloadedText(...) / setupAndFetchTextTracks()
    } else { // text tracks included in the HLS playlist
        ... RCTPlayerOperations.setMediaSelectionTrackForCharacteristic(..., characteristic: .legible, ...)
    }
}
```

The branch is chosen based on **"does this source have any sideloaded
tracks at all"**, not **"is the specific track the user picked one of the
sideloaded ones."** Once fix #3 makes incorporated tracks visible in the
picker, selecting one (e.g. an embedded "English" track) on a video that also
has sideloaded tracks (Chinese/French) still took the sideloaded branch,
looked for `"English"` inside `source.textTracks`, found nothing, and did
nothing — it never reached the `.legible` `AVMediaSelectionGroup` selection
path.

### Fix

Only take the sideloaded branch when the *selected* title/language actually
matches an entry in `source.textTracks`; otherwise route to the
`.legible`/incorporated path (this also covers "off", which previously never
reached the incorporated-disable path either):

```swift
func setSelectedTextTrack(_ selectedTextTrack: SelectedTrackCriteria?) {
    _selectedTextTrackCriteria = selectedTextTrack ?? SelectedTrackCriteria.none()
    guard let source = _source else { return }

    // Whether the *selected* track is actually one we sideloaded - not just whether this
    // source happens to have any sideloaded tracks at all. A video can have both sideloaded
    // (source.textTracks) and manifest-incorporated tracks at once; picking an incorporated
    // one (e.g. an embedded "English" track) must still fall through to the .legible
    // AVMediaSelectionGroup path below, even though source.textTracks is non-empty.
    let criteriaValue = _selectedTextTrackCriteria.value
    let matchesSideloadedTrack = !source.textTracks.isEmpty && criteriaValue != nil && source.textTracks.contains {
        (_selectedTextTrackCriteria.type == "language" && $0.language == criteriaValue) ||
            (_selectedTextTrackCriteria.type == "title" && $0.title == criteriaValue)
    }

    if matchesSideloadedTrack { // sideloaded text tracks
        if let uri = _source?.uri {
            /// Check for URL and Custom TextTracks as it won't work with HLS playlist https://docs.thewidlarzgroup.com/react-native-video/component/props#texttracks-1
            if uri.contains("m3u8") && source.textTracks.count > 0 {
                self.setupAndFetchTextTracks()
            } else {
                RCTPlayerOperations.setSideloadedText(player: _player, textTracks: source.textTracks, criteria: _selectedTextTrackCriteria)
            }
        }
    } else { // text tracks included in the HLS playlist (or disabling/"off")
        // Clear any custom-rendered sideloaded subtitle so it doesn't linger on screen
        // underneath/alongside the manifest-incorporated track now being selected.
        self.subtitleLabel?.isHidden = true
        self.subtitles = []

        Task { [weak self] in
            guard let self,
                  let player = self._player else { return }

            await RCTPlayerOperations.setMediaSelectionTrackForCharacteristic(
                player: player,
                characteristic: .legible,
                criteria: self._selectedTextTrackCriteria
            )
        }
    }
}
```

---

## 5. iOS/tvOS — incorporated-track cues never rendered on screen

**File:** `ios/Video/RCTVideo.swift`
**Method:** `handleLegibleOutput(strings: [NSAttributedString])`

### Bug

Once fix #4 lands, AVFoundation correctly selects the incorporated track and
starts delivering its cue text via `AVPlayerItemLegibleOutput`'s delegate,
which reaches this method:

```swift
func handleLegibleOutput(strings: [NSAttributedString]) {
    guard onTextTrackDataChanged != nil else { return }

    if let subtitles = strings.first {
        self.onTextTrackDataChanged?(["subtitleTracks": subtitles.string])
    }
}
```

This only forwards cue text to a JS event (`onTextTrackDataChanged`) — it
never touches `subtitleLabel`, the actual on-screen `UILabel` this player
renders subtitles into. That label is otherwise *only* ever updated by the
sideloaded-track pipeline (`updateSubtitles(for:)`, driven by a manually
fetched + parsed VTT file and a periodic time observer). The two pipelines
are completely disconnected, so incorporated-track cues were being generated
and delivered correctly by AVFoundation and then dropped before ever
reaching the screen. (Consuming app has nothing listening to
`onTextTrackDataChanged`, for context — but even if it did, that's a
separate/JS-side rendering path, not this label.)

### Fix

Render the cue directly onto `subtitleLabel`, attaching it with constraints
first if it isn't already in the view hierarchy (mirrors the constraints
`updateSubtitles(for:)` uses in its default/no-VTT-styling case):

```swift
func handleLegibleOutput(strings: [NSAttributedString]) {
    // Cues for a manifest-incorporated text track (selected via .legible AVMediaSelectionGroup,
    // see setSelectedTextTrack) arrive here from AVPlayerItemLegibleOutput. This used to only
    // forward them to onTextTrackDataChanged, which nothing in JS listens to - subtitleLabel
    // (the only on-screen subtitle surface in this app) was otherwise only ever driven by the
    // sideloaded-track pipeline's own manual VTT fetch + time observer (updateSubtitles). Render
    // incorporated-track cues onto that same label so both kinds of tracks actually display.
    let cueText = strings.first?.string ?? ""
    if !cueText.isEmpty {
        if subtitleLabel?.superview == nil {
            self.addSubview(subtitleLabel)
            NSLayoutConstraint.activate([
                subtitleLabel.bottomAnchor.constraint(equalTo: self.bottomAnchor, constant: -responsiveSize(15.0)),
                subtitleLabel.centerXAnchor.constraint(equalTo: self.centerXAnchor),
                subtitleLabel.widthAnchor.constraint(lessThanOrEqualTo: self.widthAnchor, multiplier: 0.8),
                subtitleLabel.heightAnchor.constraint(greaterThanOrEqualToConstant: 20.0),
            ])
        }
        subtitleLabel?.textAlignment = .center
        subtitleLabel?.text = cueText
        subtitleLabel?.isHidden = false
    } else {
        subtitleLabel?.isHidden = true
    }

    if onTextTrackDataChanged != nil, let subtitles = strings.first {
        self.onTextTrackDataChanged?(["subtitleTracks": subtitles.string])
    }
}
```

---

## 6. iOS/tvOS — crash risk on empty `commonMetadata`

**File:** `ios/Video/Features/RCTPlayerOperations.swift`
**Method:** `setMediaSelectionTrackForCharacteristic()`

### Bug (unrelated to the subtitle feature — spotted in passing)

```swift
optionValue = currentOption.commonMetadata.map(\.value)[0] as? String
```

If `commonMetadata` is ever empty for a given `AVMediaSelectionOption`, this
force-indexes an empty array and **crashes**. `getTextTrackInfo()` elsewhere
in the codebase already guards the equivalent lookup
(`(values?.count ?? 0) > 0`) — this call site didn't.

### Fix

```swift
// .first instead of [0] - commonMetadata can be empty and would otherwise crash.
optionValue = currentOption.commonMetadata.map(\.value).first as? String
```

---

## Debug logging

All temporary debug logging used to diagnose the above (JS-side
`console.log('[SubtitleDebug] ...')` calls, the app's
`debug={{ enable: true, thread: false }}` prop passed into `<Video>`, and the
native `print("[SubtitleDebug-iOS] ...")` lines in `RCTVideo.swift` /
`RCTPlayerOperations.swift`) has been removed now that the fixes are
confirmed working. The patch/snippets in this doc reflect the clean,
log-free versions. If you hit a regression while porting, temporarily
re-adding similar logging at the same call sites (`setSelectedTextTrack`,
`setMediaSelectionTrackForCharacteristic`, `handleReadyToPlay`) is the fastest
way to see what's happening.

---

## Porting checklist

1. Check out the fork branch (`ott_sideload_text_tracks_6.19.2` or a new one).
2. Apply `patches/react-native-video+6.19.2.patch` directly
   (`git apply --directory=<fork-repo-root> ...` after stripping the
   `node_modules/react-native-video/` path prefix — patches path is relative
   to `node_modules/react-native-video/` in this app's checkout, e.g.
   `android/src/main/java/...`, `ios/Video/...`), or copy the snippets above
   by hand.
3. Decide whether to keep the `[SubtitleDebug-iOS]` print statements (see
   above) — recommend stripping for a release build, or gating behind an
   existing debug flag.
4. Commit, push, tag/bump as appropriate on the fork repo.
5. In this app's `package.json`, update the `react-native-video` dependency's
   git ref to the new commit/tag.
6. Run `npm install`, confirm `patches/react-native-video+6.19.2.patch` no
   longer has anything to apply (or delete it), delete this file if no
   longer needed as a changelog.
