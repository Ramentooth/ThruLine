# Throughline

A private daily journal that reads what you wrote, names the patterns, and lines
them up against what you actually got done in Habit Casino.

Everything is one file — `index.html`. No build step, no package manager, no
server of mine. Your entries live in your browser; the only outbound calls are
to the AI provider you configure and (if you switch it on) your own Firebase
project.

**Live at https://thruline-a12e4.web.app** (Firebase Hosting, primary)
**Also at https://ramentooth.github.io/ThruLine/** (GitHub Pages)

Firebase Hosting is the better address of the two: its domains are authorized
for Firebase Auth automatically, so Google sign-in works there with no extra
setup. Any other origin — the Pages URL, a Tailscale name — has to be added by
hand under Authentication → Settings → Authorized domains, or sign-in fails
with `auth/unauthorized-domain`.

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

## Memory

Three layers reach a reflection, in ascending durability:

| Layer | What | Cost |
|---|---|---|
| **Recent** | last 12 days in full — context is mostly local | ~500 tokens |
| **Echoes** | older entries pulled in *only* when today's language matches them (`findEchoes`, local term overlap — no embeddings), plus a one-line-per-month skeleton of everything else | ~450 tokens |
| **Standing** | the profile the model keeps about you, and the lessons you wrote | grows slowly |

The echo layer is why a reflection in November can reach March. Sending the
whole corpus would also fit — five years is only ~81K tokens — but it buries
today under a year of one-liners; this keeps recall without the dilution.

**What it knows about you** (Patterns tab) is a set of notes the model maintains
about how you think, rebuilt from the whole journal and loaded into every
conversation. It *revises* rather than rewrites, is told to say when it changes
its mind, keeps its last 6 versions, and is editable — anything you correct is
what future reflections read.

**Life lessons** are the opposite direction: conclusions you wrote deliberately.
They go into every call with an instruction to quote one back only when the day
genuinely touches it, never to recite the list. "Save a lesson" sits on every
reflection, where you tend to have the thought.

## The thinking animation

Before a reflection, a fast low-effort `conceptPass` returns 4–7 concepts, each
with **the exact phrase in your entry it came from**, plus which past themes it
echoes. Quotes that don't appear verbatim are dropped, so a highlight can never
land on the wrong text.

The animation renders that: a transparent mirror of the textarea (a textarea
can't style its own contents) lights up each quoted span, a bubble rises off it,
remembered themes drift in from the edges, and both converge where the
reflection is about to appear. It clears the moment real prose starts arriving.

Every bubble is something actually sent to the model. It is not a decorative
loader — the model's real chain of thought is never returned by the API, so
animating "its thinking" would be fiction. This animates the *inputs*, which are
real, and it covers latency the concept pass genuinely creates.

## Focus mode

Start typing and the app fades out — header, mood scales, panels, buttons and
nav all go, the sheet loses its border, and the editor fills the screen.

Getting out is deliberately hard to do by accident. Only moving the pointer
within **100px of a screen edge** ends it (`ZEN_EDGE`) — roaming around the
middle is reading your own writing, not asking for the app back. There is no
scroll handler at all, so you can scroll freely while staying in focus. Escape
and the low-contrast "done" chip cover touch and keyboard-only use, it stays off
while dictating so the stop button remains reachable, and leaving the Write tab
always restores the chrome.

The editor auto-grows to fit its content, so the page is the only thing that
scrolls — a textarea scrolling inside a scrolling page is a trap to get stuck
in. In focus mode `sizeZenPad()` adds exactly enough trailing space for the last
line of text to scroll up to the top third of the screen, no further; whatever
the faded panels below already contribute is subtracted, so it lands on the same
mark for a three-line entry and a two-hundred-line one.

Toggle it in Settings → Reflection style.

## Turning on the AI

Settings → AI. Three providers:

| Provider | What you need | Default model |
|---|---|---|
| Anthropic | An API key from console.anthropic.com | `claude-opus-5` |
| OpenAI | An API key | `gpt-4.1` |
| Ollama | Ollama running locally | `qwen3:8b` |

Reflections run as **two calls in parallel**: a streaming prose reply with no
JSON schema (that constraint is what made an earlier version answer in clipped
single sentences), and a small structured tagging pass that feeds the Entries
list and the Patterns charts. Both use adaptive thinking. Settings →
"How much should it write back?" picks Brief / Conversational / Deep, which sets
the target length and the reasoning effort (`medium` / `high` / `xhigh`).

**Talk it through** (the button under the editor) opens a conversation *before
or while* you write, as opposed to the reflection, which comes after. It opens
by saying something real — an observation across recent entries, a guess at what
is going on — rather than firing a question and stopping, and then it is just a
chat. It sees the draft you have so far. Threads save on the entry as `muse`
and are separate from the reflection thread. With no API key the button falls
back to dropping a written prompt into the entry, as it used to.

You can reply to any reflection and keep talking — the entry, its context and
the first reflection are replayed as a real conversation, so a follow-up
continues the thread instead of starting over. Threads are saved with the entry.

The key is stored under its own localStorage entry (`throughline.ai`), separate
from the journal, so exporting or erasing your journal never touches it. It is
sent only to the provider, straight from your browser.

Anthropic calls go out with `anthropic-dangerous-direct-browser-access: true`
and the server-side refusal fallback enabled — journals go to dark places, and a
classifier declining to read one is a bad experience. If a browser ever rejects
the beta header on preflight, the app retries once without it automatically.

## Optional: sync across devices

The config is already baked into `FB_CONFIG` at the top of `index.html`
(project `thruline-a12e4`), so on a new device the only step is Settings →
Sync & backup → Firebase sync → **Sign in with Google**. Those values are public
identifiers by design; the rules and the authorized-domain list are what protect
the data. The paste box under "Use a different Firebase project" overrides the
built-in config on one device only.

Deploy hosting and rules with:

```bash
firebase deploy
``` Entries merge per-entry by
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
| `throughline.v1` | entries (text, reflections, both chat threads), reports, prefs — the export payload |
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
