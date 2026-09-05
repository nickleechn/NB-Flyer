# NB Fly Video Edit

How a flight review or travel video gets cut, from a folder of raw clips and a
recorded voice-over to a finished 4K upload with subtitles and metadata.

## House rules

These are defaults. Do not ask about them each time.

- **Select calm footage before applying a fix.** Long recordings are normal:
  the presenter cannot keep stopping to film separate shots while travelling.
  Treat each take as a source of usable segments, not a continuous sequence
  that must appear in the edit. Avoid sweeping gimbal moves, repeated turns,
  walking bob and large camera movements that can make viewing uncomfortable.
- **Match stabilisation to the selected segment.** Prefer a settled view;
  then choose standard correction, gentle correction, camera fix, or no
  correction as appropriate. Do not use stabilisation as a reason to retain
  an otherwise uncomfortable segment.
- **Shoot 60 fps, deliver 30 fps.** Source is 59.94p; every export is 30p.
- **Aim for 4K.** Deliver 3840 × 2160. Never downscale to 1080p unless asked.
- **Use 50% speed for suitable footage.** Slow a calm, usable segment when
  a beat needs more screen time. Do not slow a sweeping or shaky movement
  simply to make it usable; select a steadier section instead. Retain the
  established 60p-source / 30p-delivery workflow.
- **Let the story breathe.** Use narration beats to organise the edit, but
  do not make the video end at the voice-over's original runtime. Insert
  intentional narration pauses to showcase the footage, with music carrying
  those moments. Do not fill every second with speech.
- **Preserve a good cut.** When revising an approved edit, reuse its FCPXML
  and shot order. Change the requested processing and extend suitable shots
  for breathing room rather than rebuilding the sequence unnecessarily.
- **Never repeat a shot.** Two different sections of one long take are fine;
  the same moment twice is not.
- **Cut the script before reusing footage.** If a chapter runs out of unique
  material, drop a line of VO rather than padding with a repeat.

## Inputs

- A folder of camera clips (DJI naming: `DJI_<YYYYMMDDHHMMSS>_<seq>_D.MP4`).
  The filename timestamp sorts chronologically even when the sequence number
  resets, so sort on filename.
- A recorded voice-over, and the script it was read from.
- A music bed.

## Pipeline

### 1. Catalogue the footage before deciding anything

Never edit from filenames. Pull one frame from the middle of every clip,
label it with index, timestamp and duration, and tile them into contact
sheets to actually look at.

```bash
ffmpeg -ss "$mid" -i "$clip" -frames:v 1 -vf scale=426:240 -q:v 4 out.jpg
magick "$t" -gravity North -background black -splice 0x24 \
  -font "/System/Library/Fonts/Avenir Next.ttc" -pointsize 18 -fill yellow \
  -annotate +0+3 "#$i  $hhmmss  ${dur}s" labelled.jpg
magick montage labelled/*.jpg -tile 4x4 -geometry +3+3 sheet.jpg
```

For long takes, pull 4 frames instead of 1 — the midpoint frame lies about
what a 60-second clip contains.

### 1a. Break long recordings into usable segments

Review long takes at multiple points and inspect motion around candidate
in/out points. Mark separate source ranges for settled compositions or clear
story details, and exclude the movement connecting them. One recording may
provide several non-overlapping shots at different places in the edit.

For train or aircraft boarding, extract the useful beats separately: the
entrance or carriage/aircraft identification, the doorway, a settled cabin
view, and the seat. Use only the beats actually present. Cut out the long
walk through the platform, jet bridge or aisle when it contains substantial
camera movement. Do not keep an uninterrupted boarding walk merely to show
chronology; distinct shots can communicate the same progression.

Trim camera repositioning, gimbal sweeps, abrupt pans and tilts, walking bob,
and turns between subjects. Let the camera settle before the chosen in-point
and cut before the next repositioning. Small connecting movements may remain
when they are comfortable and useful, but prefer the stable portions. If no
comfortable segment exists, omit the shot or use another unique view.

