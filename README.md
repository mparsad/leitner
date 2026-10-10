# Handoff: Leitner — English vocabulary review (mobile)

## Overview
A mobile-first Leitner spaced-repetition app for English vocabulary. The user screenshots a word in the Longman dictionary, gives it to Claude with a fixed prompt, gets back a `word_entry = {...}` code block (Python-dict style), and pastes it into the app. The app parses the code, stores the word, and schedules reviews in 4 boxes: **1 day → 3 days → 1 week → 1 month → Learned (out of rotation)**.

## About the Design Files
The files in this bundle are **design references built in HTML**. They are working prototypes that show the intended look and behavior. They are not production code to copy directly. Your task is to **rebuild this design in the target environment**. If there is no existing codebase, pick a suitable stack. Recommended: **React + Vite + TypeScript as an installable PWA** (offline, home-screen icon, localStorage or IndexedDB). The whole app is client-side, with no backend.

`Leitner.dc.html` opens directly in a browser. It needs `support.js` and the `_ds/` folder next to it, served over http (e.g. `npx serve .`). All logic is in the `<script data-dc-script>` block: the `class Component` and the helper functions above it. Port that logic faithfully.

## Fidelity
**High-fidelity.** Colors, type, spacing, borders and interactions are final. Recreate them closely.

---

## Data model

```ts
type Sense = { number: number; pattern: string; definition: string; synonyms: string[]; examples: string[] };
type Derived = { word: string; part_of_speech: string; examples: string[] };
type Leitner = {
  box: 1 | 2 | 3 | 4;
  added_date: string;      // "YYYY-MM-DD", local date
  next_review: string;     // "YYYY-MM-DD"
  last_review?: string;
  correct_count: number;
  wrong_count: number;
  learned: boolean;
  learned_date?: string;
};
type Word = {
  id: string; word: string; syllables: string; pronunciation: string; part_of_speech: string; source: string;
  senses: Sense[]; derived_forms: Derived[]; persian_meaning: string; tags: string[]; leitner: Leitner;
};
type Store = { words: Word[]; log: Record<string /*date*/, { c: number; w: number }>; mode: 'flip' | 'type' };
```
Persisted as JSON in `localStorage["leitner.v1"]`. When storage is empty, the prototype seeds 6 demo words (`seedData()`). **Drop the seed in production** and start with an empty state.

## Leitner rules (exact)
- Intervals: `{1: 1, 2: 3, 3: 7, 4: 30}` days.
- **New word**: box 1, `next_review = today + 1`. If the imported code has `leitner.box` 1–4, that box is used. Any imported dates are ignored.
- **Due**: `!learned && next_review <= today`. Due items are sorted by box ascending, then by next_review.
- **Correct**: `correct_count++`. If box < 4, box +1. If box == 4, `learned = true` and the word leaves rotation.
- **Wrong**: `wrong_count++`. box = max(1, box − 1). **The word moves back one box; it does not reset to box 1.**
- After grading: `next_review = today + IV[box]`, `last_review = today`, and `log[today].c` or `log[today].w` increments.
- **Requeue wrong** (setting, default on): a missed word is appended to the end of the current session once more as a "practice round". That round doesn't change the box or the counts.
- **Re-importing an existing word** (case-insensitive match on `word`): content is replaced and tags are merged. Persian meaning is kept if the new one is empty. **Leitner progress is kept.**
- Manual move (detail screen): set any box 1–4 (resets next_review = today + IV[box]) or mark Learned.

## Import parser (critical)
Input is Claude's output, Python-dict syntax. The parser must accept:
- An optional `word_entry =` prefix (any identifier `=`); several entries in one paste; an array of entries.
- `None/True/False`, `#` comments, `//` comments, trailing commas, single or double quotes, curly quotes, unquoted keys.
- Behavior: tokenize to JSON (`pyToJson`), then split top-level `{…}`/`[…]` chunks, run `JSON.parse` on each, and keep the objects that have `word`. Then normalize (`normContent`): missing arrays become `[]`, `pattern: None` becomes `""`.
- Errors: "The code looks cut off — a bracket is not closed." / "No entry with a "word" field was found." / the JSON parse message. Shown inline as: `Could not read this code: …`.
- A live preview lists the parsed words with a **New** or **Update** badge as the user types.

### Prompt for Claude (shown in the Add screen with a Copy button; keep the text verbatim)
See the `<pre>` in the Add screen of `Leitner.dc.html`. It asks Claude to output only `word_entry = {...}` blocks with the fields: word, syllables, pronunciation, part_of_speech, source, senses[number, pattern, definition, synonyms, examples], derived_forms[word, part_of_speech, examples], persian_meaning, tags.

---

## Screens / Views
Layout shell: a centered column, `max-width: 480px`, `min-height: 100vh`, with 1px divider borders left and right. A sticky header sits on top and a sticky 4-tab bar on the bottom. The tab bar is hidden during review. Main padding: `20.4px 13.6px 27.2px`. Vertical gap between sections: `20.4px`.

