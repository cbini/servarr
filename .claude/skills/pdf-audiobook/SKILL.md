---
name: pdf-audiobook
description: Turn a PDF book (often a scanned, OCR'd one) into a chaptered M4B audiobook for Audiobookshelf, using scripts/pdf2tts, Claude proofreading agents and Abogen (Kokoro TTS) on this Mac. Use when the user wants to convert, narrate or make an audiobook from a PDF.
---

# PDF → audiobook

Runs on this Mac (M2, 8 GB): servarr's 2 vCPU/3 GB VM is too small for TTS. Work in a folder under the scratchpad, named after the book (`WORK` below).

## 1. Clean the text

```bash
cp book.pdf WORK/book.pdf
scripts/pdf2tts WORK/book.pdf WORK/book.txt      # needs uv
grep '<<CHAPTER_MARKER' WORK/book.txt            # check chapters before going on
```
`pdf2tts` strips running headers, page numbers, captions and hyphenation and writes Abogen `<<CHAPTER_MARKER:…>>` lines. If chapters are missing or split wrongly, fix the script (run `scripts/pdf2tts --self-test` after), not the text.

## 2. Proofread with agents

```bash
scripts/proofread split WORK/book.txt WORK/book.pdf WORK
```
Then launch general-purpose agents in parallel, one per 1–2 chapter files (skip tiny front matter, contents and index), each prompted: "Follow the brief in WORK/BRIEF.md. Your files: ch/NN.txt[, ch/MM.txt]." Pass the brief's path, not a placeholder. When all have reported:

```bash
scripts/proofread join WORK "WORK/Title.txt"
```
It reports skipped fixes (wrong `count`: check and fix by hand) and prints the pronunciation overrides for step 4. Tell the user about recurring `script_notes` worth adding to `pdf2tts` and anything left `uncertain`.

## 3. Abogen

Start the web UI in the background with `scripts/abogen` (http://localhost:8808; `ABOGEN_PORT` changes it). It installs Abogen from `tts/uv.lock` on first run, so don't `uv tool install` it, and it handles three Abogen quirks:
- it sets `PYTORCH_ENABLE_MPS_FALLBACK=1`; without it, previews fail with "MPS requested but fallback not enabled".
- it puts an ffmpeg on PATH; Abogen's own download unpacks into `…/ffmpeg/darwin_arm64/` but looks in `darwin/`, so jobs fail with `FileNotFoundError: …/ffmpeg/darwin/ffmpeg`.
- it writes the settings that must not stay default into `~/Library/Application Support/abogen/config.json`: Audio Format m4b (default wav), voice af_heart (Kokoro's only A-grade voice), and contraction expansion off (it rewrites quotations as "do not", "it is"). Other settings, and pronunciation overrides (`overrides.json`), keep whatever was set in the UI.

To upgrade Abogen, run `uv lock --upgrade` in `tts/`, then check a preview and a short job before committing the new lock. Abogen's README still says `numpy<2` (abogen issue #50); that crash is gone with torch 2.15, and NumPy 2 was tested at the same speed. espeak-ng silently fails ("Error processing file '/Users/runner/work/espeakng-loader/…/phontab'") if the venv path is longer than about 160 characters, so keep the checkout at a short path. `abogen-cli` is the same web server; Abogen has no real CLI.

## 4. The user runs the job in the browser

Don't start the conversion yourself. Walk the user through:
- Upload `WORK/Title.txt`. Book parameters: fill in Subtitle, Author, Publisher (Abogen currently ignores them in the M4B tags anyway; see step 5).
- Chapters: untick Front matter, Contents, notes/bibliography and Index.
- Review entities: no casting for single-narrator nonfiction. Add the overrides from `proofread join`, most frequent first, and test them with the voice Preview. Overrides must be plain respellings ("Koh-mus"): Kokoro reads IPA symbols aloud and spells out ALL-CAPS syllables.
- Speed is about 3.4× real time on the M2's GPU, so a 10-hour book takes about 3 hours. The Mac must stay awake (`caffeinate -i`).

## 5. Deliver

Output is saved next to the input file. Copy the M4B and the proofread `.txt` (so the book can be re-voiced without proofreading again) to servarr:
```bash
ssh servarr 'mkdir -p "/data/usenet/complete/audiobooks/Author/Title"'
scp WORK/*.m4b "WORK/Title.txt" "servarr:/data/usenet/complete/audiobooks/Author/Title/"
```
Audiobookshelf picks it up from that folder. The M4B's title/author tags come out as a hash and "Unknown", so set them in Audiobookshelf with Match or Edit.
