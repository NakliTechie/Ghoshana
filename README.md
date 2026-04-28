# 📢 Ghoshana

**Browser-native project authoring for the [prashnam-voice](https://github.com/prashnam/prashnam-voice) family.** No server, no install, no API keys. Your projects live in a folder you pick on your own disk.

> *Ghoshana* (घोषणा, "announcement") is the browser version of the Python desktop tool that records multilingual polls, announcements, and IVR flows for the [Prashnam](https://prashnam.com/) language-data project.

## Status

**Layer 0** — framework and persistence shell. Pick a folder, create projects with name/domain/languages, and they save to disk as JSON. No segments, translation, or audio yet.

Roadmap:

- ✅ **Layer 0** — file-system persistence + project list + create flow.
- ⏳ **Layer 1** — translation pipeline (lifted from [Anuvaad](https://github.com/NakliTechie/Anuvaad)) + poll segment editor.
- ⏳ **Layer 2** — on-device audio (Web Speech default + take history).
- ⏳ **Layer 3** — option rotations, announcement domain, IVR DAG editor, ZIP export.
- ⏳ **Layer 4** — Indic Parler-TTS download-on-demand toggle for high-quality voices.

## How to run

```bash
python3 -m http.server 8080
# open http://localhost:8080/
```

No build step. The whole app is one HTML file. Tested in Chrome, Edge, and Firefox; iOS / Android Safari are not supported (the File System Access API is desktop-only).

## What it stores, where

When you pick a folder via "Pick a folder", projects live as plain JSON files inside it:

```
<your-folder>/
  asking-mood-a1b2c3/
    project.json
    audio/...                  # populated in Layer 2
  ...
```

Nothing else is written anywhere. Closing the tab does not lose your work — your folder is the source of truth. The browser also keeps a mirror copy in IndexedDB so you can see your project list before re-granting folder permission on the next visit.

## Privacy

Everything runs in your browser. The only network requests are:

- Fetching this `index.html` from GitHub Pages (or from `localhost` when self-hosted).
- (Future Layer 1) Fetching the IndicTrans2 ONNX model bundle from Hugging Face on first translation, then caching it in IndexedDB.

No telemetry. No accounts. No third-party servers.

## Tech

- **No framework, no bundler.** Single HTML file with inlined CSS + JS.
- **File System Access API** for "your folder, your data" persistence (pattern adapted from sibling [Slate](https://github.com/NakliTechie/Slate)).
- **IndexedDB** for the folder-handle mirror and (Layer 1+) the model cache.
- **Hash router** (`#/`, `#/p/<id>`) for screen state in URLs.

Layer 1 will add:

- **[onnxruntime-web](https://onnxruntime.ai/)** for the in-browser IndicTrans2 model run.
- **[`naklitechie/indictrans2-en-indic-dist-200M-ONNX`](https://huggingface.co/naklitechie/indictrans2-en-indic-dist-200M-ONNX)** — the ONNX bundle (already validated bit-exact vs PyTorch on 528 fixtures).
- **JS port of `IndicTransToolkit.IndicProcessor`** — script normalization + Moses tokenization (100% parity with Python).

## Licence

MIT. The upstream `prashnam-voice` Python tool is also MIT.

---

Built by [Chirag Patnaik](https://github.com/NakliTechie) as part of the [prashnam](https://github.com/prashnam) language-tools series.
