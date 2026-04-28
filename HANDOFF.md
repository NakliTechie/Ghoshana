# Ghoshana — handoff to next session

> **Ghoshana** (घोषणा, "announcement") — browser-native version of the
> [`prashnam-voice`](https://github.com/prashnam/prashnam-voice) Python desktop
> tool. Same UX, same features, no install, no server.

This document is the bootstrap for a fresh Claude Code session. Open it in
**`/Users/chiragpatnaik/Code/Browser/Ghoshana/`** as the working directory —
all code goes here. Reference projects live one level up at
`/Users/chiragpatnaik/Code/Browser/`.

---

## Status of preceding work

The `prashnam-voice` browser-prep was completed in the prior session:

- ONNX export of IndicTrans2 200M (en→indic) — bit-exact vs PyTorch on 528 fixtures.
  Published at [`naklitechie/indictrans2-en-indic-dist-200M-ONNX`](https://huggingface.co/naklitechie/indictrans2-en-indic-dist-200M-ONNX).
- Fast tokenizer JSON files for the dual-vocab IT2 setup (also published in the same repo).
- Vanilla-JS port of `IndicTransToolkit.IndicProcessor` — 100% parity vs Python on 528 fixtures.
- A translation-only single-file demo at [`Browser/Anuvaad/`](../Anuvaad/), live at
  [naklitechie.github.io/Anuvaad](https://naklitechie.github.io/Anuvaad/), with
  IndexedDB-backed model caching so return visits skip the 1.3 GB download.

The next ambition is **full feature parity with the Python desktop tool**: project
management, three project shapes (poll / announcement / IVR with DAG editor),
take history with rotations, lexicon, voice/pace overrides, and **on-device audio
generation** via Indic Parler-TTS.

---

## Confirmed scope (decided in the prior session)

| Question | Decision |
|---|---|
| Which prashnam-voice features to port? | **All of them.** Goal is feature parity with the desktop tool. |
| Audio engine? | **Indic Parler-TTS** is the target. Tiered rollout — see "Audio strategy" below. |
| Persistence? | **File System Access API** as primary, anchored on Slate's pattern. IndexedDB as scratch + as the model-cache backbone. No size caps. |
| Repo / name? | **Ghoshana**, separate from Anuvaad. New GitHub repo (`NakliTechie/Ghoshana`) when ready. |

---

## Recon already done — three reports in conversation history of this session

The prior session ran three Explore/general-purpose agents in parallel to scope the
port. **You should re-read those reports first thing in the next session** — they
contain everything needed to plan code without re-recon. Search the prior transcript
for these report headers:

1. **"PRASHNAM-VOICE BROWSER PORT SCOPING REPORT"** — full feature surface, FastAPI
   endpoints, dataclass quotes, file:line citations across `prashnam_voice/`. Walks
   layered roadmap: Layer 0 (FS + persistence + routing) → Layer 1 (translation-only
   v0) → Layer 2 (TTS) → Layer 3 (rotations / IVR / takes / export).

2. **"Slate File System Access API Pattern — Engineering Report"** — the 5
   load-bearing FSA functions to copy verbatim from `Browser/Slate/`. Permission
   flow with `queryPermission` / `requestPermission` escalation, IDB-backed handle
   serialization, feature-detect cascade. ~300 LOC to lift directly.

3. **"Indic Parler-TTS in the browser — feasibility report"** — hard-headed
   feasibility: 3.75 GB fp32 / ~1.9 GB fp16, ~430 autoregressive decoder steps for
   5s of audio, no public ONNX export exists (yet), realistic browser TTFA is
   6-15s on WebGPU desktop / 40-120s on WASM / mobile is dead. Full Parler-TTS port
   = 4-8 weeks with real risk of dead-end on the dual-encoder export.

---

## Recommended audio strategy (decision required up front in next session)

The user's stated preference is **Indic Parler-TTS** specifically. The realistic
options:

- **(A) Full Parler-TTS in browser.** ~4-8 weeks. Desktop-only TTFA is 5-15s. No
  public ONNX export exists — first risk gate. The dual-encoder Indic variant adds
  export complexity beyond the standard Parler-TTS-Mini.
- **(C) Piper VITS bundle.** 4-6 priority Indic languages (Hin/Tam/Tel/Ben), ~1.5
  weeks, instant-ish TTFA, significantly lower fidelity than Parler — but ships.
- **(E) Tiered.** Web Speech API as the always-instant default (built-in on every
  browser), Indic Parler-TTS as a "high-quality preview" toggle that downloads
  on-demand for desktop+WebGPU users. v0 ships in days, parity work continues
  behind the toggle.

**The prior session's recommendation: (E).** Confirm with the user at the start
of the next session before committing 4-8 weeks to (A).

---

## Layered port plan

### Layer 0 — Framework + persistence (Week 1)

**Goal**: working empty shell with FS Access wired up.

- Lift Slate's FSA pattern (`fs.js`, `session.js` from `Browser/Slate/`):
  - `openFromHandle(handle)` with `queryPermission`/`requestPermission` escalation
  - `saveSession()` / `loadSession()` (IDB single-row session restore)
  - Feature detect (`hasFSA`, `hasDirPicker`) with `<input type="file">` fallback
  - On-disk shape: user picks a top-level "Ghoshana root" folder; under it,
    `<project_id>/project.json` + `<project_id>/audio/<seg_id>/<lang>/<rotation_id>/<attempt_id>.{mp3,json}`
- Hash-based router: `#/` (project list) and `#/p/<id>` (editor)
- Project list / create flow — wireframed but no segments yet
- TS data-model port from `prashnam_voice/projects.py:98-230` (Project, Segment dataclasses)

### Layer 1 — Translation v0 (Week 2)

**Goal**: usable poll editor with English → Indic translation in the browser.

- Lift the working ORT-Web pipeline from `Browser/Anuvaad/index.html` (specifically
  `loadModel()`, `greedyTranslate()`, the IDB caching, the inlined IndicProcessor
  + IT2Tokenizer) — already validated, 100% parity with PyTorch.
- Add poll-domain segment editor (question + N options, with template wrapping
  `prashnam_voice/projects.py:350-407`).
- Lexicon UI — global + per-language, applied pre-translate via the ported
  `_apply_lexicon` (already in `browser-prep/js/lexicon.js`).
- Numeral normalization (port `prashnam_voice/text_normalize.py:38-56`).
- Per-segment translation across all selected target languages, with
  background queue + UI progress.

### Layer 2 — Audio (Weeks 3-4)

**Goal**: per-segment audio playback per language.

- **(E)** path first: `speechSynthesis.speak()` with per-language voice resolution.
  Hours of work. Ships v0 with audio.
- Voice/pace UI hierarchy: project default → per-segment override (per
  `prashnam_voice/config.py:27-77`).
- Take history + `current_takes` pointer (per
  `prashnam_voice/projects.py:145-147`); UI to switch between takes.
- Audio stored under FS root.

### Layer 3 — Rotations + IVR + take browsing + export (Weeks 5-6)

- Rotations: deterministic seeded shuffle of non-locked options
  (`prashnam_voice/projects.py:305-347`).
- Announcement domain (untemplated body segments).
- IVR DAG editor — port the canvas SVG node+edge layout from
  `prashnam_voice/server/static/app.js:1751-1900`. Walk simulator with DTMF input.
- ZIP export via JSZip (per
  `prashnam_voice/server/app.py:795-806`).

### Layer 4 — Indic Parler-TTS toggle (separate track, weeks)

- ONNX export of Parler-TTS-Mini-v1.1 first (single-encoder, sanity check).
- ONNX export of Indic Parler-TTS dual-encoder variant (real risk).
- DAC codec is already ported at `onnx-community/dac_44khz-ONNX` — free.
- Reimplement `apply_delay_pattern_mask` autoregressive loop in JS (precedent:
  Xenova's Whisper / OuteTTS wrappers).
- Behind a "high-quality preview" toggle, download on-demand.

---

## Key reference paths

```
prashnam-voice (Python source — read-only reference):
  /Users/chiragpatnaik/Code/prashnam-voice/
    prashnam_voice/projects.py        — Project, Segment, ProjectStore
    prashnam_voice/server/app.py      — All HTTP endpoints
    prashnam_voice/server/static/app.js  — 2,246-line vanilla JS frontend (UI to mirror)
    prashnam_voice/translator.py      — IT2 wrapper (logic to port)
    prashnam_voice/tts.py             — Indic Parler-TTS wrapper
    prashnam_voice/pipeline.py        — synthesize_segment_lang, translate_segments
    prashnam_voice/text_normalize.py  — numeral normalization
    prashnam_voice/config.py          — language registry, voices, paces
    prashnam_voice/domains.py         — poll / announcement / IVR domain abstraction
    prashnam_voice/adapters/          — local + Sarvam adapter pluggability

Slate (FS Access API reference):
  /Users/chiragpatnaik/Code/Browser/Slate/
    fs.js                             — openFromHandle, save/load helpers
    session.js                        — IDB session restore
    app.js                            — feature detect + boot

Anuvaad (working translation pipeline — drop-in):
  /Users/chiragpatnaik/Code/Browser/Anuvaad/
    index.html                        — full single-file working app, IDB-cached
                                          (this is the encoder.run + decoder.run loop
                                          ported from 04_parity_test.py — copy this
                                          chunk into Ghoshana's translation layer)
  Live: https://naklitechie.github.io/Anuvaad/

JS port libraries (already validated 100% parity vs Python):
  /Users/chiragpatnaik/Code/prashnam-voice/browser-prep/js/
    indic_processor.js                — preprocess + postprocess + transliterate
    indic_processor.test.js           — 528/528 fixture parity test
    tokenizer.js                      — BPE encoder/decoder for tokenizer.json
    tokenizer.test.js                 — 4/4 parity smoke test
    lexicon.js                        — whole-word substitution

Ground-truth fixtures:
  /Users/chiragpatnaik/Code/prashnam-voice/browser-prep/fixtures/
    test_sentences.json               — 48 EN inputs × 4 categories
    expected_outputs.json             — 528 PyTorch greedy outputs (raw, Devanagari-normalized)
    expected_processed.json           — 528 IndicProcessor-postprocessed (native script)
    parity_report.json / parity_report_int8.json
```

---

## Memory entries from prior sessions (carry forward)

- **Quality > size for browser bundles** — up to 5 GB fine; do not quantize at
  quality cost without asking. Recorded 2026-04-28 after the int8 quality drift
  was rejected as not-shippable.
- **GitHub repo path is `prashnam/prashnam-voice`**, NOT `chiragpatnaik/...`
  (the local clone is at `/Users/chiragpatnaik/Code/prashnam-voice` because of
  the local user account, but the public org is `prashnam`).

---

## Suggested first-message handoff prompt (paste into the next session)

```
I'm continuing the prashnam-voice browser port. Read HANDOFF.md in this
directory first — it has the full context, layered roadmap, and links to
the prior session's recon reports.

Specifically: I want to start Ghoshana, a browser-native full feature port
of the Python prashnam-voice tool at /Users/chiragpatnaik/Code/prashnam-voice.

The translation pipeline is already validated and lives at
/Users/chiragpatnaik/Code/Browser/Anuvaad/index.html (live at
naklitechie.github.io/Anuvaad). The working ORT-Web greedy decode loop
+ IDB caching + IndicProcessor + tokenizer JS port can be lifted directly.

Audio strategy: prior session's recommendation is tiered — Web Speech as
the always-instant default, Indic Parler-TTS as a download-on-demand
high-quality toggle (since full Parler-TTS in browser is a 4-8 week R&D
bet with 5-15s TTFA on WebGPU desktop and unusable mobile). Confirm
this strategy before any audio code lands.

Start with Layer 0: FS Access shell mirroring Slate's pattern from
/Users/chiragpatnaik/Code/Browser/Slate/, with project list + create
flow and the data model from prashnam_voice/projects.py:98-230.

Don't write any code until you've read HANDOFF.md and proposed the
Layer 0 file structure.
```

---

## Open questions for the user (raise at start of next session)

1. **Confirm audio strategy** — go with **(E) tiered** as recommended, or commit
   to **(A) full Parler-TTS port** despite the latency / risk profile?

2. **GH repo creation timing** — create `NakliTechie/Ghoshana` empty now (so
   incremental commits land), or wait until Layer 0 is complete?

3. **Voice catalog** — do you want to mirror the prashnam-voice voice list per
   language (Divya for Hindi, Jaya for Tamil, etc., per `config.py:27-57`), or
   start with a smaller curated set and expand later?

4. **Sarvam adapter** — keep the Sarvam cloud option as a fallback (requires
   user-supplied API key), or strip it for the "no server, no token" property
   the README leads with?

5. **Multi-tab / multi-window concurrency** — does the user expect to be able
   to open multiple Ghoshana tabs against the same FS root? (FS Access handles
   are per-origin and can be shared, but two tabs writing to the same project
   simultaneously is a hazard.)
