# Paragraph Structure Checker (Azure OpenAI variant)

A student-facing web page that checks whether a single pasted paragraph meets basic structural requirements. Built from the Syllabus Bot (Azure) template: the page lives on GitHub Pages, a small Cloudflare Worker does the checking, and the language-judgement step runs on your institution's Azure OpenAI deployment.

**Privacy change from the template:** student writing is copyright-protected, so this version has **no Qualtrics logging, no research consent letter, and no stored text of any kind.** The paragraph goes to the worker, on to Azure OpenAI for judgement, and the report comes back. Nothing is written down.

## What it checks

| Requirement | How it is judged | Result shown to student |
|---|---|---|
| **Single paragraph** | Code: counts line-break-separated blocks | Complete / Incomplete, with number of paragraphs found |
| **Three distinct quotations** | Code: finds “curly”, "straight" and ‘single’ quotations of 3+ words; repeated quotes count once | Complete / Incomplete, with "N of 3 distinct", repeats named, unclosed quote marks flagged |
| **Topic sentence** (sentence 1) | Code catches questions, exclamations, fragments, "This paragraph will discuss…"; Azure judges declarative and clear | Meets / Needs revision, with each issue listed |
| **Evidentiary claim** per quotation | Azure: is the quote tied to a specific claim and a named noun or instance (author, text, provision, event…)? | Per-quotation pass/fail, reason, and "X of N lack one" |
| **Analysis after each quotation** | Code: flags no following sentence, another quotation, or reused wording (4+ shared words). Azure: does the *immediately following sentence*, on its own, state the quotation's evidentiary value in the student's words and match what the quote says? | Per-quotation pass/fail with the category (restates, continues the argument, assumes it is self-evident, mismatch, nothing follows), the quote, and the next sentence |
| **Negative framing** | Azure: claims built on what something is not, lacks, or fails to do | Count, each sentence, signal words, "If not X, then Y" |
| **Negative parallelism** | Code patterns ("not X but Y", "not only… but also", "isn't X; it's Y", "X, not Y", "rather than", "never X; rather Y", "neither… nor", "less X than Y", "more than just X") plus Azure | Same as above |

Paragraph and quotation count are "soft" (complete/incomplete only). The others are the substantive checks and always give specific instances with counts.

**"If not X, then Y":** for every negative instance the report gives X (what is denied) and Y (what is asserted or implied instead), e.g. *If not “partnership”, then “Canada’s foreign policy… is defined by asymmetry”*, so the student sees the positive claim they can state directly. The model is told not to rewrite the student's sentences.

Negatives inside direct quotations are ignored, since they belong to the source.

**If Azure is unavailable or its content filter declines a text**, the student still gets the code-based results (paragraph, quotes, obvious negative patterns, rule-based topic-sentence problems). AI-judged sections are shown as "Not checked" and the page says why.

## Setup

1. **Create the repo** from this template on GitHub (name it e.g. `paragraph-checker-azure`).
2. **Deploy the worker:** dash.cloudflare.com → Compute (Workers) → Create → Hello World → name it → Edit code → select all, delete, paste `worker.js` → Deploy. Visiting the worker URL should show "Method Not Allowed".
3. **Set variables** (Settings → Variables and Secrets): see `env-vars-checklist.txt`. Required: `AZURE_OPENAI_KEY` (Secret), `AZURE_ENDPOINT`, `AZURE_DEPLOYMENT_NAME`. Recommended: `ALLOWED_ORIGIN` = `https://<username>.github.io` so only your page can use the worker. Optional: `AZURE_API_VERSION`.
4. **Point the page at the worker:** in `index.html` set `WORKER_URL`, commit.
5. **Publish:** repo → Settings → Pages → branch `main`, folder `/ (root)`.
6. **Brightspace:** put your Pages URL in the iframe `src` of `brightspace.html`, and paste that in as a content item.

## Tuning the rules

Constants at the top of `worker.js` (redeploy after editing):

- `REQUIRED_QUOTES` (3) and `MIN_QUOTE_WORDS` (3): quoted strings under 3 words are treated as terms or scare quotes, listed as "not counted", and do not count toward the three.
- `MAX_CHARS` / `MIN_WORDS`: input limits.
- The rubric wording the model applies is in `SYSTEM_PROMPT`. Change definitions there (for example, what counts as a "named instance").

## Testing

`npm test` (Node 18+) runs 21 tests against the worker with Azure mocked. No keys or network needed. Re-run after editing `worker.js`.

## Known limits

- The topic sentence is taken to be the **first sentence**.
- "Immediately following" means the next sentence, strictly. If a student explains the quote in the same sentence ("…, which shows…") or two sentences later, the check fails by design: the explanation must stand on its own after the evidence. Two quotes in one sentence share one following sentence, and each is judged separately.
- Text pasted from a PDF often has hard line breaks that read as several paragraphs; the report says so.
- Straight single quotes (`'like this'`) are not treated as quotation marks, because they are indistinguishable from apostrophes.
- The AI-judged parts (topic sentence clarity, evidentiary claims, negative framing) are judgements, not proofs; the page tells students to treat them as a prompt to re-read. Spot-check with real student paragraphs before relying on it.
- A model can miss or over-flag negatives. The code patterns are a floor for parallelism, not a ceiling.

## Data handling — confirm with your privacy office / IT

Where the text travels: student browser → **Cloudflare Worker** (relay, in transit only, never stored or logged by this code) → **your Azure OpenAI resource** → back. Two things worth confirming for a copyrighted-student-work policy:

1. **Cloudflare is a third-party processor in the middle.** If policy requires the text to stay wholly inside Microsoft, the worker logic (a single `fetch` handler) can be moved to an Azure Function / Static Web Apps API with little change.
2. **Azure OpenAI abuse monitoring** may retain prompts for a limited period by default unless your tenant has an approved exemption. Check your resource's data-retention settings.

The GitHub repo and Pages site contain only code, never student data. Do not enable Cloudflare Workers Logs / Logpush for this worker, since those could capture request content.

## Files

- `index.html` — student interface
- `worker.js` — Cloudflare Worker backend (the running copy lives in Cloudflare; this is the backup)
- `brightspace.html` — LMS iframe wrapper
- `tests/worker.test.js`, `package.json` — tests
- `env-vars-checklist.txt` — worker variables

## License

© Dan Bousfield. CC BY 4.0 — https://creativecommons.org/licenses/by/4.0/