Avoid a string of rapid cuts or conflicting movement directions when splitting
a take. Give each selected view enough time to register. For music-only
breathing room, choose calm footage rather than extending a moving-camera
walk. Record each segment's own source in/out points in the EDL and retain
the existing rule against overlapping or repeated source moments.

### 2. Transcribe the voice-over with timings

```bash
whisper-cli -m ggml-small.en.bin -f vo16k.wav -osrt -of vo
```

Those timings are the edit's initial skeleton, not a fixed runtime. Keep a
map from the original narration to the revised timeline whenever pauses or
trims change its placement.

### 2a. Leave room for the footage

Add short, deliberate music-only passages at natural transitions: a landscape
opening up, an arrival, an evening walk, or a quiet view on the return leg.
Choose the number and length for the footage rather than inserting a pause
after every sentence. A few seconds can be enough; there is no requirement
to keep a four-minute voice-over inside a four-minute video.

Split narration only inside a verified gap between complete phrases or
sentences. Insert silence without stretching the spoken audio or cutting
off a word. Keep tiny splice fades inside those gaps to avoid clicks. Extend
suitable moving footage, use an unused source range, or use the established
50% slow-motion option when appropriate. Do not freeze a frame or repeat a
source moment just to fill a pause. Recheck source capacity and overlaps.

Record each inserted pause and remap the narration, cuts, subtitles and
chapters consistently. A subtitle that crosses a pause needs splitting or
retiming so it does not hang over the music-only passage. Use the revised
picture runtime when sizing or looping the music.

### 3. Build an EDL and validate it mechanically

One line per cut: `timeline_start, clip_index, source_in, note`. Derive each
cut's duration from the next cut's start. Then check by script, not by eye:

- No two cuts may use overlapping source ranges from the same clip.
- No cut may be longer than its source clip (minus a safety margin).
- Cuts that are short get slowed rather than stretched past the source end.

### 4. Trim the voice-over where footage runs out

Cut whole sentences, on silence, never mid-phrase.

```bash
ffmpeg -i vo.mp3 -af "silencedetect=noise=-35dB:d=0.6" -f null -
```

Pick splice points *inside* detected silences, rebuild with short crossfades,
then re-transcribe the joins to confirm they still read naturally.

Keep a map from original VO time to new timeline time and apply it to
everything downstream — cut list, chapter markers, subtitles.

### 5. Choose stabilisation per shot, then render

Classify each cut before rendering and record its mode and settings in the
EDL. Do not apply the same correction strength to every shot.

| Shot | Starting approach | What must remain natural |
|---|---|---|
| Ordinary handheld shot of a mostly static scene | Standard two-pass stabilisation | Small shake is reduced without drifting framing |
| Gimbal sweep, boarding walk, repeated turn or other large camera move | Exclude the moving passage; extract settled segments before and after it | Clear progression without prolonged camera travel |
| Small useful residual camera movement in a selected segment | Gentle correction only if the movement remains comfortable | Natural motion without correction snapping or catching up |
| Nearly stationary camera on a mostly static scene | Consider camera fix / a locked-frame treatment | Stable framing with live subject movement and a modest crop |
| Scenery through a moving train or aircraft window | Compare a gentle pass with the untreated source; prefer the source if correction follows the scenery | Passing objects and parallax, without sudden pulls, tilts or zoom changes |

**Camera fix means holding the framing steady, not freezing the video.**
For a short, nearly static segment with enough image area to crop, consider
an available tripod/locked-frame mode or fixed-reference stabilisation.
Preview the result before accepting it. Reject excessive crop, warped edges,
black borders, or sudden framing shifts. Do not try to pin a large pan,
forward boarding walk or strong parallax to one frame; select a settled
segment instead. Do not lock to scenery passing outside a moving vehicle.
If the tool cannot produce a clean camera fix, use a lighter treatment or
another segment.

