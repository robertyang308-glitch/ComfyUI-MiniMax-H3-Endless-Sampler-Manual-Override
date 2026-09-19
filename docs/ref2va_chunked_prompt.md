---
name: ref2va-chunked-prompt
description: Write a MiniMax H3 Ref2VA prompt packaged for the Endless Sampler Advanced node's full_prompt input - six-section base prompt plus per-chunk description blocks. Use for full-reference (Ref2VA) work only, at chunk sizes up to 226 frames.
---

# Ref2VA Chunked Prompt

Produces one document containing a full-reference H3 prompt **and** the
per-chunk description blocks that replace its `detailed_description`
during a chunked render.

Ref2VA only. T2VA, I2VA, FL2VA and L2VA are out of scope, as is
`integrated_multimodal_description`.

---

## 1. Output document

Four parts, in this order:

```
# chunk_frames = <spans, comma separated>
# context_keyframes = 5
# total duration (frames) = <total>
# chunks = <count>
subject_definitions:
...
summary:
...
retention_analysis:
...
detailed_description:
...
overall_soundscape:
...
non_diegetic_music:
...
# ---- chunks ----
<block for chunk 1>
---
<block for chunk 2>
---
<block for chunk 3>
```

Lines beginning with `#` are stripped before the prompt reaches the model,
so headers are safe. Blocks are separated by a line of exactly three
dashes. **Never emit YAML frontmatter** — its closing `---` would be read
as a block separator.

Output the finished document and nothing else. No preamble, no code
fences, no commentary. Keep field names, section order and timing
notation verbatim; H3 only parses them exactly as written.

---

## 2. Frame arithmetic

H3 accepts only frame counts of the form **17k + 5**. Every span and the
total must come from this list:

| span | length | delivers (chunks 2+) |
|---:|---:|---:|
| 22 | 0.92s | 17 |
| 39 | 1.63s | 34 |
| 56 | 2.33s | 51 |
| 73 | 3.04s | 68 |
| 90 | 3.75s | 85 |
| 107 | 4.46s | 102 |
| 124 | 5.17s | 119 |
| 141 | 5.88s | 136 |
| 158 | 6.58s | 153 |
| 175 | 7.29s | 170 |
| 192 | 8.00s | 187 |
| 209 | 8.71s | 204 |
| 226 | 9.42s | 221 |

**226 is the ceiling. Never exceed it.**

The first chunk delivers its full span; every later chunk delivers its
span minus 5, because five frames overlap the previous chunk and are
trimmed. So:

```
total = span_1 + (span_2 - 5) + (span_3 - 5) + ...
```

That total must itself be on the 17k + 5 list. An off-grid total is
silently rounded down and the block count stops matching.

### Choosing spans

Vary them. A chunk boundary is free where a cut falls and costly in the
middle of continuous action.

- **Cut-heavy or fast action:** 56, 73, 90. Short spans let a boundary
  land exactly on each cut.
- **Dialogue:** the smallest span holding the whole exchange. A line
  crossing a boundary must be repeated verbatim in both blocks and risks
  a seam mid-word.
- **Long takes, slow or static shots:** 192, 209, 226. Fewer chunks means
  less overlap re-sampled.

---

## 3. Base prompt: the six sections

In this order, with these exact field names:
`subject_definitions`, `summary`, `retention_analysis`,
`detailed_description`, `overall_soundscape`, `non_diegetic_music`.

Only `detailed_description` is replaced per chunk. **Every other section is
sent to the model unchanged with every single chunk.**

### 3.0 Keep changing detail out of the static sections

This is the most common way a chunked Ref2VA prompt goes wrong.

Because `subject_definitions`, `summary` and `retention_analysis` are
repeated verbatim for every chunk, anything written there is asserted as
true of *all* of them. A detail that only becomes true partway through the
video contradicts every earlier chunk, and the model has to reconcile a
description that does not match what it is being asked to render.

**Rule: if it changes across the video, it belongs in a chunk block, not
in the static sections.**

Say the scene is `<Subject 1>` getting dressed: a robe at first, then
underwear, then a blue dress, then a coat.

Wrong — the dress is asserted from frame one:

```text
<Subject 1> is the young woman in <Picture 1>, with long dark hair, wearing a blue A-line dress and a camel wool coat.
```

Right — define only what holds throughout, and put each garment in the
chunk where it is actually worn:

```text
subject_definitions:
<Subject 1> is the young woman in <Picture 1>, with long dark hair, fair skin, and a slim build.
```

```text
# ---- chunks ----
<Subject 1> stands before the mirror in a loose grey robe, fingers at the tie...
---
...she steps into the blue A-line dress and draws it up over her shoulders...
---
...she shrugs on the camel wool coat and turns to check the fall of the hem...
```

**What belongs where:**

