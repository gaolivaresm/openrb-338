# Writing a song: the beat cookbook

How to make a song from scratch and hear it, without a DAW: `tools/rbs.py build` writes an `.rbs`
from a JSON spec, and `orb render` turns it into audio. Everything here was checked against the
repository's own code and songs, not written from memory.

The 1997 machine has three instruments and no sampler:

| Machine | In the song | What it is | Good for |
|---|---|---|---|
| TR-808 | `tr808` | the boomy drum machine | hip-hop, trap: the kick and the cowbell |
| TR-909 | `tr909` | the 80s drum machine | house, techno, boom-bap: a tighter kick and snare |
| TB-303 | `tb303` | one monophonic bass line with a resonant filter | bass lines; acid house by nature |

No piano, no strings, no samples, no long arrangement. It is patterns of up to 16 steps, and a
song that chains patterns.

## The short way

```sh
cd ~/Proyectos/openrb-338

cat > /tmp/beat.json <<'JSON'
{
  "glob": { "tempo": 90000, "song_mode": 1, "loop": 1, "loop_start": 0, "loop_end": 3072 },
  "tr909": { "patterns": { "0": { "length": 16, "steps": {
      "0": [0,1,0,0,0,0,0,0,0,0,0,0],
      "4": [0,0,0,0,0,0,0,0,0,1,0,0],
      "8": [0,1,0,0,0,0,0,0,0,0,0,0],
      "12":[0,0,1,0,0,0,0,0,0,1,0,0]
  } } } },
  "tracks": { "tr909": [ {"pos": 0, "ctrl": "pattern", "value": 0} ] }
}
JSON

python3 tools/rbs.py build /tmp/beat.json /tmp/beat.rbs
python3 tools/rbs.py check /tmp/beat.rbs          # reads back byte for byte
./target/release/orb render /tmp/beat.rbs -o /tmp/beat.wav
xdg-open /tmp/beat.wav
```

`build` merges the spec over the template song (`tests/songs/shipped/Silent Default Song.rbs`):
every field you leave out keeps the template's value, so a spec names only what it changes.

## The three levels of a spec

```
glob        tempo (0.001 BPM units), song_mode, loop, loop_start, loop_end (ticks)
tr808       params (levels and tunings) and patterns
tr909       params and patterns
tb303       a LIST of two: index 0 is the first 303, index 1 the second
mixer       master, and one channel per machine: level, pan, delay_send, dist
delay, pcf, dist, comp    the effect settings
tracks      the events: which pattern plays when, and every knob change
```

`tempo` is in **thousandths of a BPM**: `90000` is 90 BPM, `140000` is 140 BPM.

### The drum patterns

`steps` maps a step index (`"0"`…`"15"`) to a row of 12 numbers, one per column. The column
order is fixed and differs between the two machines (`DRUM_COLUMNS` in `tools/rbs.py`):

```
tr808: accent, bass, snare, low_tom, mid_tom, hi_tom, rim_shot, clap, cow_bell,
       cymbal, open_hihat, closed_hihat
tr909: accent, bass, snare, low_tom, mid_tom, hi_tom, rim_shot, clap,
       closed_hihat, open_hihat, crash, ride
```

> The last four are **swapped** between the two machines. Check the list before you write a row.

A value means, for the 808: `0` off, `1` on. For the 909: `0` off, `1` on, `2` accent, `3` flam.