Treat these as starting choices, not guaranteed fixes. For window shots,
use a reliable camera-fixed reference or tracking region only when the tool
supports it and the shot provides one. Do not assume the passing landscape
is a stable reference. If the available stabiliser cannot separate scene
motion from camera shake, bypass it rather than forcing a stronger pass.

Preview the complete selected cut in motion at delivery speed, including
its beginning and end, against the untreated source. Check for rubbery
movement, horizon tilting, framing jumps, crop changes, and the camera
appearing to resist then catch up with a pan. A contact-sheet frame cannot
validate stabilisation. Reduce correction, choose a calmer unique source
range, or omit the segment if uncomfortable movement remains. Bypass
correction only when the untreated selected footage itself is comfortable. Slow motion is a pacing option,
not a repair for bad tracking or warping.

For the standard mode, the following is a two-pass vidstab starting point
at native 4K; adapt the settings after the motion preview:

```bash
# pass 1 — analyse, with 2s of padding either side for a smoother path
ffmpeg -ss "$PAD_START" -t "$PAD_DUR" -i "$src" \
  -vf "fps=30,vidstabdetect=shakiness=4:accuracy=9:stepsize=12:result=$trf" -f null -

# pass 2 — transform, then trim to the exact cut
ffmpeg -ss "$PAD_START" -t "$PAD_DUR" -i "$src" \
  -vf "fps=30,vidstabtransform=input=$trf:smoothing=15:optzoom=1:interpol=bicubic,\
unsharp=5:5:0.4:3:3:0.2,trim=start=$OFF:duration=$D,setpts=PTS-STARTPTS,format=yuv420p" \
  -c:v h264_videotoolbox -b:v 60M -y cut.mp4
```

For a 50% shot, slow it before the fps conversion so every source frame lands
on an output frame:

```bash
-vf "setpts=2.0*PTS,fps=30,vidstabtransform=..."
```

Notes that matter:

- `smoothing=15` is a starting point for the standard mode, not a universal
  setting. A large smoothing window can fight an intentional camera move;
  compare a lower setting for gentle mode instead of increasing smoothing
  to compensate for large scene motion.
- `optzoom=1` crops just enough to hide the edges. Check one output frame
  against its source to confirm the zoom is modest and there are no black
  borders.
- Inspect every corrected cut in motion. Give passing scenery and large
  camera moves particular attention using the mode selection above.
- Pad the analysis window either side of the cut. vidstab needs context to
  build a smooth path, and a 3-second cut on its own gives it almost none.

### 6. Titles as PNGs, not drawtext

Homebrew's ffmpeg has no libfreetype, so `drawtext` is unavailable. Render
title cards and lower-thirds with ImageMagick at full 4K and overlay them.
Better typography anyway.

Give each overlay its own short input and place it with `setpts`, so ffmpeg
only generates frames for the window it appears in:

```bash
-loop 1 -framerate 30 -t "$DUR" -i card.png
# ...
[2:v]format=rgba,fade=t=in:st=0:d=0.5:alpha=1,fade=t=out:st=...:d=0.6:alpha=1,
setpts=PTS-STARTPTS+12.59/TB[t0];
[0:v][t0]overlay=enable='between(t,12.59,18.89)':eof_action=pass[v0]
```

Emoji and arrow glyphs are missing from most system fonts — draw arrows as
shapes rather than typing `→`.

### 7. Audio

Keep the voice level even from passage to passage. Use gentle vocal
compression or phrase-level gain where needed, preserving natural expression
and quiet gaps; integrated loudness alone does not guarantee consistent
speech. Avoid audible pumping or boosted breaths.

Keep the music clearly audible at a steady background level through narration. Do not use
sidechain compression, speech-triggered ducking, or volume dips whenever
the presenter speaks. Choose the music gain once for the mix, so the voice
remains clear while the music stays audible and consistent. If the bed is
hard to hear, raise its fixed level rather than only lifting it during pauses. Check a spoken
passage and a pause at the same playback volume.

