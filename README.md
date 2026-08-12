# Adairway Golf

A single-file, no-backend golf tracker: rounds, stats, practice log,
bag/equipment, course notes, and a journal — with real handicap math, live
GPS distance, and course data pulled from a free public API. Runs as one
static HTML file with Firebase for account sync.

## Setup on GitHub Pages (free)

1. Create a repo on GitHub.
2. Upload `index.html` to the repo root (drag-and-drop on github.com, or via
   `git add / commit / push`).
3. Repo → **Settings → Pages** → Source: "Deploy from a branch", branch
   `main`, folder `/ (root)`. Save.
4. GitHub gives you a URL like `https://<username>.github.io/<repo>/`
   (takes a minute or two to go live the first time).
5. Open it on your phone and use **"Add to Home Screen"** so it launches
   like an app. The page has PWA meta tags (theme color, app title, status
   bar style) so it looks the part.

## Account sync (Firebase)

Data syncs across every device via your `adairway-golf` Firebase project.
One-time setup in the [Firebase Console](https://console.firebase.google.com):

1. **Enable sign-in providers** — Authentication → Sign-in method → enable
   **Google** and **Email/Password** (the app offers both on the sign-in
   screen — whichever you prefer).
2. **Authorize your domain** — Authentication → Settings → Authorized
   domains → add `<your-username>.github.io`. Skip this and sign-in fails
   with an "unauthorized domain" error.
3. **Create the Firestore database** — Build → Firestore Database → Create.
4. **Set security rules** so only you can touch your own data — Firestore →
   Rules:
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /users/{userId}/store/{docId} {
         allow read, write: if request.auth != null && request.auth.uid == userId;
       }
       match /users/{userId}/meta/{docId} {
         allow read, write: if request.auth != null && request.auth.uid == userId;
       }
     }
   }
   ```
   (The `meta` collection holds a one-time flag for first-run bag seeding.)

Every device signed into the same account sees the same data — rounds, bag,
courses, everything.

## How the tabs work

Every tab opens showing its history/list first — Rounds, Practice, Bag,
Courses, and Notes all tuck their entry form behind a "+ New round" /
"+ Add a club" / etc. toggle, so you're not scrolling past a big form just
to check past data. Tap the toggle (or edit an existing item) to expand it;
it collapses itself again after you save, cancel, or update. Play's
course-setup card works the same way but starts open, since setting up is
the natural first step there — collapse it manually once you're set for
the round.

**Play** — one hole at a time, big tap targets, built for actually being on
the course. A compact insight strip up top shows your handicap index, last-5
round average, and a one-line "focus" (once you've logged 3+ rounds).
Quick-load a saved course, pick tees (if the course has multiple sets on
file), track score/putts/chips, fairway & GIR with optional miss direction
(left/right off the tee, short/long/left/right on approach), bunker, penalty
stroke, and GPS distance to the green — with a club suggestion pulled from
your Bag's stored distances, adjusted for the course's elevation if set.

**Rounds** — the full 18-hole scorecard for entering a round after the fact,
or editing one you already logged. Same stat rows as Play (par, yards, score,
putts, chips, FIR, GIR, miss detail, bunker, penalty), plus a Front 9 / Back
9 / Full 18 selector so a 9-hole round doesn't skew your stats.

**Stats** — year filter, an auto-generated "What the numbers say" insights
card (par-type strengths/leaks, 3-putt rate, penalty frequency, recent
trend — gated behind at least 3 rounds so it doesn't overstate confidence
on thin data), handicap index (real USGA-style differential math), rolling
5-round score/putts trends with gear-change and practice-session markers,
FIR%/GIR% trends, scoring by par-type, FIR/GIR-vs-score scatter plots, and a
miss-tendency breakdown (fairway side, approach side, approach depth,
bunker/sand-save proxy) once you've logged enough miss detail. Every chart
has a plain-language caption explaining what the dots/lines/axes mean and
which direction is "good."

**Practice** — session log (focus, duration, notes), editable in place.

**Bag** — clubs with shaft, status, notes, and an average distance — the
distance is what powers the club-suggestion feature in Play. Editable in
place.

**Courses** — the only source of course data in the app (see below). Search
OpenGolfAPI, add manually, edit anything, and open an interactive map to set
green locations for every hole at once.

**Notes** — a free-form journal, editable in place like everything else.

**Backup** — no longer its own tab. Tap **"backup"** next to your account
email in the header to open it as a popup — export everything as one
`.json` file, or restore from one. Still worth doing occasionally even with
account sync — see "Backing up your data" below for why.

## Courses: the single source of truth

There are no built-in course presets — nothing auto-appears, and deleting a
course removes it everywhere, permanently, including from the Rounds/Play
quick-load dropdowns (which are built directly from whatever's actually in
this list). Only courses with 18-hole par data show up in those dropdowns,
since there'd be nothing to load otherwise.

Both ways of adding a course live under one **"+ Add a course"** toggle in
the Courses tab — search up top, manual fields below it.

**Look up a course** searches OpenGolfAPI's free, keyless public database and
fills in par, yardage (per tee color, when published), rating, slope, and
available tee sets. It only ever calls plain read endpoints — no account, no
API key, nothing about you or your rounds is ever sent, just the course name
you type in. It's community-maintained data (think of it like a wiki), so:
- Coverage varies — smaller regional courses may be missing entirely, or
  have partial data (e.g. rating/slope but no hole-by-hole par).
- If the hole pars sum to a different total than the course's published par,
  you'll get an on-screen warning so you can sanity-check it against your
  scorecard before trusting it.
- Deliberately not used: their sign-in flow, API keys, or their "moments"
  data-contribution pipeline. Not needed for anything here.

**Add manually** — name, rating, slope, elevation, and two optional fields
for **Par** and **Yardage** as 18 comma-separated numbers
(e.g. `4,4,3,5,4,4,4,3,5,4,4,3,5,4,4,4,3,5`) — so a course you type in by
hand can show up in the dropdowns too, same as a looked-up one. Re-entering a
name you've already saved updates that course instead of duplicating it —
handy for retroactively adding par to an old entry, or refreshing stale data.

**Elevation** (ft) feeds the altitude-adjusted "plays like" distance and club
suggestion in Play — a handful of well-known courses have elevation
pre-filled (only if you haven't set one yourself), everything else you enter
once and it's remembered.

## GPS: two ways to build your own green-location database

OpenGolfAPI's green-boundary data is a sparser layer than "the course
exists," so this app treats it as one option, not a dependency:

1. **Map editor** — "map greens" on any course in the Courses tab opens an
   interactive map. It defaults to **satellite imagery** (Esri World
   Imagery — free, no account needed) specifically so you can actually see
   the green shapes, with a layer switcher in the top-right to flip back to
   a street map if you'd rather. Click a hole number, then click its green
   on the map. Do all 18 in a couple of minutes from your couch, or fine-tune
   one hole after a round.
2. **On-course single save** — in Play, "Save my current spot as this hole's
   green" captures your live position and saves it for that hole, once,
   permanently.
3. **Aerial view, mid-round** — "🛰️ Aerial view" in Play opens a satellite
   view centered on your live position, with a 📍 marker for you and a 🚩 for
   the saved green (if any), plus the live distance between them. If a hole's
   green isn't set yet, or looks off, tap anywhere on the image to drop a new
   flag and save it right there — handy for the exact situation that prompted
   this: a couple of courses where the green just isn't visible/mapped well
   enough any other way.

**"Get distance"** checks in this order: your own saved location first
(works offline, zero network calls), then OpenGolfAPI if the course was
looked up and has coverage, then a plain "no data yet" message with a nudge
toward the two options above. Works for **any** course — you don't need an
OpenGolfAPI lookup at all for GPS to function, since you can map the greens
yourself.

*Why Esri instead of actual Google Maps:* Google's satellite tiles aren't
usable for free without their Maps API (a Google Cloud account, billing
enabled, and an API key) — pulling their tiles directly without that isn't
allowed. Esri World Imagery is a genuinely free, no-signup satellite basemap
that solves the same "I can't see the green" problem without that setup.

## Handicap index & course handicap

- **Handicap Index** (Stats tab): `(score − course rating) × 113 / slope`
  per round, best differentials from your last 20 rounds, averaged with the
  standard 96% shave — the real USGA formula. Only counts rounds at courses
  with rating & slope on file. 9-hole rounds are scaled to an 18-hole-
  equivalent differential rather than skewing the index.
- **Course Handicap** (shown live in Rounds/Play once you've selected a
  course and tees): `Index × (Slope / 113) + (Rating − Par)`, using whichever
  tee set you've picked. Tells you your target score for *this* course
  before you even start.

## A few structural notes worth knowing

- **9-hole rounds**: the Front 9 / Back 9 / Full 18 selector actually
  excludes the unplayed holes from every stat and the handicap calc — this
  was a real bug early on (a 9-hole round was being compared against a full
  18-hole par/rating, making stats look artificially great) and is fixed.
- **One-time seeding**: your account gets the Ping i15 starter bag exactly
  once, on first-ever sign-in — never again, even if you later delete every
  club. An empty list stays empty; delete really means delete.
- **Multi-tee support**: courses looked up via OpenGolfAPI store every
  published tee set (name, color, rating, slope, par, yardage). Picking
  different tees in Rounds/Play updates yardages and the course handicap
  live.
- Once a course is selected (quick-load dropdown or typed in), its name now
  shows as a clean header — right above the hole number in Play, and right
  above the scorecard in Rounds — so it's always obvious which course you're
  looking at.
- **Fixed: an in-progress round used to disappear.** Play mode only kept the
  current hole-by-hole state in memory — closing the tab, letting your phone
  put it to sleep, or just stepping away for a while would silently wipe it,
  even though nothing was actually saved yet. It now autosaves a local draft
  to this browser after every change (score, putts, hole navigation, course
  info) and resumes it automatically next time you open Play, with a small
  banner confirming it picked up where you left off (and a "discard and
  start over" option if you'd rather not). This draft is local to the
  device you're playing on — not synced to your account — and clears itself
  automatically once you tap "Finish & save round."

- **Fixed: double-tap-to-zoom on quick button taps.** Mashing the score/putts
  +/- buttons (or any button) quickly used to trigger the phone's double-tap
  zoom gesture. Buttons, dots, and toggles now opt out of that specifically —
  pinch-to-zoom on the page itself still works fine, this only stops the
  accidental zoom while tapping controls.
- **Penalty stroke tag**: a chip next to Bunker in Play, and a matching row
  on the Rounds scorecard — tags whether a hole had a penalty stroke (OB,
  water, lost ball, etc). It's a yes/no tag like Bunker, not an exact stroke
  count, so "Penalties/Rd" in Stats means holes-with-a-penalty per round, not
  total penalty strokes. Feeds into the Stats insights below.
- **"What the numbers say"** (Stats) and the **insight strip** (top of Play)
  both pull from the same underlying logic: par-type strengths/leaks (needs
  6+ holes of a given par type to speak up), 3-putt rate (needs 18+ holes
  with putts logged), penalty frequency, and a last-5-vs-prior trend (needs
  8+ rounds). Deliberately conservative about small samples — with only a
  few rounds logged, most of these stay quiet rather than overstate a
  pattern from 2-3 data points. The Play strip only shows once you've got 3+
  rounds logged at all, and always reflects your all-time data regardless of
  whatever year filter is selected on the Stats tab.
- **1st putt distance**: three buckets (<5 ft, 5-15 ft, 15+ ft), tagged next
  to the Putts stepper in Play and as a row on the Rounds scorecard. This
  exists to separate two things that look identical in a plain "2 putts"
  entry but aren't: hitting the green in regulation and 2-putting from 25
  feet (normal — most golfers don't make those) versus missing the green,
  chipping it close, and still needing 2 putts (a short game shot that
  didn't quite finish the job). Stats shows 1-putt% and average putts per
  distance bucket, plus — once you've logged 5+ missed greens with a
  distance tagged — what percentage of your recovery shots after a miss
  actually finished inside 5 feet, and how often you converted those into a
  save. Entirely optional per hole; leave it blank on any hole and it's just
  excluded from these stats rather than counted as anything.

## The Claude-artifact companion version

Earlier in this project there was a parallel build (`golf-tracker.html`)
meant to run as a Claude.ai artifact using Claude's own storage instead of
Firebase. Given how far this has grown — a Firebase sign-in gate blocking the
whole app, a Leaflet map pulling map tiles, and live OpenGolfAPI calls — that
version isn't really viable to keep maintaining in parallel. Claude's
artifact sandbox can't make the external network calls or (likely) complete
an auth popup that this app now depends on for basically everything. Worth
formally retiring that copy and treating this `index.html` as the one real
version, unless you specifically want a network-free fallback — happy to
build a deliberately-scoped-down version for that if useful, but it'd need
to be a real fork, not a silent parity copy.

## Backing up your data

Firebase sync solves "I lost my phone" — it doesn't solve "I fat-fingered a
delete," a Firestore security rule getting misconfigured, or losing access
to the account itself someday. A backup file is your only copy that lives
completely outside the live database, which is why it's still around even
though storage moved off local/per-device. It just doesn't need its own tab
anymore — tap **"backup"** next to your account email in the header:
- **Export** — downloads everything as one `.json` file. Save it to Drive,
  Files, email, wherever.
- **Restore** — replaces everything currently loaded with a previously
  exported file. Also how you'd move data if you ever left Firebase.

Worth doing every so often, especially after a good round.
