# Proofreading brief: OCR fixes for a TTS audiobook

A scanned PDF was OCR'd, then cleaned by `scripts/pdf2tts` into plain text, one file per chapter. The text will be read aloud by a TTS engine (Kokoro), so fix what would be *heard* wrong.

Files are in `{work}/`:
- `ch/NN.txt` are the chapter texts. Read your assigned files in full. Do not edit them.
- The source scan is `{pdf}`, for checking ambiguous spots (see below).
- Write your output to `fixes/NN.json`, one per chapter file.

## What to fix (OCR and conversion damage only)

- **Missing drop cap.** The first body paragraph of a chapter often lacks its decorative first letter ("he white men" → "The white men"), and stray junk letters often appear nearby ("out of C their" → "out of their").
- **Misread characters and words**: "Tzmes" → "Times", "“T don't" → "“I don't", "ina city" → "in a city".
- **Junk fragments** inside sentences, e.g. "(qe", "Se LE SA", stray single capitals.
- **Wrongly joined or split words**, including hyphenated compounds wrongly joined ("oldline" → "old-line").
- **A paragraph wrongly split** mid-sentence or mid-quote (usually at a page break). Fix by replacing the "\n\n" between the halves with a space.
- **Wrong capitals mid-sentence** from OCR ("filed suit Claiming" → "filed suit claiming").

## What NOT to do

- Do not reword, modernise, "improve" style or punctuation, or fix the author's own spelling choices or period quotations (quoted old documents may have archaic spelling, so leave them).
- Do not touch curly vs straight quotes or apostrophes, em-dashes, or `<<CHAPTER_MARKER:...>>` lines, except that you may fix OCR errors inside the spoken title line just below the marker (and then fix the marker to match).
- When unsure whether something is an OCR error, check the PDF, or leave it and mention it in `uncertain`.

## Checking the PDF

Find a page by searching:
`uvx --quiet --with pymupdf python -I -W ignore -c "import pymupdf,sys; d=pymupdf.open(sys.argv[1]); print([i for i,p in enumerate(d) if sys.argv[2] in p.get_text()])" {pdf} "some phrase"`
Render a page to look at it:
`uvx --quiet --with pymupdf python -I -W ignore -c "import pymupdf,sys; pymupdf.open(sys.argv[1])[int(sys.argv[2])].get_pixmap(dpi=110).save(sys.argv[3])" {pdf} 40 {work}/page40.png`
then Read the PNG. Use this sparingly, for cases you can't settle from context.

## Output format: `fixes/NN.json`

```json
{{
  "fixes": [
    {{"old": "he white men in jackets", "new": "The white men in jackets", "count": 1}}
  ],
  "pronounce": [
    {{"word": "krewe", "respelling": "crew", "count": 41}}
  ],
  "uncertain": ["short note on anything you left alone but suspect"],
  "script_notes": ["patterns a cleanup script could fix generically, with an example"]
}}
```
- `old` must be an exact substring of the file (copy it exactly, including curly quotes), long enough to be unambiguous. `count` is how many times `old` occurs in the file; every occurrence gets replaced, and a fix whose count is wrong is skipped. Verify counts with `python3 -c` and `str.count`.
- Keep each fix minimal: `old` is the error plus a few words of context.
- `pronounce`: names and non-English or regional words a US-English TTS voice would likely say wrong, used 3+ times in your chapter, with how often. `respelling` is plain English letters with hyphens and an initial capital only ("Koh-mus", "Chop-uh-too-lus"): no IPA, no stress marks, no ALL-CAPS syllables (the TTS spells those out). These become Abogen pronunciation overrides.
- `script_notes`: recurring damage that code could catch on any book (junk, header or caption leftovers, paragraph-joining mistakes, hyphenation errors), for improving `scripts/pdf2tts`. Skip one-off OCR misreads.

Validate that each JSON file loads with `python3 -c 'import json,sys; json.load(open(sys.argv[1]))' file`.

## Final reply

Keep it short: per file, the number of fixes, the `uncertain` items, and the `script_notes`. Don't list every fix; they're in the JSON.
