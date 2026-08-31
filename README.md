# Throughline

A private daily journal that reads what you wrote, names the patterns, and lines
them up against what you actually got done in Habit Casino.

Everything is one file — `index.html`. No build step, no package manager, no
server of mine. Your entries live in your browser; the only outbound calls are
to the AI provider you configure and (if you switch it on) your own Firebase
project.

**Live at https://ramentooth.github.io/ThruLine/**

## Run it locally

```bash
python3 -m http.server 4780 --directory /Users/simonsakata/ThruLine
```

Then open http://localhost:4780. (Also registered as the `throughline` launch
config, so Claude Code can start it directly.)

## Deploy to GitHub Pages

Pages serves `index.html` straight from the repo root on `main`, so shipping a
change is just:

```bash
git add -A && git commit -m "your change" && git push
```

Pages rebuilds on its own within a minute or so. HTTPS is on by default, which
matters here — the microphone (Web Speech dictation) only works on a secure
origin, so the deployed page can do voice input where a plain `http://` host
could not.

## Alternative: deploy to Firebase Hosting

One-time:

```bash
npm install -g firebase-tools && firebase login
```

Put your real project id in `.firebaserc` (replace `REPLACE-WITH-YOUR-FIREBASE-PROJECT-ID`),
then:

```bash
firebase deploy --only hosting
```

That publishes `index.html` at `https://<project-id>.web.app`. On your phone,
open it and use Share → Add to Home Screen; it runs full-screen like an app.

### Hosting it next to Habit Casino

On GitHub Pages every `ramentooth.github.io` project shares one origin, so if
Habit Casino is ever hosted there too, Throughline can read
`localStorage['habitCasino.v1']` directly — Settings will show a "Read from this
browser" button and you never have to export a file. On different domains,
export from Habit Casino (Settings → Export) and import the JSON here.

## Turning on the AI

Settings → AI. Three providers:

| Provider | What you need | Default model |
|---|---|---|
| Anthropic | An API key from console.anthropic.com | `claude-opus-5` |
| OpenAI | An API key | `gpt-4.1` |
| Ollama | Ollama running locally | `qwen3:8b` |

The key is stored under its own localStorage entry (`throughline.ai`), separate
from the journal, so exporting or erasing your journal never touches it. It is
sent only to the provider, straight from your browser.

Anthropic calls go out with `anthropic-dangerous-direct-browser-access: true`
and the server-side refusal fallback enabled — journals go to dark places, and a
classifier declining to read one is a bad experience. If a browser ever rejects
the beta header on preflight, the app retries once without it automatically.

## Optional: sync across devices

Settings → Sync & backup → Firebase sync. Paste the config object from your
Firebase console, sign in with Google, done. Entries merge per-entry by
`updatedAt`, so writing on your phone and then your laptop never silently drops
one of them.

You need two things enabled in the Firebase console:

1. **Authentication** → Sign-in method → Google.
2. **Firestore Database** → create it, then `firebase deploy --only firestore:rules`
   to apply `firestore.rules` (each user can only read and write their own doc).

Sync is off until you set it up; without it the app is entirely local, which is
also a legitimate way to run it — just export a backup now and then.

## What's where in `index.html`

The `<script>` is banner-commented in this order:

```
DATA MODEL · HELPERS · AI · WRITE VIEW · ANALYTICS · CHARTS
ENTRIES VIEW · PATTERNS VIEW · HABIT CASINO · FIREBASE SYNC
SETTINGS · ROUTING & INIT
```

Storage keys:

| Key | Holds |
|---|---|
| `throughline.v1` | entries, reports, prefs — the export payload |
| `throughline.ai` | provider, key, model — never exported |
| `throughline.habits.v1` | the reduced Habit Casino day index |
| `throughline.fb` | your Firebase config (not a secret) |

To syntax-check a JS edit without a test suite:

```bash
awk '/^<script>$/{f=1;next} /^<\/script>$/{f=0} f' index.html > /tmp/tl.js && node --check /tmp/tl.js
```

## How the habit correlation works

Habit Casino's export carries a `history` array of completions
(`{ts, kind, name, category, level, reward}`). Throughline reduces it to one row
per local day, then for each habit compares your average mood on days you did it
against days the tracker was running and you didn't — requiring at least three
days on each side before it will show a row. "Next day" runs the same comparison
against the *following* day's mood, which is the more interesting question.

It is association, not cause. A good day makes habits easier just as readily as
habits make a good day, and the sample sizes are printed next to every row for
exactly that reason.