**Header** (`.nav`): the brand mark is a 12×12 square with a 1.5px accent border, followed by "LEITNER" (letter-spacing .06em). On the right is today's date (`en-GB`: "Wed 7 Oct"), 12px uppercase, neutral-700.

**Tab bar**: 4 equal columns (Today / Add / Words / Progress), each 56px tall at minimum, with a 22px Lucide icon (stroke 1.5) above an 11px uppercase label. Active tab is accent-700; inactive is neutral-700. The Words tab stays active on the detail screen.

### 1. Today (home)
- **Due card** (blueprint frame):
  - Kicker "Due today" (accent-700). On the right: "N in rotation", 12px.
  - The due count: Barlow Condensed 600, **104px**, line-height .85.
  - A note line. Use the text that fits the state:
    - Empty: "No words yet. Add your first Longman entry."
    - Due items exist: "Knew it moves a word up one box. Didn't know moves it back one."
    - Nothing due: "All caught up. Next: tomorrow · 3 words."
    - Nothing left: "All caught up — nothing left in rotation."
  - **Review mode** segmented control: "Flip card" / "Type the word". Persisted in `store.mode`.
  - A **Topic** select ("All topics" plus the tags). Shown only if any tags exist; it filters the due set.
  - Primary blueprint button "START REVIEW", 52px tall, 18px text. Disabled when nothing is due.
- **Boxes**: a 4-column grid of hairline cells, each at least 132px tall: "BOX 01" (10px, .12em, accent-700), the label ("1 day"…; 15px condensed uppercase), the count (38px condensed), and "N due" or "none due" (11px). Tapping a cell opens Words filtered to that box. Below the grid, a full-width "Learned" row with a check icon and the count opens Words filtered to Learned.

### 2. Review session
- Top row: progress label "3 / 12" (append " · again" on a practice round), a 4px progress bar (neutral-300 track, accent fill), and an × icon button (40px) that ends the session.

**Flip mode**
- Front: a blueprint card with a box tag ("Box 2 · 3 days"), the source label, the word (h1 54px), syllables plus IPA, a 44px speaker icon button (speechSynthesis, `en-GB` default, rate 0.9), and the part of speech in italic accent-700.
- Sticky bottom: "SHOW ANSWER" (primary, 54px).
- After reveal, the body appears:
  - Numbered senses: the number is condensed 24px in the accent. Then the bold pattern, the definition (16px / 1.45), a "SYN" tag with synonyms, and examples as italic "› …".
  - Derived forms, the Persian meaning (RTL, Vazirmatn, labelled "FA"), and tags.
- Sticky grade buttons: "Didn't know" (secondary) | "Knew it" (primary). Under each is the destination hint, e.g. "→ Box 1 · 1 day" / "→ Box 3 · 1 week" / "→ Learned ✓", or "Practice round".

**Type mode**
- A blueprint card: "Which word is it?" plus the box tag, the part of speech, then numbered definitions and examples. The headword is masked as `_____`: case-insensitive regex on the first `max(3, len−2)` letters + `\w*`, so inflected forms are masked too. The Persian meaning is shown if present.
- An input ("Type the English word", autocorrect and autocapitalize off) with a "Check" button; Enter also checks. A ghost button "I don't know — show me".
- Comparison: lowercase, trimmed, leading "to " stripped, whitespace collapsed; must match exactly.
- Result banner:
  - Correct: accent border, accent-100 fill, accent-800 text, "Correct".
  - Wrong: neutral-200 fill, "Not quite — you typed "x"".
  - Gave up: "Answer revealed".
- After the result, the full entry (header and body) is shown. Sticky "NEXT" button. If the answer was wrong, a ghost link "Typo — count it as correct" also appears. Hint text: "Moves to Box 3 · 1 week".

**Session complete**: a blueprint card titled "Session complete" with a 2-cell grid: MOVED UP / MOVED BACK, 48px numbers. Below it, "N still due today." or "Nothing else due today.", and the button "BACK TO TODAY".

### 3. Add
- Title "Add words" and a helper line.
- A monospace textarea, at least 220px tall, 12px.
- Error box (neutral-200 fill).
- Preview rows: word (20px condensed) + meta ("adjective · 3 senses · 1 derived") + a New or Update tag. Under the rows, a "Topic tags for these words" input (comma separated).
- Primary button: "IMPORT WORD" or "IMPORT N WORDS", disabled when nothing parses. Success message: "2 added to Box 1 · 1 updated (progress kept). First review: tomorrow."
- A "How to get the code" blueprint card with 3 numbered steps, the prompt `<pre>` (max-height 200px, scrollable, 11px mono), and a "Copy prompt for Claude" secondary button. The button label changes to "Copied" for 1.8s.