Normalise the voice-over as needed, set the music's fixed gain by listening,
then mix with `amix=inputs=2:duration=first:normalize=0`. With `normalize=0`,
the mixer does not automatically reduce the input levels. Measure the final
combined mix and target −16 LUFS integrated with true peak no higher than
−1 dBTP. Correct the overall mix gain or limiting as needed without adding
speech-triggered music changes.

Opening and closing fades are fine. Loop a short music bed with `acrossfade`
between copies rather than butting them together. Loop the selected track
as needed to cover the revised runtime, choosing compatible musical phrases
and checking the join for clicks or a sudden level change. These transitions
should follow the edit or track boundaries, not voice activity.

### 8. Assemble frame-exactly

**This is where sync is won or lost.** Pin every cut boundary to an absolute
frame number computed from the timeline, and give each cut an exact frame
count:

```python
F = [round(t * 30 / speed) for t in beat_times]   # absolute frame per beat
N = [F[i+1] - F[i] for i in range(len(F)-1)]      # frames per cut
```

Then render each cut with `-frames:v $N` and concat with `-c copy`. Verify
`sum(N)` equals the total frame count of the finished file before shipping.

### 9. Subtitles

Transcribe the **original-speed** voice-over, not the final mix and not a
time-compressed version — recognition degrades on sped-up audio and quietly
drops words. Remap those timings through the same cut/speed map used for the
picture.

When remapping, a segment that *straddles* a removed region must be clamped,
not dropped, or whole sentences vanish silently.

Then correct proper nouns against the script — Rapi:t, Swissôtel,
Häagen-Dazs, Aesop, Suntory, flight numbers.

Target: ≤ 2 lines, ≤ 42 characters per line, ≥ 1s per cue, no overlaps.

## Gotchas that cost real time

- **`-ss` after `-i` is an output option and applies *after* the filter
  chain.** Combining it with a `trim` filter double-trims: the clip comes out
  short *and* starts late. Use input-side seeking (`-ss`/`-t` before `-i`) when
  a filter also trims.
- **The concat demuxer's `outpoint` is not frame-exact.** It dropped roughly
  one frame per segment — 43 frames over 119 cuts, which is 1.4 seconds of
  drift by the end. Re-encode each cut to an exact frame count instead.
- **`ffprobe -count_frames` can print the value twice.** Pipe through
  `head -1` before parsing.
- **whisper.cpp word tokens carry their own leading spaces.** Concatenate them
  directly; joining with `" "` turns "Nankai" into "N ank ai".
- **Long clips need multi-frame thumbnails.** A midpoint frame of a 64-second
  take told me nothing about the approach and touchdown inside it.

## Output spec

| Property | Value |
|---|---|
| Resolution | 3840 × 2160 |
| Frame rate | 30 fps constant |
| Video codec | HEVC, ~55 Mbps, `-tag:v hvc1` |
| Audio | AAC 256k, 48 kHz stereo |
| Loudness | −16 LUFS integrated, ≈ −1 dBFS true peak |
| Container | MP4, `+faststart` |

Also ship: an `.srt`, an FCPXML matching the delivered cut, and a metadata
file with title options, description and chapter timestamps taken from the
edit rather than invented. For revisions, keep the previous export available
and identify the new version clearly. Supply separate narration and music
tracks in the FCPXML so their levels remain adjustable in Final Cut Pro.

## Tooling

- `ffmpeg` with `--enable-libvidstab`. Homebrew's default build does not
  include it; build from source against `libvidstab` rather than replacing the
  system ffmpeg.
- `whisper-cpp` + a `ggml-small.en` model for transcription.
- `imagemagick` for title cards and contact sheets.
- VideoToolbox encoders (`h264_videotoolbox`, `hevc_videotoolbox`) — roughly
  an order of magnitude faster than x264/x265 for this, and fine at these
  bitrates.

Run per-clip work in parallel with `xargs -P`; it is the difference between
three minutes and half an hour.