| Static sections | Chunk blocks |
|---|---|
| Identity: face, build, hair, permanent features | Clothing that is put on, removed or changed |
| A location's fixed architecture and fittings | Lighting that shifts, doors that open, mess that accumulates |
| A label's reference role and what it denotes | Position, pose, expression, action at a given moment |
| The task type, and at most the subjects and setting | Everything that happens, in the order it happens |
| Relationships between references | Props picked up, put down or transformed |
| Style and register of the whole piece | Weather, time of day, anything mid-transition |

The same applies to `overall_soundscape` and `non_diegetic_music`, which
are also repeated every chunk: describe only sound present throughout.
Music that starts partway, or a noise that stops, belongs in the chunk
block where it happens.

If a garment, prop or location detail genuinely persists from the first
frame to the last, it is static and can be defined. The test is whether it
is true of every chunk, not whether it is important.

### 3.1 `subject_definitions`

Four label types:

| Label | Meaning |
|---|---|
| `<Subject N>` | Visible content reused or modified in the target video |
| `<Picture N>` | A reference image used as a concrete frame or composition anchor |
| `<Video N>` | A reference video providing an edit source, continuation point, or temporal structure |
| `<Audio N>` | An audio signal copied or referenced |

One line per item that must be tracked separately. State what the label
denotes, its reference role, and the features to follow — **restricted to
what holds for the entire video**, per 3.0. Define the person, not the
outfit they change into.

`<Subject N>` covers people, animals, objects, scenes, environments,
clothing, props, styles, actions and poses. One subject may draw on
several assets; one asset may supply several subjects.

```text
<Subject 1> is the young woman in <Picture 1>, with long dark hair, a blue cardigan, and a thin silver necklace.
<Subject 2> is the woman whose appearance comes from <Picture 1> and whose walking motion comes from <Video 1>.
```

Use a standalone `<Picture N>` only when the image itself is a frame
anchor. If it merely defines a character, scene or style, cite it inside
that `<Subject N>` definition instead.

`<Video N>` is for whole-video relationships — editing, continuation, or
referencing camera movement, cuts and rhythm. Visible content taken from
a reference video still belongs under `<Subject N>`.

`<Audio N>` is a standalone audio asset or an enabled synchronized track.
When it corresponds to a target speaker, reuse that speaker's global ID:

```text
<Audio 1> is the voice-timbre reference for <Subject 1> (S1).
```

`<Video N>` and `<Audio N>` are numbered independently; the same source
file may be `<Video 1>` and `<Audio 2>`. A reference video does not create
an `<Audio N>` merely because it contains sound.

### 3.2 `summary`

Keep this as short as the task allows. It is repeated with every chunk, so
any narrative in it fights whichever chunk is not at that point in the
story.

**No action, no sequence, no outcome.** Do not describe what happens, what
anyone does, what changes, or how the piece ends. State the task type and,
only if it is genuinely needed, what the target video is.

Often the bracketed prefix alone is enough:

```text
summary:
[reference generation]
```

```text
summary:
[reference generation + audio reference] The target video shows <Subject 1> in <Subject 2>.
```

Wrong — this narrates, so every chunk is told about events it is not
rendering:

```text
[reference generation] <Subject 1> stands before the mirror in a robe, then dresses, and finally puts on a coat before leaving.
```

The task-type prefix:

| Task type | When |
|---|---|
| `keyframe completion` | An image is a concrete frame anchor |
| `reference generation` | An asset guides a character, scene, style, action or camera without being a frame anchor or edit source |
| `video editing` | An existing source video is directly modified |
| `video continuation` | New content continues or extends an existing video |
| `audio reuse` | The same audio signal is reused |
| `audio reference` | Only style, timbre, content, texture, beat or continuity is referenced |

Combine with ` + `, no repeats. Use only previously defined labels; do not
introduce new ones here.

If a sentence follows the prefix, it names the subjects and the setting
and stops. The shot flow, the action and the ending all live in the chunk
blocks. For editing tasks, begin with
`The target video is an edited version of <Video 1>.`

### 3.3 `retention_analysis`

One line per label, preserving the meaning set in `subject_definitions`.

Visible content uses `fully_preserved`, `partially_preserved`,
`attribute_transfer`, or `weak_reference`:

```text
<Subject 1> (appears in [Shot 1], [Shot 3]): fully_preserved - ...
<Picture 2> ([Shot 1] first frame): fully_preserved - ...
<Video 1> (cut and pacing structure): weak_reference - ...
```

Audio uses `fully_copy`, `partially_copy`, `reference`, or
`weak_reference`:

```text
<Audio 1>: fully_copy - reused 1:1 as the complete final audio track.
```

Never write `(Sx)` in this section. Newly added actions or backgrounds in
the target video are not losses of reference fidelity.

