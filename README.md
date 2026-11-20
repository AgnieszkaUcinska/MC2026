# MC2026 — Delivery Forecast

A client-side Monte Carlo delivery forecasting dashboard, served as a GitHub Pages site:
**https://agnieszkaucinska.github.io/MC2026/**

Everything runs in the browser. There is **no build step, no server, no dependencies, and no
network calls** — the whole app is the single file [`index.html`](index.html). Your data never
leaves your machine.

## The two tabs

- **Backlog forecast** — "when will the backlog be cleared?" Forecasts a flat backlog of items
  from historical throughput, with a fixed-scope vs. scope-creep prognosis.
- **Epic forecast** — "when will these 6 Epics, worked in order, all be done?" The richer model:
  focus factor, per-Epic learning effect, seasonal uplift, holiday capacity dips, size ranges per
  Epic, and full explain-on-demand. Supports saved scenarios, JSON export/import, shareable links
  and CSV export.

## How the forecast works (short version)

We run thousands of imaginary deliveries ("trials"). Each trial samples a throughput for every
sprint by drawing — at random, with replacement — from your real past sprints (a *bootstrap*),
adjusts it for focus / season / holidays, draws a size for each Epic within its min–max range,
and works the Epics in order (surplus capacity carries to the next Epic). We record when each
trial finishes, then read probabilities and dates off the whole distribution. A fixed random
**seed** makes every run reproducible and shareable.

The heavy simulation runs in a **Web Worker** so the page stays responsive; 10,000 trials update
in well under a second.

---

## How to update (for a non-developer)

You edit one file, `index.html`, then commit and push. GitHub Pages redeploys automatically
(about a minute). You do **not** need to install anything.

```bash
# after editing index.html:
git add index.html
git commit -m "Update forecast defaults"
git push
```

### Where to change the starting numbers (defaults)

- **Epic forecast defaults** live in one place: the `DEFAULTS()` function inside the second
  `<script>` block (search the file for `function DEFAULTS()`). Dates, sprint length, focus factor,
  the six Epics (name / min / max / familiarity), seasonal multipliers, the Christmas capacity
  override, the history values, trials, confidence and seed are all there.
- **Backlog forecast defaults** are the `value="…"` attributes on the inputs inside
  `<details class="assume">` in the first tab, and mirrored in the "Reset to defaults" handler
  (search for `resetBtn`).

Anyone using the app can also change every one of these live in the **Assumptions & inputs**
panel — the defaults are just the starting point.

### Where to change the explanation texts

All the plain-language help lives in the Epic script:

- The `EXPLAIN` object — the text behind every **?** button (one entry per chart/input).
- The `showMethod()` function — the **Methodology & glossary** drawer (glossary + limitations).
- The `explainResults()` function — the templated **"Explain my results"** narrative.

They are ordinary strings; edit the words, commit, push.

### How a deploy happens

Pushing to the `main` branch triggers GitHub Pages to rebuild and publish the site from the repo
root. There is no CI pipeline to wait on — the file *is* the site.

---

## Shared scenarios (Firebase) — one-time setup

The Epic tab has a **Shared scenarios (team)** panel so colleagues can publish and load scenarios
from a shared cloud library, unlocked with one **team password**. It uses a free Firebase
Firestore database. Until you configure it, the panel shows a "not configured" notice and the rest
of the app works normally.

**How the password protects your data:** when someone publishes, the scenario is encrypted in their
browser with a key derived from the team password (PBKDF2 → AES-GCM). Only the encrypted blob is
uploaded, so Firestore never sees your numbers, names, or the password. Entering the correct
password is what makes the library readable. ⚠️ With no login, the database is *open* — the
password protects the **contents** (confidentiality), not against someone deleting entries. That's
fine for a trusted team; for stronger control add Firebase Authentication later.

### Steps (about 5 minutes, no credit card)

1. Go to **https://console.firebase.google.com** → **Add project** (accept defaults; you can skip
   Google Analytics).
2. In the project, click the **Web** icon (`</>`) to "Add an app". Give it a nickname, register it,
   and copy the `firebaseConfig` object it shows you (apiKey, authDomain, projectId, …).
3. Left menu → **Build → Firestore Database → Create database**. Choose **production mode** and a
   region near your team.
4. Open the **Rules** tab and paste the rules below, then **Publish**:

   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /scenarios/{doc} { allow read, create, delete: if true; allow update: if false; }
       match /_meta/{doc}     { allow read, create: if true; allow update, delete: if false; }
     }
   }
   ```

5. In `index.html`, search for `FIREBASE_CONFIG` (near the top of the Epic script) and replace the
   `PASTE_ME` values with the config from step 2. Commit and push.
6. **Share the team password privately** (chat/1Password — *not* in the repo). The first person to
   unlock with a password sets it for everyone; after that, a wrong password is rejected.

Notes: the shared panel needs **HTTPS** (the live Pages site works; opening the local file won't do
encryption). One Firebase project = one team password. Firestore's free "Spark" tier is far more
than a scenario library needs and the database never sleeps.

---

## The model, and its limits

- The past is assumed to represent the future. A reorg, new tech, or team change breaks that.
- Sprints are assumed independent (no momentum or fatigue carried between them).
- No scope creep is added on the Epic tab unless you widen the Epic size ranges.
- Estimates are treated as ranges, not certainties.
- Focus factor, seasonal uplift and familiarity are human guesses — the **Sensitivity (tornado)**
  chart shows which of them your forecast depends on most, so you know where to focus estimation
  effort.

> **Focus factor note:** the default assumes history was measured at full focus, so it scales
> throughput down to 75%. If your historical throughput *already* reflects split focus, set the
> focus factor to 100% (there's a one-click confirmation box) to avoid double-counting.

## Notes for developers

The simulation lives in the `ENGINE_SRC` string in the Epic script — a set of pure functions
(`mulberry32`, `sampleSize`, `runTrial`, `simulate`, `oneRun`) with no DOM access. It is the single
source of truth: the same string is compiled both in the main thread (for the "show one run"
stepper) and inside the Web Worker (for the bulk simulation). Because the engine is pure, it can be
copied into a `.mjs` file and unit-tested with Vitest/Node if the project ever adopts a build step;
during development it was validated against the spec's hand-calc (total scope 21–35 stories) by
running `ENGINE_SRC` directly in JavaScriptCore.
