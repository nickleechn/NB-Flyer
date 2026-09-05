# NB Fly Video Edit

How a flight review or travel video gets cut, from a folder of raw clips and a
recorded voice-over to a finished 4K upload with subtitles and metadata.

## House rules

These are defaults. Do not ask about them each time.

- **Always stabilise.** The cameras are cheap and most handheld shots need it.
  Stabilisation is on by default, not something to enable selectively.
- **Shoot 60 fps, deliver 30 fps.** Source is 59.94p; every export is 30p.
- **Aim for 4K.** Deliver 3840 × 2160. Never downscale to 1080p unless asked.
- **Use 50% speed to fill a gap.** If a shot is too shaky to use at speed, or a
  beat in the script needs more screen time than the clip has, slow it to 50%.
  60p conformed to 30p is *exactly* 2× slow motion with every source frame
  used — perfectly smooth, no interpolation, no frame blending.
- **The voice-over leads the edit.** Cut picture to the VO's sentence
  boundaries, not the other way round.
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

### 2. Transcribe the voice-over with timings

```bash
whisper-cli -m ggml-small.en.bin -f vo16k.wav -osrt -of vo
```

Those timings are the edit's skeleton. Every picture cut lands on a VO beat.

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

### 5. Render each cut, stabilised

Two-pass vidstab per cut, at native 4K:

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

- `smoothing=15` (about half a second at 30p). Larger windows are worse on
  short cuts — a 40+ frame window on a 100-frame cut fights the intentional
  camera move and makes the result *shakier*. Measured: 15 → 21% less jitter,
  40 → 7% worse, 80 → far worse.
- `optzoom=1` crops just enough to hide the edges. Check one output frame
  against its source to confirm the zoom is modest and there are no black
  borders.
- Shots through a train or aircraft window are the one case to eyeball after
  stabilising: the scene moves while the camera is still, so verify vidstab
  followed the camera rather than the scenery.
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

VO normalised to −16 LUFS, music ducked underneath by sidechain:

```bash
[mus][vo_copy]sidechaincompress=threshold=0.03:ratio=8:attack=15:release=500[ducked];
[vo][ducked]amix=inputs=2:duration=first:normalize=0,alimiter=limit=0.95
```

`amix` halves levels unless you pass `normalize=0`. Target −16 LUFS
integrated and about −1 dBFS true peak.

Loop a short music bed with `acrossfade` between copies rather than butting
them together.

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
edit rather than invented.

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
