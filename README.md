# Hodge Memorial — Clubhouse Board

A live leaderboard and trash talk board for the Hodge Memorial golf trip.

Squabbit runs the tournament. This reads Squabbit's public leaderboard, stores it
in a Google Sheet, and serves it to a TV in the clubhouse and a phone page where
the field can talk trash at each other.

Zero cost. No server. No build step.

---

## How it works

```
  Squabbit (source of truth)
    app.squabbitgolf.com/w/tournament/{ID}
              │
              │  Apps Script scrapes it every 5 min
              ▼
  Google Sheet  ──  boards, roster, trash talk, history snapshots
              │
              │  /exec  (JSON in, JSON out)
              ▼
  GitHub Pages
    board.html   →  the TV
    talk.html    →  everyone's phone
```

Squabbit stays the system of record. Scores are entered in the app exactly like
always. Nothing here writes back to Squabbit — if this whole thing falls over,
the tournament is unaffected.

---

## Files

| File | What it is | Endpoint it needs |
|---|---|---|
| `index.html` | Recommend-a-golfer form | **Recommendation** script |
| `board.html` | TV leaderboard | **Squabbit** script |
| `talk.html` | Trash talk, phones | **Squabbit** script |
| `cup.html` | Hodge Cup scoring — upload the export | **Squabbit** script |
| `Code.gs` | Scraper + API (lives in Apps Script) | — |

---

## The Hodge Cup

An overall winner, on top of World's #1 (low gross) and The Max (low net).

| Source | Points |
|---|---|
| Ryder Cup match point | **×3** (max 4 → 12) |
| Low gross finish | 10 / 7 / 5 / 3 / 2, then 1 for 6th–10th |
| Low net finish | same ladder |
| Skin | 0.5 each |

**The gate: win at least one match point or you are not eligible.** Shoot the
lights out, beat nobody, get nothing. Your name still shows on the board with
your score — struck through. That's the point.

**Where the numbers come from**

Everything except match results is scraped automatically. Match results are
*not on Squabbit's public page* — they only exist in the signed-in CSV export.
So `cup.html` is the one manual step: sign in, export, drop the file on the
page, hit publish. It reconstructs all 35 matches hole by hole and pushes the
standings to the Sheet.

**How the reconstruction works**

The export gives hole-by-hole scores plus the format each round used
(`Best Ball`, `Scramble`, `Singles Matchplay`) and groups players into
foursomes by tee time. A foursome in a team round *is* a match — two a side.
Team score on a hole is its better ball. Run match play, close it out when a
lead exceeds the holes left.

Validated against 2025: **rounds 2, 3 and 4 reconstruct exactly**, including
all 14 singles at 5.5–8.5. Round 1 comes out Michigan 5, where Squabbit
recorded 4 — one match (Scherf & Desmarteau vs Westerhouse & Hauter, 1 up on
18) disagrees. Probably conceded or adjusted on the day. Worth remembering that
this reconstruction is very good, not gospel.

**Gross excludes the scramble.** A scramble score isn't yours — you and your
partner post the same number. The page detects the scramble round from the
format label. Drop it, and 2025's low gross comes out Will Crane +9 over Kellen
VanDreumel +11, which is the result that actually happened.

*Minor known discrepancy: the CSV credits Jason Bergman 12 skins, the public
widget says 13. They disagree by one in R3. The Cup uses the CSV.*

---

### ⚠️ Two different Apps Script projects

`index.html` posts to the golfer-recommendation script. `board.html` and
`talk.html` talk to the Squabbit script. **They are not the same URL.** Pasting
the wrong one produces no error and no data, which is the worst kind of bug.

---

## Setup

### 1. The Apps Script

1. Create a new Google Sheet → **Extensions → Apps Script**
2. Select all in the editor, delete, paste `Code.gs`
   *(paste over an empty file — if you paste inside the default `myFunction`
   stub, everything ends up nested and nothing runs)*
3. Run `previewParse`. Authorize when prompted. Check the log:
   three boards, ~28 rows each.
