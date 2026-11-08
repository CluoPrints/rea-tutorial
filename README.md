# rea-tutorial

Hands-on test & tutorial notes for **REA (Reverse Engineer Anything)** — by [CluoPrints](https://cluoprints.com/).

> This repo holds our notes from actually installing and running REA. It is not affiliated with REA or its author. Everything below is what we observed in our own test — nothing more, nothing less.

## Full tutorial

- English: <https://cluoprints.com/rea-tutorial.html>
- Version française : <https://cluoprints.com/rea-tutorial-fr.html>

## What REA is

REA stands for **Reverse Engineer Anything**. It is a free, open-source tool (MIT licence) published by GitHub user `morluto` ([morluto/rea](https://github.com/morluto/rea)). Technically, it is an **MCP server plus a command-line tool** that connects professional reverse-engineering tools — Ghidra, IDA, Hopper, JADX and others — to AI agents, so an agent can inspect a program and explain what it does.

REA does **not** recover original source code and does **not** clone apps. Its README is explicit: your agent uses REA's findings to explain behaviour, or to write and test a new implementation.

Official sources only: the GitHub repo `morluto/rea`, the website <https://rea.tools>, and the npm package `rea-agents`. A similar repository name or a copied README does not make a download official.

## What we tested

We installed **REA 6.8.0** (`rea-agents@6.8.0`, verified on npm with maintainer `morluto`). Requirements from the official README: Node.js 22.x (≥ 22.23.4), 24.x (≥ 24.22.1) or 26.12.1+, plus npm. The install took about 40 seconds, and `rea doctor` confirmed Node was healthy but reported that no native analysis engines (Hopper, Ghidra, IDA) were installed — static JavaScript and .NET inspection work without a native engine; deep native-binary analysis needs one.

We tested REA **only against our own code**: our free Social Image Sizes tool on this site, which turns one uploaded image into eight social-media sizes. We asked two questions:

1. How does this tool generate and export the different image sizes?
2. What happens to an uploaded image — is it uploaded to a server, or processed locally?

## What we found

| | Result |
| --- | --- |
| Install | ~40 seconds, version 6.8.0 confirmed |
| Analysis time | ~5 seconds |
| Code analysed | 4 files, 15,835 bytes, 2,869 AST nodes |
| Evidence report | 2.9 MB of JSON |

**Attempt 1 — the blind spot REA admitted itself.** We ran the documented command (`rea analyze-javascript-application <folder> --json`) on the tool's page plus its two linked scripts. REA fully parsed both external scripts, but did not analyse the tool itself, because the tool's code lives in an inline `<script>` block inside the HTML. To REA's credit, its report said so in plain words: "2 inline JavaScript bodies … that were not analyzed." Lesson: for ordinary web pages, extract the scripts first (or use REA's browser-capture workflow).

**Attempt 2 — scripts extracted, full run.** We copied the page's two inline scripts, byte for byte, into separate `.js` files and re-ran the same command. This time REA parsed everything (4 files, zero parse errors). What it reported, checked against our source:

- ✅ **Correct:** `drawPreset()` creates a canvas, gets its 2D context, draws the image with `drawImage`, encodes it with `canvas.toBlob`, and hands the result out via `URL.createObjectURL`; a `PRESETS` table drives the sizes. REA placed `drawPreset()` at byte offset 2921 of the script — exact to the character.
- ✅ **Correct:** background-fill calls (`fillStyle` / `fillRect`) for the "fit whole image" mode, the exact warning text "Upscaled from a smaller original — may look soft.", and a `setInterval` timer of 400 ms (a real quirk in our code: a polling fallback for Android browsers).
- ✅ **Correct:** zero network calls across all four files — no `fetch`, no `XMLHttpRequest`, no WebSocket. The only storage found was the cookie-consent flag in a different file (`app.js`, line 63 — also exact). That answers Q2: the image is loaded with an object URL and processed entirely in the browser; it is never uploaded.
- ⚠️ **Missed:** the eight preset dimensions (1080×1080, 1080×1350, 1080×1920, …) and the crop-vs-fit scaling formula (`Math.max` / `Math.min`). REA identified the `PRESETS` table and the drawing calls, but did not surface the table's values or the fit math as structured findings.

**Our verdict: as a structural map, REA is precise** — mechanisms, call chains and source locations were right every time we checked. **But it shows *what* the code does and *where*, not every data table and formula inside it.** Coverage is partial: even our 16 KB test target left hundreds of unresolved points, which REA labels "unknown" rather than guessing.

Nothing REA stated was false — its citations were exact. Its weaknesses were omissions.

## What REA can't do (despite the hype)

- It does not recover original source code, and it does not clone apps.
- Coverage is partial. Inline page scripts were skipped on the first run, and hundreds of points stayed unresolved even after extraction.
- No engine, no deep native analysis. Hopper, Ghidra, IDA or Kuna must already be installed, under your own licence. REA grants no tool licences — IDA Pro, Hopper and JEB remain commercial products under their own terms.
- Static JavaScript analysis doesn't run the app. REA reads the files; behaviour that only appears at runtime can be missed unless you use its capture workflow.
- It analyses targets locally (per the official FAQ); your agent receives the tool results.

## Who this is for

- Developers and researchers who want to understand **their own** code (or code they are authorised to examine) with an AI agent's help.
- Anyone curious about what static analysis can — and can't — tell you about a program.
- **Not** for inspecting other people's commercial software: only analyse software you own or are authorised to examine. Third-party commercial software is a legal grey zone, and copyright and licence terms still apply. We don't offer reverse-engineering services, and nothing here is a how-to for inspecting other people's apps.

## Try the tool REA audited

The page REA analysed is one of our free browser tools — it resizes one image into every social-media size. Find it on [cluoprints.com](https://cluoprints.com/).
