# Example: one minute in one run

A 59.7-second Ref2VA clip (1433 frames, 864x480, 24 fps, with audio) rendered
in a single Endless Sampler run from nine manually written chunks.

| File | What it is |
|---|---|
| `prompt.txt` | The `full_prompt` document: header, six-section base prompt, nine chunk blocks |
| `refs/picture_1_traveller.png` | `<Picture 1>` |
| `refs/picture_2_conductor.png` | `<Picture 2>` |
| `refs/picture_3_carriage.png` | `<Picture 3>` |
| `refs/image_prompts.txt` | Text-to-image prompts used to make the three references |
| `output.mp4` | The rendered result |

## Settings

- Node: **Endless Sampler (Advanced)**, `use_full_prompt` on, `prompt.txt` in `full_prompt`
- Latent: 1433 frames (must match `# total duration (frames)`)
- Reference images connected in the order `<Picture 1>`, `<Picture 2>`, `<Picture 3>`
- Sampling: 3 steps with a 3-step distilled LoRA
- `context_keyframes = 5`, video continuation 22 frames

## What it demonstrates

| Chunk | Span | Delivered frames | Role |
|---|---|---|---|
| 1 | 226 | 0-225 | Long handheld following shot |
| 2 | 226 | 226-446 | Same shot, continued across the boundary with no marker |
| 3 | 175 | 447-616 | Cut, then a three-line dialogue kept inside one chunk |
| 4 | 73 | 617-684 | Cut to a short insert |
| 5 | 73 | 685-752 | Cut, entry into a tunnel |
| 6 | 73 | 753-820 | Cut to a close-up |
| 7 | 226 | 821-1041 | Lighting change and slow pull-back, one continuous shot... |
| 8 | 226 | 1042-1262 | ...carried on across the boundary |
| 9 | 175 | 1263-1432 | Cut to the closing shot |

- **Seamless joins.** Eight chunk boundaries, none visible: no jump, flicker
  or colour shift. Subjects and set stay consistent for the full minute.
- **Variable spans.** Long chunks for continuous camera moves, a mid-length
  chunk sized to hold a whole conversation, three 3-second chunks for rapid
  inserts.
- **Placed cuts.** Measured hard cuts landed at 18.79 s, 25.88 s and
  52.79 s against planned 18.92 s, 26.00 s and 52.92 s: within three frames.

## Writing cuts at a chunk boundary

A cut written as the very first frame of a block (`[Shot 1]` at
`0:00.000`) does not happen: the context frames hand the previous picture
into the new chunk, and the model continues it. Open the block with half a
second of the previous shot, then cut:

```
The two-shot holds for a moment as the conductor turns away. [Shot 3] At 0:00.500, the shot cuts to ...
```

`0:00.500` local lands about 0.29 s after the chunk's first delivered frame
(the first 5 frames, 0.208 s, are the trimmed overlap).

## Run statistics

RTX 5090 (32 GB), 192 GB system RAM, dynamic VRAM loading.

| Metric | Value |
|---|---|
| Sampler wall time | 5 min 45 s for 59.7 s of video (about 5.8x real time) |
| Whole workflow | 457.6 s including model loads and the final decode |
| Average per chunk | 38.4 s |
| H3 sampling, total | 3 min 16 s (21.8 s per chunk) |
| Qwen re-encode, total | 1 min 21 s (9.0 s per chunk) |
| Continuation VAE decode, total | 38.9 s (4.9 s per call) |
| Other overhead | 28.9 s |
| VRAM, average during DiT evaluation | 22.75 GiB (71.5 %) |
| VRAM, peak | 26.64 GiB (83.7 %) |
| PyTorch high-water | 1.53 GiB allocated, 2.53 GiB reserved |
| Process RAM, peak | 43.6 GiB |

Per chunk:

| Span | Chunk time | of which H3 | Frames delivered per second of chunk time |
|---|---|---|---|
| 226 | 31.3 - 48.3 s | 24.6 - 30.3 s | about 4.6 - 7.2 |
| 175 | 35.7 - 39.6 s | 21.5 - 23.2 s | about 4.3 - 4.8 |
| 73 | 31.9 - 32.5 s | 13.8 - 14.0 s | about 2.1 |

Each chunk carries roughly 18 s of fixed cost (Qwen re-encode, continuation
decode, model initialisation) regardless of span. A 73-frame chunk costs
almost as much wall time as a 226-frame one, so reserve short spans for
places that need a cut, and use the longest span the material allows
everywhere else.