### 4. Words (list + search)
- A search input with a magnifier icon, 44px tall. It matches word, Persian meaning, tags, definitions and derived words.
- Filter chips: All, Due, 1 day, 3 days, 1 week, 1 month, Learned. Active chip is `.tag.tag-accent`; others are `.tag`. All chips are 34px tall.
- A topic select (if any tags exist) and a count line ("12 words").
- Rows sorted A–Z. Each row has the word (21px condensed) + POS (italic accent-700) on the first line and the first definition (single line, ellipsis) below. On the right: a box tag and "Due now" / "Tomorrow" / "In N days" / "Done". Tapping a row opens the detail screen.
- Empty state: "No words match."

### 5. Word detail (edit)
- "← All words" ghost button, then the same entry header and body as review.
- **Leitner facts** in a 4-cell grid: Box, Next, Knew, Missed.
- "Move to box" segmented control: 1 day | 3 days | 1 week | 1 month | Learned.
- **Edit**:
  - A Persian meaning input (RTL) and a topic tags input, with a Save button. "Saved" shows for 1.8s.
  - A `<details>` section, "Edit full entry code": a JSON textarea that goes through the same parser, plus a Save code button.
- "Delete word", which asks for inline confirmation: "Delete "x"?" with Delete and Cancel.

### 6. Progress
- 2×2 tiles: Words, Learned, Accuracy (correct ÷ total, "—" if none), Reviewed today. Numbers are 40px condensed.
- Words by box: horizontal bars (14px, hairline frame, accent fill), as a percentage of all words, for each of the 4 boxes plus Learned.
- **Reviews · last 14 days**: a stacked column chart, 120px tall, accent for Knew and neutral-400 for Missed, with a legend. Day-of-month labels below; today's label is accent-700.
- **Coming up**: a 7-cell grid. "Today" counts everything due now; the other cells count words due exactly that day, labelled with the weekday short name.

---

## Settings (exposed as tweaks in the prototype; put them in a Settings screen)
- `accent`: `en-GB` (default) | `en-US`. Sets the speech voice.
- `autoSpeak`: false. Speaks the word when a card appears (flip mode) or after checking (type mode).
- `requeueWrong`: true.

## Suggested additions (not in the prototype)
- Export/import backup (JSON download/upload).
- PWA manifest + service worker.
- Optionally use IndexedDB instead of localStorage.

## Design Tokens (Industry design system, from `_ds/.../styles.css`)
- **Colors**
  - Ground and text: bg `#f2f2f3`, surface `#e9e9ea`, text `#1d1f20`, divider = text at 16% alpha.
  - Accent `#5980a6`. Accent ramp: 100 `#eef6ff`, 200 `#d6ebff`, 300 `#b5d9fd`, 400 `#94bce3`, 500 `#749dc4`, 600 `#597ea3`, 700 `#416180`, 800 `#2c455d`, 900 `#1d2d3d`.
  - Neutral ramp: 100 `#f5f5f8`, 200 `#e7e7ea`, 300 `#d4d4d7`, 400 `#b7b7ba`, 500 `#98989b`, 600 `#7a7a7d`, 700 `#5d5d60`, 800 `#424244`, 900 `#2b2b2d`.
- **Type**: headings in Barlow Condensed 600; body in Barlow (Google Fonts); Persian text in Vazirmatn 400/500.
- **Spacing**: 3.4 / 6.8 / 10.2 / 13.6 / 20.4 / 27.2 px (space-1…8).
- **Radius**: 2 / 4 / 7 px. Cards and buttons are square-cornered.
- **Shadows**:
  - sm `0 1px 2px` at 14% ink
  - md `0 3px 10px` at 16% ink
  - lg `0 12px 32px` at 22% ink
- **Blueprint frame**: hairline border plus "+" registration marks at all 4 corners. Used on hero cards and primary buttons. Copy `.blueprint` and `.corner` from styles.css.
- **Components used**: `.btn` (primary, secondary, ghost, icon), `.tag` (accent, neutral), `.input`, `.field`, `.seg` / `.seg-opt`, `.nav`, `.card-kicker`.
- **States**:
  - Focus: `outline: 2px solid accent; offset 2px`.
  - Disabled: 45% opacity.
  - Hover on cells and rows: accent at 6–8% tint.

## Assets
- Icons: Lucide at stroke-width 1.5 (home, plus, list, bar-chart, volume-2, x, check, search, copy, arrow-left).
- No images.

## Files
- `screenshots/` — 01-home, 02/03 review flip (front/answer), 04/05 review type (prompt/result), 06 session complete, 07 add/import, 08 words list, 09 word detail, 10 progress. They show only the top viewport (≈540px) of each screen; scroll content is described above.
- `Leitner.dc.html` — the full prototype: template plus logic. The logic includes the parser, the scheduler, seed data and all screens.
- `support.js` — the prototype runtime (reference only; not needed in production).
- `_ds/industry-…/styles.css` — design tokens and component classes.
- `_ds/industry-…/_ds_bundle.js` — the design-system bundle.
