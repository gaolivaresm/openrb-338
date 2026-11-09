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

### Accent is not a sound

The first column of both machines is `accent`, and it is not a drum. A step with accent makes
**every** sound of that step louder, by the same amount, and the machine's `accent_level` knob sets
that difference. It is how a flat pattern gets life: a closed hi-hat on all sixteenths with accents
on the odd steps is the classic example (manual, page 35).

### Open and closed hi-hat exclude each other

Open hi-hat and closed hi-hat cannot sound together (manual, page 35):

- A closed hi-hat cuts off an open one that is still ringing.
- On the **808**, both on the same step give a very short open hi-hat, which sounds like a closed
  one.
- On the **909**, both on the same step give the open hi-hat.

So an open hi-hat followed by a closed one is a two-step gesture, not two independent sounds.

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

**Slide goes on the note you slide *from*, not the one you slide to** (manual, page 45). The step
with `slide` ties into the next one and the pitch glides continuously. A slide from a rest still
glides: the glide starts at the silent step's pitch and lands on the next note's.

`note` is a scale position, not a MIDI note, so what you get depends on the 303's `tune` and the
song's `vintage` setting. Write the pattern, render, and listen: there is no table that turns
`note: 5` into a named pitch without running the engine.

### The 303's knobs

A knob has two homes in the file, and only one of them is heard:

- The **chunk** (`tb303[0].params`) is what the panel shows when the song opens.
- The **track** (`tracks.tb303_1`) is a list of knob events, and **the automation wins**: in song
  mode the engine follows the events, not the chunk.

Setting `cutoff` in the chunk and nothing in the track gives you silence for that knob. This is
the easiest way to lose an afternoon: the file looks right, the value is there, and the bass does
not change when you edit it. Write the knobs as track events:

```json
"tracks": {
  "tb303_1": [
    {"pos": 0, "ctrl": "cutoff",    "value": 52},
    {"pos": 0, "ctrl": "resonance", "value": 35},
    {"pos": 0, "ctrl": "env_mod",   "value": 45},
    {"pos": 0, "ctrl": "decay",     "value": 62},
    {"pos": 0, "ctrl": "accent",    "value": 58},
    {"pos": 0, "ctrl": "tune",      "value": 64},
    {"pos": 0, "ctrl": "waveform",  "value": 0},
    {"pos": 0, "ctrl": "pattern",   "value": 0}
  ]
}
```

Check it: change `cutoff` to 20 and to 90 and render both. If the two WAVs are byte-identical, the
knob is not reaching the engine. (`md5sum` on the two files answers it at once.)

A second trap, which caught this document's own first beat: the knobs went into `tb303_2`'s track
while the notes were in `tb303_1`'s. Both machines are in the song, `tb303_1` plays, and the knob
events sit on the one that does not. Keep every event of an instrument on that instrument's track.

| Parameter | Range | What it does |
|---|---|---|
| `cutoff` | 0 to 127 | the filter: low is dark and round, high is bright and thin |
| `resonance` | 0 to 127 | routes the filter's output back to its input: a hollow, pronounced character. With envelope modulation, slide and accent, it makes the 303 what it is |
| `env_mod` | 0 to 127 | how far the envelope opens the filter. At its lowest it barely acts; higher makes each note brighter and then darker as it decays |
| `decay` | 0 to 127 | the length of that envelope: fast or slow dulling of each note |
| `accent` | 0 to 127 | how different an accented step sounds from the others, in volume and in character (the manual, page 49) |
| `tune` | ±1 octave in semitones | the base pitch, for putting patterns in a key |
| `waveform` | 0 or 1 | `0` is the sawtooth (all harmonics, full), `1` the square (odd harmonics only, hollow) |

For a low hip-hop bass line: low `cutoff` (30 to 45), low `accent`, short `decay`. The 303 is a
monophonic acid synth by design: a high `resonance` with a loud `env_mod` is its signature sound,
and it is not what a plain bass line usually wants.

> The `Envelope Modulation` and `Decay` descriptions are the original manual's (page 48 to 49,
> `docs/propellerhead_rebirth_rb338_v2.01_manual.pdf`), which is the authority on what the knobs
> are meant to do; the code only says what they compute.

## Shuffle (swing)

Shuffle has two parts, and they live in different places:

- **Amount**, one for the whole song: `glob.shuffle`, `0` to `127` (`docs/song_load.md`, GLOB[14]).
  It delays the sixteenth notes that fall between the eighths, which is the swing feel.
- **On or off, per section**: each pattern carries a shuffle flag, so the 808 can swing while the
  909 stays straight (manual, page 21: "activate Shuffle for the desired sections").

The manual's pointer for the amount: **around "two o'clock" you get a classic triplet feel.** Its
knob is drawn as a dial, so the 0 to 127 value at two o'clock is about 85; the template's 64 is a
milder swing. `0` is straight.

