# Adairway — Golf Tracker

A single-file, no-backend golf tracker: rounds, stats, practice log, bag/equipment,
course notes, and a journal. Works entirely in the browser — no server, no signup.

## Setup on GitHub Pages (free)

1. Create a new repo on GitHub (public or private both work with Pages on a
   paid plan; public is simplest if you're on the free tier).
2. Upload `index.html` (rename `golf-tracker.html` to `index.html`) to the
   repo — either drag-and-drop on github.com, or:
   ```
   git init
   git add index.html
   git commit -m "Add golf tracker"
   git branch -M main
   git remote add origin https://github.com/<your-username>/<repo-name>.git
   git push -u origin main
   ```
3. In the repo, go to **Settings → Pages**.
4. Under "Build and deployment", set **Source** to "Deploy from a branch",
   branch `main`, folder `/ (root)`. Save.
5. GitHub gives you a URL like:
   `https://<your-username>.github.io/<repo-name>/`
   (takes a minute or two to go live the first time.)
6. Open that URL on your phone, then use your browser's **"Add to Home
   Screen"** option so it launches like an app.

## Account sync (Firebase)

This version now syncs across every device via your own Firebase project,
instead of being stuck to one browser. A few one-time steps in the
[Firebase Console](https://console.firebase.google.com) for your `adairway-golf`
project to finish wiring this up:

1. **Enable sign-in providers** — Build → Authentication → Sign-in method →
   enable **Google**, and **Email/Password** if you want that option too (the
   app offers both — Google or email+password, whichever you prefer).
2. **Add your GitHub Pages domain** — Authentication → Settings → Authorized
   domains → add `<your-username>.github.io`. Without this, sign-in will fail
   with an "unauthorized domain" error.
3. **Create the Firestore database** — Build → Firestore Database → Create
   database (any region close to you is fine; production mode is fine since
   we set explicit rules below).
4. **Set security rules** so only you can read/write your own data — Firestore
   Database → Rules, paste this in, then Publish:
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /users/{userId}/store/{docId} {
         allow read, write: if request.auth != null && request.auth.uid == userId;
       }
     }
   }
   ```

Once that's done, re-upload `index.html`, open it, and sign in with Google.
Every device you sign into with that same account sees the same data.

**Migrating your existing rounds**: your old data is sitting in this browser's
localStorage, separate from the new Firestore-backed storage — signing in
starts you fresh. Before you re-upload the new file, open the *current* live
version, go to **Backup → Download backup (.json)**, then after signing into
the new version, use **Backup → Restore from a backup** to bring it all over.

The Claude-artifact version is unaffected by any of this — it still uses your
Claude account's own storage and works the same as before.


## Backing up your data

Even with Firestore syncing across devices, it's still worth having an
independent copy — the **Backup** tab lets you:
- **Export** — downloads everything (rounds, practice log, bag, courses,
  journal) as one `.json` file. Save that file to Google Drive, Files, email,
  wherever — it's your safety net if anything ever gets deleted or corrupted,
  or if you ever want to move off Firebase entirely.
- **Restore** — pick a previously exported `.json` file to replace everything
  currently loaded with it. Also how you migrate data between this GitHub
  version and the version in Claude (which uses separate storage).

Worth doing every so often, especially after a good round.

## What's new: analytics

The Stats tab now includes:

- **Handicap index** — a real USGA-style differential calc `(score − course rating) × 113 / slope`, averaged over your best differentials from the last 20 rounds (with the standard 96% shave). This only works for rounds at courses that have a rating and slope set in the Courses tab — your 7 regulars are pre-loaded with real numbers, but double-check them against your actual tee box (I used approximate "white tee" figures; exact values vary by tee).
- **Rolling 5-round averages** on the score and putts charts, layered over the raw per-round dots, so you can see the trend without the noise of any one bad day.
- **Gear and practice markers** — small dots along the score chart marking when you added/retired a club or logged a practice session (hover for details).
- **Scoring by par type** — average to par on 3s/4s/5s, plus a birdie/par/bogey/double+ distribution bar.
- **Correlation scatters** — FIR% and GIR% plotted against score-to-par, so you can see which one actually tracks with better rounds for you.

Bag entries also now have an optional **distance** field (yds) — worth filling in over time as you get real yardages from the range or GPS.

## Course lookup + GPS (GitHub version only)

The Courses tab now has a **Look up a course** search box backed by OpenGolfAPI's
free, keyless course database (16,000+ US courses). Search a name, pick a match,
and it fills in par (all 18 holes), rating, and slope for you — review before saving.

**What this does and doesn't do, on purpose:**
- Only ever calls the plain read endpoints (course search, course detail, green
  geometry). No account, no API key, no sign-in.
- **Never sends anything about you** — not your rounds, not your email, not your
  location — to OpenGolfAPI or anywhere else. The only data that leaves your
  browser is the course name you type into the search box.
- GPS distance-to-pin works the same way: your phone gives your live position
  directly to this page (never transmitted anywhere), a green coordinate is
  looked up once for the hole you're on, and the distance is calculated locally
  in the browser. Tap **"📍 Get distance"** in the Play tab on a course you've
  looked up this way.
- Deliberately skipped: their sign-in flow, API keys, and their "moments/shots"
  data-contribution pipeline — none of that is needed for what this app does,
  and it's not something to wire in without understanding it fully.

Coverage varies by course since the data is community-maintained (like a wiki) —
if a course or hole doesn't have data yet, you'll get a clear message and can
still enter par/rating/slope manually.

## Editing rounds and clubs

Both the Rounds history and the Bag list now have an **edit** link alongside
delete. For a round, it loads the full scorecard back into the form (including
holes-played) so you can fix a typo'd score or add putts you forgot — hit
"Update round" instead of creating a duplicate. Same idea for clubs — edit
pulls a club back into the Add-a-club form, tweak anything, "Update club" saves
it in place. "Cancel edit" on either form backs out without saving.


This only works in the GitHub-hosted version — the Claude-artifact version can't
reach external APIs, so the OpenGolfAPI lookup and course-search will just show
a "couldn't reach" message there. GPS itself works in both (see below).

## GPS fallback: save your own green locations

OpenGolfAPI's green-boundary coverage is uneven — plenty of courses (especially
smaller regional ones) just don't have that data published yet. Rather than
depend on it, GPS now builds its own database as you play:

- **"Save my current spot as this hole's green"** (Play tab) — stand near or on
  the green, tap it once, and that exact spot is saved for that course and
  hole, forever. No internet needed after that — it's stored the same way as
  everything else.
- **"Get distance"** checks in this order: your own saved spot first (if you
  have one), then OpenGolfAPI (if the course was looked up and has coverage),
  then tells you plainly if neither exists yet — with a nudge to just save one.
- This works for **any course**, including the 7 pre-loaded regulars — you
  don't need to look a course up via OpenGolfAPI at all for this to work.
- The Courses tab now shows how many of a course's 18 greens you've personally
  saved, so you can see coverage build up round by round.

Practically: the first time at a course, GPS might say "no data yet" on most
holes. Tap "save" once per hole as you play it, and every round after that,
it's instant and doesn't touch the network at all.

## Fixed: deleted defaults coming back

The app used to reseed the 7 default courses (and the Ping i15 bag) any time
that list was *empty* — meant for "first time ever opening this," but it
couldn't tell that apart from "you just deleted everything on purpose." So
deleting all your courses to clean up duplicates, then refreshing, silently
brought the old no-yardage defaults right back.

Fixed with a one-time flag: seeding now only ever happens once, the very first
time your account is used. After that, an empty list stays empty no matter how
many times you refresh — delete really means delete.

**This matters for tomorrow's migration**: when you sign into the new Firebase
version for the first time, it's a brand-new account with nothing in it yet —
that first login is when the one-time seeding happens (Ping i15 bag only —
see below, courses are no longer auto-seeded at all). Restore your backup
*after* that first sign-in, not before, so you don't end up with the seeded
bag sitting alongside a restored one as duplicates.

## Built-in course presets removed entirely

The 7 pre-loaded regulars (and the silent fallback data behind them) are gone.
This was the real root of the dropdown/yardage confusion — even after courses
were deleted from your list, a hardcoded copy of their names and par data was
still baked into the code as a fallback, so they kept reappearing in the
Rounds/Play dropdown no matter what you did in the Courses tab.

Now: **the Courses tab is the only source of truth.** Nothing auto-appears on
first login, nothing falls back to hidden data, and deleting a course removes
it everywhere, permanently — the dropdown is built directly from whatever's
actually in your list.

**One consequence worth knowing**: only courses with 18-hole par data show up
in the Rounds/Play quick-load dropdown, since there'd be nothing to load
otherwise. "Look up a course" fills that in automatically. If you add a course
manually instead, there are now two optional fields — **Par** and **Yardage**,
each 18 comma-separated numbers (e.g. `4,4,3,5,4,4,4,3,5,4,4,3,5,4,4,4,3,5`) —
so a manually-added course can show up in the dropdown too. Re-entering a name
you've already saved updates that course rather than duplicating it, so this
also works for retroactively adding par to a course you added before this
existed.

**To get a clean slate**, once you're signed into the new version: open the
Courses tab and delete anything left over from before (any stale copies with
no yardage set). Then re-add each course either by searching for it in **Look
up a course** (gets you real par, yardage, rating, and slope automatically) or
by typing it in manually. Takes a couple minutes once, and after that the
dropdown will only ever show exactly what you've chosen to keep.


## Hole yardages

The scorecard now has a **Yards** row alongside Par. It auto-fills when you look
up a course via OpenGolfAPI (if that course's data includes yardage). The 7
pre-loaded regulars don't have per-hole yardage baked in yet — easiest fix is to
search for them again in the **Look up a course** tool now that it exists; it'll
pull real numbers automatically.

## Playing 9 holes (fixed a real bug)

There's now a **Holes played** selector (Full 18 / Front 9 / Back 9) on both the
Rounds form and the Play tab. Pick Front or Back and the other 9 holes grey out
so they can't accidentally get counted.

This also fixed a real bug: previously, a 9-hole round's score was being compared
against a full 18-hole par and course rating, which made your average-to-par and
handicap index look artificially, impossibly good. Now:
- Stats (avg to par, FIR%, GIR%, putts) only count holes that actually have a
  score entered.
- The handicap index treats a 9-hole round properly — it estimates a 9-hole
  course rating as half the 18-hole rating, computes the differential, then
  scales it to an 18-hole-equivalent number the way USGA pairs two 9-hole
  rounds together. It's a simplified stand-in (real courses publish separate
  front/back ratings that we don't have), but it's honest about being an
  estimate and won't wreck your index the way the old bug did.