State the retention relationship, not the moment-to-moment appearance. A
subject whose clothing changes is still `fully_preserved` if its identity
is retained; do not list the garments here.

### 3.4 `detailed_description`

Write this **in full, on the global timeline**, exactly as for an
unchunked Ref2VA prompt. The sampler replaces it per chunk, but it is the
source the blocks are derived from and its shot markers define where the
real cuts fall — which is what chunk boundaries should follow.

- English body; preserve the original language of dialogue, lyrics and
  visible text.
- One or two sentences establishing style **before** `[Shot 1]`.
- `[Shot 1]` opens with no timestamp. Later shots use
  `[Shot N] At MM:SS.mmm, ...` on the **global** clock.
- Camera movement as natural English inside the shot.
- Stable speaker IDs `(S1)`, `(S2)`, assigned by order of actual vocal
  events. Dialogue as `<d>[Language] ...</d>`.
- Insert `<Subject N>`, `<Picture N>`, `<Video N>`, `<Audio N>` at first
  appearance and wherever their role applies. Describe a subject's
  referenced characteristics, frame position and current action at its
  first clear appearance; reuse the label later without redefining it.
- Frame anchors read naturally: `the shot begins from <Picture 1>`,
  `the shot ends on <Picture 3>`.
- When a referenced subject speaks, keep both labels: `<Subject 2> (S1)`.
  Off-screen speech keeps the same form, marked `off-screen`. A speaker
  with no defined subject gets a stable voice description plus `(Sx)`.
- Verbal content that exists only inside a reused soundtrack cites
  `<Audio N>` and gets no `(Sx)`.
- Reused dialogue keeps exact source words and language. Write
  `[unclear]` rather than guessing. Standard punctuation only.
- Use `<scenetrans>` and `<cutoff>` for dialogue crossing a cut or
  truncated by the ending.

Normally 350-500 English words for generation tasks; dialogue-dense
content prioritises fitting the spoken timeline over word count.

### 3.5 `overall_soundscape` and `non_diegetic_music`

`overall_soundscape` summarises ambience and physical sound across the
whole video. `non_diegetic_music` describes audience-only score, with
instrumentation, tempo and dynamics, or `N/A`.

Shot-synchronised sound events and all dialogue stay in
`detailed_description`. Never repeat dialogue or lyrics here.

When reference audio is used, state the copy or reference relationship in
whichever section matches the audible layer.

---

## 4. Chunk blocks

One block per chunk, after the boundary, separated by `---`.

Each block replaces `detailed_description` for that chunk. **Write
description prose only.** Never include `subject_definitions:`,
`summary:`, `retention_analysis:`, `overall_soundscape:` or
`non_diegetic_music:` inside a block — they come from the base prompt and
are sent with every chunk.

**Each chunk restarts its clock at 0:00.000.** This is the single biggest
difference from the base prompt. A global timestamp inside a block points
outside the chunk being rendered. A chunk's local range ends at
(span − 1) / 24 seconds.

Shot markers, on the local clock:

- Chunk begins mid-shot: open with plain prose, **no marker**.
- Chunk's first frame is a real cut: `[Shot 1]`.
- A real cut later in the chunk: `[Shot N] At M:SS.mmm,` numbering from 2
  within that chunk only.

Never add, move or invent cuts. Prefer shortening a span so a cut lands on
the boundary instead of inside a chunk.

Chunks after the first open on 5 frames already rendered. Continue that
motion: never restart an action whose result is already visible, never
re-establish a subject, never reset a camera move mid-travel.

All label and speaker rules from 3.4 apply unchanged inside blocks. A
dialogue line overlapping a boundary appears complete and verbatim in
every block it overlaps.

---

## 5. Complete example

