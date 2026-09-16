# agent-mem presentation

English eight-minute hackathon presentation for `agent-mem`.

## Contents

- `presentation/agent-mem-presentation.pdf` is the ready-to-present deck.
- `presentation/agent-mem-presentation.tex` is the Beamer source.
- `presentation/agent-mem-speaker-script.md` contains the timed English script.
- `presentation/assets/` contains the dashboard and Archify images used by the deck.
- `archify/` contains the two standalone diagrams for the live walkthrough.

## Build the PDF

Requires Tectonic or a compatible LaTeX installation.

```bash
cd presentation
tectonic --outdir . agent-mem-presentation.tex
```

## Live walkthrough

Open these standalone files in a browser:

- `archify/agent-mem-storage-dataflow.html`
- `archify/agent-mem-session-injection-sequence.html`

The deck explains the path from a captured source to bounded session context. The dashboard image is a local, read-only UI snapshot without desktop chrome or a cursor.