4. Run `refresh` once. It creates the tabs.
5. **Deploy → New deployment → Web app**
   - Execute as: **Me**
   - Who has access: **Anyone**
6. Copy the **`/exec`** URL. Not `/dev` — `/dev` only works while you're
   logged in, so it'll work on your laptop and show nothing on the TV.
7. **Triggers** (clock icon) → Add trigger → `refresh` → Time-driven →
   Minutes timer → **Every 5 minutes**

### 2. The pages

Paste the `/exec` URL into both:

```javascript
// board.html
API_URL: 'https://script.google.com/macros/s/AKfy.../exec',

// talk.html
const ENDPOINT = "https://script.google.com/macros/s/AKfy.../exec";
```

Commit. GitHub Pages serves them at:

- `scherf6.github.io/hodge/board.html`
- `scherf6.github.io/hodge/talk.html`

### 3. The TV

Point a Fire Stick / smart TV browser at `board.html`. It rotates boards on its
own and re-pulls every 60 seconds. Nothing to click.

---

## Every year: the flip

When the new tournament is built in Squabbit:

1. Open the tournament's public page, copy the ID out of the URL:
   `app.squabbitgolf.com/w/tournament/` **`TILGwVRXI`**
2. In `Code.gs`, change one line:
   ```javascript
   TOURNAMENT_ID: 'TILGwVRXI',
   ```
3. If you configured a different mix of games, update the labels **in the order
   they appear on the Squabbit page**:
   ```javascript
   BOARD_NAMES: ['Ryder Cup', 'Skins', 'Strokeplay'],
   ```
   These are positional, not detected. Wrong order = right numbers under wrong
   headings, which nobody notices until someone gets loud about it.
4. **Deploy → Manage deployments → pencil → Version: New version → Deploy**
5. Clear the `History` tab so last year's snapshots don't pollute the recaps.

**Known IDs**

| Year | ID | Course |
|---|---|---|
| 2025 | `TJQeP4MeT` | Shanty Creek — Hawk's Eye, Legend, Summit, Cedar River |
| 2026 | `TILGwVRXI` | — |

---

## Gotchas

**Editing `Code.gs` does not update the live endpoint.** Apps Script pins a
deployment to a code version. You must cut a **new version** under Manage
deployments or `/exec` will serve stale code forever. The URL doesn't change.
This catches everyone exactly once.

**Test in incognito.** Logged into your own Google account, a misconfigured
endpoint can look like it works. Incognito is a truthful test of what the TV
sees.

**The trash talk POST sends `text/plain` on purpose.** A JSON content-type makes
the browser fire a CORS preflight, and Apps Script cannot answer one. Don't
"fix" this.

**Squabbit returns a placeholder until the first scores post.** The payload
carries `live: false` and no boards. The pages handle it — `board.html` shows a
not-started state rather than an empty grid.

**The roster is derived from the leaderboard.** Which means before the first
round is scored, there is no roster, so the trash talk dropdown is empty. If you
want trash talk running during the buildup, add a manual roster fallback to
`CONFIG` before you flip the ID.

**Be a good citizen.** Squabbit is free and run by a small team who answer their
own support email. Five minutes between polls is plenty. Don't crank it down.

---

## The Sheet

| Tab | Contents |
|---|---|
| `Ryder Cup` / `Skins` / `Strokeplay` | Latest scrape, human-readable |
| `Talk` | Trash talk posts — `at`, `name`, `message` |
| `History` | A full JSON snapshot every 5 minutes |

`History` is the interesting one. By Sunday night it holds a few hundred
snapshots, which means you can compute *movement* — who climbed, who tumbled,
who was leading Saturday afternoon and gave it away at Cedar River. That's the
raw material for the nightly recap. Delete a snapshot and you lose that moment
permanently, so leave it alone.

---

## Not built yet

- **Nightly AI recap.** Read the day's `History` snapshots, find who moved,
  write the roast, post it to the ticker. Needs an Anthropic API key in Script
  Properties (never in the client — these pages are public).
- **Manual roster fallback** so trash talk works before round one.
- **Skins carryover math**, if we ever want to show what's actually on the line.
