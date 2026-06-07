# BuildStream — Preprint Paper

**Title:** BuildStream: A Reproducible AWS-Native Reference Architecture for Mobile CI/CD with Ephemeral Code-Signing Credentials

**Author:** Santosh Kumar Puppala

This directory contains the LaTeX source and compiled PDF of the BuildStream
preprint, written for submission to [arXiv](https://arxiv.org/). It is a
**preprint and has not been peer reviewed.**

## Contents

| File | Description |
|---|---|
| [main.tex](main.tex) | IEEE conference-format LaTeX source (single file, inline bibliography) |
| [main.pdf](main.pdf) | Compiled 6-page PDF |
| [figures/](figures/) | Evidence figures (Jenkins, Secrets Manager, and S3 screenshots) used in the Evaluation section |

The architecture diagram (Fig. 1) and the credential-exposure-window diagram
(Fig. 2) are drawn inline in TikZ and require no external image files.

## Building locally

The paper compiles with any standard TeX distribution (TeX Live, MacTeX, or
TinyTeX). It uses the `IEEEtran` class plus `tikz`, `listings`, `booktabs`,
`hyperref`, `xcolor`, and `graphicx`.

```bash
pdflatex main.tex
pdflatex main.tex   # second pass resolves cross-references
```

With [TinyTeX](https://yihui.org/tinytex/), install the required packages first:

```bash
tlmgr install ieeetran pgf listings xcolor booktabs hyperref url cite \
  caption float grfext capt-of
```

## Building on Overleaf / arXiv

Upload `main.tex` and the `figures/` directory. Both Overleaf and arXiv compile
the source directly; no local build is required. The bibliography is inline
(`thebibliography`), so no separate `.bib` file or BibTeX pass is needed.

## Scope and honesty note

The Evaluation section reports a real AWS deployment with end-to-end pipeline
runs, but the paper is explicit (Section VIII, *Limitations and Threats to
Validity*) about what is and is not demonstrated — notably that the iOS
production signing path is specified and implemented but not exercised against a
paid Apple Developer certificate, that distribution stages are stubbed, and that
timings are illustrative single runs rather than a statistical benchmark.