```text
# chunk_frames = 141, 141
# context_keyframes = 5
# total duration (frames) = 277
# chunks = 2
subject_definitions:
<Subject 1> is the coffee-shop environment in <Picture 1>, featuring an exposed brick wall, an orange tufted sofa with patterned pillows, a neon sign, and a wooden coffee table.
<Subject 2> is the fluffy white Samoyed in <Picture 2>, with thick white fur, pointed ears, a dark nose, and a curved tail.
<Subject 3> is the young blonde woman in <Picture 3>, with long blonde hair and a light-pink button-down shirt with rolled-up sleeves.
<Audio 1> is the voice-timbre reference for <Subject 3> (S1), containing a spoken English vocal layer.

summary:
[reference generation + audio reference] The target video shows <Subject 3> and <Subject 2> in <Subject 1>, with <Audio 1> as the voice-timbre reference for <Subject 3>.

retention_analysis:
<Subject 1> (appears in [Shot 1], [Shot 2], [Shot 3]): fully_preserved - the exposed brick wall, orange tufted sofa, patterned pillows, neon sign, and wooden coffee table are retained.
<Subject 2> (appears in [Shot 1], [Shot 3]): fully_preserved - the Samoyed's thick white fur, pointed ears, dark nose, and curved tail are retained.
<Subject 3> (appears in [Shot 1], [Shot 2], [Shot 3]): fully_preserved - the blonde woman's identity, long hair, and light-pink shirt are retained.
<Audio 1>: reference - its vocal timbre guides the dialogue delivery of <Subject 3> without copying the original signal.

detailed_description:
The target video uses a realistic multi-camera sitcom style with warm indoor lighting.
[Shot 1] A medium shot establishes <Subject 1>, the coffee shop with its exposed brick wall and orange tufted sofa. <Subject 3> (S1) sits holding a chocolate-chip cookie. <Subject 2>, the white Samoyed, lunges toward it and pulls its leash taut. <Subject 3> (S1) jerks her hand back and, in the clear youthful timbre referenced from <Audio 1>, exclaims, <d>[English] Hey! Watch your dog!</d>
[Shot 2] At 00:03.000, the shot cuts to a close-up of <Subject 3> (S1) guarding the cookie against her chest, her annoyance softening as she glances off-frame.
[Shot 3] At 00:05.875, the shot cuts to a wider two-shot of <Subject 1> with <Subject 2> settled at her feet. <Subject 3> (S1) replies with an amused cadence, <d>[English] Well, he has good taste at least.</d> A canned audience laugh begins immediately after the line and continues to the final frame.

overall_soundscape:
Soft indoor coffee-shop room tone continues throughout the scene.

non_diegetic_music:
N/A
# ---- chunks ----
The target video uses a realistic multi-camera sitcom style with warm indoor lighting.
[Shot 1] A medium shot establishes <Subject 1>, the coffee shop with its exposed brick wall, orange tufted sofa, patterned pillows, neon sign, and wooden coffee table. <Subject 3> (S1), the young woman with long blonde hair and a light-pink button-down shirt, sits on the sofa holding a chocolate-chip cookie. <Subject 2>, the thick-furred white Samoyed with pointed ears and a dark nose, lunges toward the cookie and pulls its leash taut. <Subject 3> (S1) jerks her hand back and, in the clear youthful timbre referenced from <Audio 1>, exclaims with light annoyance, <d>[English] Hey! Watch your dog!</d> [Shot 2] At 0:03.000, the shot cuts to a close-up of <Subject 3> (S1) guarding the cookie against her chest, her annoyance softening as she glances off-frame toward the dog.
---
[Shot 1] A wider two-shot of <Subject 1> shows <Subject 3> (S1) on the orange sofa with <Subject 2> now settled at her feet, its curved tail resting against the wooden coffee table. She lowers the cookie and looks down at the Samoyed with an amused expression. <Subject 3> (S1) replies in the same clear youthful timbre referenced from <Audio 1>, <d>[English] Well, he has good taste at least.</d> She raises the cookie in a small toast-like gesture. A classic canned audience laugh begins immediately after the line and continues through the final frame.
```

Note in the example:

- The global cut at `00:05.875` is frame 141, exactly the chunk boundary.
  That is why chunk 2's block opens with `[Shot 1]` — a real cut on its
  first frame — and why spans of 141 were chosen.
- The cut at `00:03.000` falls inside chunk 1, so it appears there as
  `[Shot 2] At 0:03.000` on the local clock, which happens to coincide
  with the global one because chunk 1 starts at zero.
- Chunk 2 re-describes `<Subject 1>` and `<Subject 3>` because a new shot
  begins there. A block continuing mid-shot would not.

---

## 6. Self-check

1. Every span is on the 17k + 5 list and **none exceeds 226**.
2. `# total duration (frames)` is on the 17k + 5 list.
3. `span_1 + sum(span_i − 5)` equals that total.
4. `# chunks` equals the number of blocks.
5. Spans vary with the material; cuts land on boundaries where possible;
   no spoken line is split across a boundary.
6. Base prompt has all six sections, in order, with exact field names.
7. `detailed_description` is written in full on the global timeline.
8. Exactly one boundary line, reading `# ---- chunks ----`.
9. Blocks separated by exactly three dashes on their own line.
10. No field labels inside any block.
11. Every timestamp inside a block is local and within its own span.
12. Every `<Subject N>`, `<Picture N>`, `<Video N>`, `<Audio N>` used is
    defined, and `(Sx)` IDs are consistent throughout.
12a. `summary` contains no action, sequence or outcome - task-type prefix,
    and at most one sentence naming subjects and setting.
12b. **Nothing in `subject_definitions`, `summary`, `retention_analysis`,
    `overall_soundscape` or `non_diegetic_music` describes something that
    changes during the video.** Read each line and ask whether it is true
    of chunk 1 and of the last chunk. If not, move it into a block.
13. No YAML frontmatter in the output.
