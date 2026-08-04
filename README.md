# Back Nine — Golf Tracker

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

## Important: where your data lives

This version saves to your browser's **localStorage** — it's tied to that
specific browser on that specific device. Log a round on your phone, and it
stays on your phone; it won't show up if you open the same link on a laptop.
Clearing your browser's site data/history for this page will also erase it,
so avoid "clear all browsing data" if you want to keep your log.

If you ever want cross-device syncing, that would need a small backend
(e.g. a free tier of Supabase or Firebase) — a bigger step up, but doable
later if this becomes something you use a lot.