`length` is `1` to `16` (the pattern's steps; the rest of the 16 are silent).

### The 303 patterns

A step is an object, not a bare number (`Step303` in `tools/rbs.py`):

```json
"0": {"note": 1, "rest": 0, "accent": 1, "slide": 0}
```

| Field | Range | Meaning |
|---|---|---|
| `note` | 0 to 12 | tone/pitch, a semitone ladder (`docs/RBS42.txt`: `0x00` to `0x0c`) |
| `rest` | 0 or 1 | **set = rest** (silence), the opposite of what you would guess |
| `accent`, `slide` | 0 or 1 | accent, and a slide into the next note |
| `up`, `down` | 0 or 1 | transpose up / down |

A step left out of the spec keeps the template's, which is `rest: 1` — silence. So you write only
the notes, and the holes stay silent.

`note` is a scale position, not a MIDI note, so what you get depends on the 303's `tune` and the
song's `vintage` setting. Write the pattern, render, and listen: there is no table that turns
`note: 5` into a named pitch without running the engine.

### The 303's knobs

| Parameter | Range | What it does |
|---|---|---|
| `cutoff` | 0 to 127 | the filter: low is dark and round, high is bright and thin |
| `resonance` | 0 to 127 | high makes the acid "wah"; near 127 it self-oscillates |
| `env_mod` | 0 to 127 | how much the envelope opens the filter on each note |
| `decay` | 0 to 127 | the envelope's tail |
| `accent` | 0 to 127 | how hard an accented step hits |
| `tune` | 0 to 127 | the base pitch |
| `waveform` | 0 or 1 | `0` is the saw (verified: `docs/song_load.md` section 10) |

For a low hip-hop bass line: low `cutoff` (30 to 45), low `accent`, short `decay`. A high
`resonance` above 70 with a loud `env_mod` is the 1997 acid sound, which is probably not what a
beat wants.

## Making it a song, not a one-bar loop

A template is in **pattern mode** (`song_mode: 0`) with a one-bar loop, so it plays one pattern
and stops. To get four bars:

```json
"glob": { "song_mode": 1, "loop": 1, "loop_start": 0, "loop_end": 3072 }
```

`loop_end` is in ticks, and one bar is **768 ticks** (`TICKS_PER_BAR`), four beats of 192. So
`0` to `3072` is four bars; `0` to `768` is one.

Then `tracks` says which pattern plays when, and every knob change, in 32nd notes (32 per bar):

```json
"tracks": {
  "tb303_1": [ {"pos": 0, "ctrl": "pattern", "value": 0} ],
  "tr909":   [ {"pos": 0, "ctrl": "pattern", "value": 0} ],
  "mixer":   [ {"pos": 0, "ctrl": "tr909_level", "value": 110} ]
}
```

The names are `mixer`, `tb303_1`, `tb303_2`, `tr808`, `tr909`, `delay`, `dist`, `pcf`, `comp`.

## Checking it is not silence

The template song really is silent, so an empty build renders zeroes and looks like success.
Check the level:

```sh
python3 - /tmp/beat.wav <<'PY'
import wave, audioop, sys
w = wave.open(sys.argv[1])
d = w.readframes(w.getnframes())
print("frames", w.getnframes(), "seconds", round(w.getnframes() / 44100, 2),
      "RMS", audioop.rms(d, 2), "peak", audioop.max(d, 2))
PY
```

`RMS 0` and `peak 0` mean silence: the spec named no steps, or the pattern is not playing (a
machine's `enabled` is 0, or `tracks` names no pattern for it). Any non-zero RMS is sound.

## Rendering

| Want | Command |
|---|---|
| the whole loop | `orb render SONG.rbs -o out.wav` |
| bars 0 to 4 | `orb render SONG.rbs -o out.wav --range 0:4` |
| FLAC or Ogg, for smaller files | `-o out.flac` or `-o out.ogg` |
| listen from the shell | `orb play SONG.rbs --seconds 11` |

`--range` counts bars from 0 and END is exclusive; `orb` refuses an END past bar 1000 and a START
not below END. `.wav` is 16-bit stereo 44100 Hz, the bytes the engine exports.

> A long range writes a big file: bar 1000 is over 400 MB of WAV. Use `.flac` or `.ogg` for
> anything long.

`orb play` opens the sound card and plays with **no window and no player**: it is audio only, and
it ends at the song's end or at `--seconds`. Nothing on screen says it is running; `pgrep -af 'orb
play'` finds it. Do not leave one playing by accident: it is heard, and nothing shows it.

## From an MCP client

With the `orb` server loaded (`docs/mcp.md`), the same steps are tools:

```
orb_render  {song, output, range?, cwd}
orb_info    {song, cwd}
orb_compare {song, reference, cwd}
```

`cwd` is the repository root, because the server is a separate process and its own working
directory is not the client's. No tool writes a song: `tools/rbs.py build` is a Python tool, run
from the shell.

## A worked example

`~/Musica/rebirth/hiphop-90bpm.rbs` is a beat of four bars at 90 BPM: a 909 with a kick on 1 and
3, a snare on 2 and 4, eighth-note hats, an open hat and a syncopated kick; and a 303 line with a
low cutoff (38) and a high resonance (82) for a dark bass. It renders to 10.7 s. Its level is RMS
2653, against RMS 0 for the silent template it was built from.

## More songs to learn from

`tests/songs/` holds the corpus: `diag/` has one song per engine feature, and `shipped/` the songs
that came with the original. Read any of them with:

```sh
python3 tools/rbs.py dump SONG.rbs     # human readable
python3 tools/rbs.py json SONG.rbs     # the model, the same shape `build` takes
```

Copying a `diag` song and changing one field is the fastest way to learn what each knob does.