> This is an estimate from the dial's geometry, not a measured value: the manual names a clock
> position, not a number. Check it by ear.

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

## Tricks from the people who used it

Two decades of community knowledge, and none of it is in the code. The sources: Peff's Keyboard
Magazine article of July 2000 ("ReBirth of the Cool", Kurt Kurasaki, six pages, with example songs
in `examples/`) and the 320 mods of the Computer Controlled Underground archive
(`github.com/peffre/ReBirth-RB-338`). Neither is in git; both are read locally.

### 32-step patterns out of 16-step ones

The 909 and the 808 cannot hold a 32-step pattern. Use two bars of a song loop:

1. Loop locators over two bars, bar set to the loop start.
2. Record, select pattern A1, advance one bar, select A2, stop.
3. Program the loop as if it were pattern mode.

Knob settings reset each cycle, so they must be recorded into the loop. For 32nd-note rolls, double
the tempo and double the delay steps with it: at 133 BPM, 266 BPM and 6 delay steps give each step
the length of a 32nd note. The article notes the app's ceiling of 500 BPM, which caps a 32-step
pattern at 250; the file format itself stores the tempo as an integer of thousandths, so the limit
is the program's, not the format's.

### Three beats to the bar

Song mode does not really handle patterns shorter than 16 steps: a 12-step pattern plays and
cycles, but a pattern change replays its first four steps first. Two ways round it: change the
12-step pattern every three bars, or use three patterns (48 steps) for four bars of 3/4. Using 12
steps for a whole bar also means the real tempo is 75% of the displayed one; 160 shown is 120
played.

### Toms in tune

The tunable 808 and 909 toms take musical samples in many mods, so they can play melodies. The
808's tuning knob has 36 frames and the 909's 41, and the article gives the cents of each. On the
808: frame 18 is the sample's own pitch, frame 22 is +100 cents, frame 14 is -100 cents, so every
four or five frames is a semitone. On the 909 those land on frames 20, 24 and 17, and its first and
last positions are not chromatic.

### The delay is an instrument, not just an effect

This is the trick that turns one 303 into a chord:

- **Plucked 303**: the COMP with its ratio at maximum and its threshold near nine o'clock gives the
  attack its pop.
- **Polyphonic 303**: a short eight-step arpeggio with slide, mix delay at maximum, delay steps 2,
  feedback near two o'clock. The delay sustains the notes over each other and they become a chord,
  leaving the second 303 free for a bass line.
- **Reverb**: delay steps at the shortest (a sixteenth triplet), pan centred, feedback at 90%. A
  mono reverb whose length is the feedback.
- **Loop sampler**: feedback at maximum loops for ever, and delay steps at 16 holds one bar of
  everything. Feedback to zero stops it.

The article's example songs use exactly these: `chords01.rbs` has delay steps 2 and feedback 77,
which is the polyphonic 303 setting.

### Wah and filter sweeps

Set the PCF to band pass, amount 30%, decay 75%, Q 80%, then record PCF frequency changes in song
mode. Opening and closing it over a 303 line is the wah effect.

### Check that a knob moved

A change that reaches the engine changes the audio. Render twice and compare with `md5sum`: equal
files mean the knob never reached the engine. That test is what exposed the track-versus-chunk trap
above, and it is the fastest way to tell a real parameter from a decorative one.


## Where this comes from

Three sources, and each knows something the others do not:

- **The code and the songs** (`tools/rbs.py`, `crates/`, `tests/songs/`) say what a value computes
  and what a field is called.
- **The original manual**, `docs/propellerhead_rebirth_rb338_v2.01_manual.pdf` (232 pages, Ernst
  Nathorst-Böös, Propellerhead 1998), says what a knob is for. The code cannot tell you that a
  closed hi-hat cuts off an open one, or that slide goes on the step you slide from.
- **The people who used it**, the Keyboard Magazine article of July 2000 and the 320 mods of the
  Computer Controlled Underground archive, say how it was actually played: 32-step patterns out of
  two bars, a delay that makes chords, toms tuned to play melodies. That knowledge exists nowhere
  in the program.

All three are copyrighted and stay out of git. What is written here is in our own words, with the
page or the file it comes from, so a reader can check it.

Where they disagree, the manual and the article win on intent and the code wins on behaviour: this
project matches what the original *did*, which the manual describes, the community explored, and
the disassembly proves.

To hear the article's example songs and the archive's mods, keep them outside the repository:

```sh
./target/release/orb render "chords01.rbs" -o /tmp/chords.wav      # the polyphonic 303
python3 tools/rbs.py dump "Plucked 303-01.rbs" | head -20          # its knobs, to learn from
```

