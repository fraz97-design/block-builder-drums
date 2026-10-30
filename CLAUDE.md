# Brief for Claude Code: Rudiment Ritual in Block Builder Drums

Read this whole file first. Then confirm the plan with Marcelo one phase at a time before building. Keep replies short and punchy. Never use em dashes, use commas.

## Links
- Repo (public): https://github.com/fraz97-design/block-builder-drums
- Live app (GitHub Pages, main branch, root): https://fraz97-design.github.io/block-builder-drums/
- Source of truth: `index.html` (one file, HTML + CSS + JS, no build step, no network calls)
- Source PDF: `rudiment-ritual.pdf` (12 pages, Marcelo's own warm-up, attach it to the session)
- Version tag lives in the `<h1>` as a small label (currently `v8`). Bump it on every deploy so Marcelo can confirm his iPad loaded the new build.

## Deploy
Edit `index.html`, commit to `main`, GitHub Pages redeploys in about a minute. iPad caches hard: tell him to swipe the app closed and reopen it, and check the version label.

## Who and what
Marcelo is a pro drummer. He uses this on an iPad (landscape and upright) via Add to Home Screen. Phone-first origin. Wants it fast, clear, big letters, no clutter.
Hard rules from him:
- Keep existing routines working. Do not regress anything below.
- Clear separation between training **hands** and training **hands and feet (coordination)**.
- Audio must play with the iPad silent switch on (already handled: `navigator.audioSession.type = "playback"` plus a silent looping `<audio>` in `ensureCtx()`). Do not remove it.
- Full screen: no auto-scroll, everything visible at once (already built via `fitFocus()`). Reuse it.
- Playhead highlight is magenta (`--now`), never a limb color. Limb colors: R blue, L orange, K green.
- Rudiment names are public domain. Build patterns from the stickings, do not embed the PDF images.

## Existing architecture (index.html)
- Time: `TPQ = 2520` ticks per quarter note (divisible by 1 to 7). Bar length from `METERS[m]` via `barT(m)`, beat via `beatT(m)`.
- State: `S` (current phrase: `cell` string of R/L/K, `groups`, `sub`, `m`, `an` accents, `ab` accent beats, `custom`), `PS` (playback prefs: bpm, click, phrase, voice volumes, `hs` hand sound, `ks` kick sound, `cs` click sound, `split`), `A` (audio: `ctx`, gains `cg` click, `pg` phrase, `vg` voice, `playing`, scheduler fields), `RT` (routine: `on`, `r`, `items`, `i`, `subO`, `mO`).
- Audio: Web Audio, all sounds synthesized. `HANDS`, `KICKS`, `CLICKS` tables each with `.play(t, ...)`. `hit(ch, t, acc)` plays one note. `tick()` is the lookahead scheduler (setInterval 25 ms, 120 ms window). `barStart(t)` handles bar boundaries and routine auto-advance.
- Playback model: one repeating "cell" (string of limbs) laid on a uniform subdivision grid (`SUBS`: q, qt, e, et, s, s5, s6, s7, beat). That is the main limitation for rudiments.
- Notation: `gridHTML(pre, span)` and `drawGrid()` render SVG staff rows, with current phrase (`.curgrid`) and next phrase (`.nextwrap`), highlight via `hl(k)`.
- Routines: `ROUTINES` array of `{cat, color, title, why, bars, tip, make(), mappable?, fixed?}`. `make()` returns `[{s:"RLRL RLRL", sub, m}]`. `startRoutine`, `applyItem`, `showRoutine`, `renderRoutines`. Limb remap (R L / L R / K L / R K) via `RMAPS` and `mapS()`.
- Tabs: Routines, Practice, Sounds, Saved, Checklist (`TABS`, `setTab`). Persistence via `load/store` (localStorage, always try/catch).
- Full screen: `setFocus`, `fitFocus`, side-by-side current/next in landscape.

## What to build (phases, confirm each with Marcelo before starting)

### Phase 1: engine
1. **Event-based scores.** Add a second playback path next to the grid path. A rudiment version is a list of timed events, not a cell on a grid:
```js
{ id:"three-stroke-ruff", name:"Three-stroke Ruff", section:"Drags and Ruffs",
  versions:[{ label:"Standard", meter:"4/4", bars:1, repeat:"as written",
    events:[ {t:0,    h:"R", grace:["L","L"], acc:false, buzz:false, dur:2520},
             {t:2520, h:"R", grace:["L","L"], dur:2520}, ... ] }] }
```
`t` and `dur` are ticks inside the bar (TPQ = 2520). Build a scheduler path that walks events per bar, reuses `hit()` and `A.pg`, and keeps the existing grid path untouched.
2. **Grace notes.** Quiet notes just before the main hit. Flam = 1 grace, drag/ruff = 2. Offset in milliseconds (about 30 ms flam, 45 and 22 ms for two), capped relative to tempo, volume about 0.4. Same pitch family as the main hit.
3. **Accents.** Louder hits (existing `acc` logic: 1 vs 0.45), driven by per-event `acc`.
4. **Buzz stroke.** Sustained noise burst across the event `dur` for roll strokes (new HANDS-style voice, snare-like).
5. **Foot layer on its own gain node (`A.fg`).** Loops every bar, never resets between rudiments, locked to the same tick clock and BPM. Own on/off, volume, kick sound, hi-hat foot sound (new short "chick" sound, highpass noise). Default pattern from page 1 of the PDF (4/4, one bar of 10080 ticks):
```js
FOOT_DEFAULT = [ {t:0, f:"K"}, {t:2520, f:"HH"}, {t:3780, f:"K"},
                 {t:5040, f:"K"}, {t:7560, f:"HH"}, {t:8820, f:"K"} ];
```
Read: kick on 1, hi-hat foot on 2 then kick on the "and" of 2, kick on 3, hi-hat foot on 4 then kick on the "and" of 4. Marcelo confirmed this reading ("quarter, eighth hi-hat foot, eighth kick, loops"). The PDF only writes it on the first line ("B.D. & H.H. simile"), so the layer must run under the whole Ritual and be available under any single rudiment. Pattern must be swappable later (kick on 1 and 3, four on the floor, custom).

### Phase 2: notation drawing
6. Draw in the existing SVG staff: grace notes (small, slashed for flam, with stem), accent marks, buzz slashes on stems, mixed beaming and rests inside one bar, repeat and simile marks, tuplet brackets (exists). Keep current note highlight (`.nh.on`, magenta).

### Phase 3: screens
7. **Top-level switch: Hands | Coordination.** Hands: Ritual, Library, Stick control course, existing hand routines (Single-beat combos, Triplet mixes, Roll prep). Coordination: Kick placement, Linear flow, Odd meters, Subdivision ladder, odd groupings, limb remap, and the foot layer controls.
8. **Ritual.** Playlist of every entry in PDF order. One Play button, auto-advance between rudiments, skip back and forward, jump to section, loop one rudiment. Length per rudiment: mix mode. Default "as written" (use notated repeats and bars from the page, including simile marks), with a setting for fixed bars. Shows current rudiment name large and the next one lit up on the last bar before the switch (reuse the `.nextwrap` `.soon` style). Foot layer plays continuously through all of it so he hears how each rudiment interacts with the kick pattern. Full screen supported.
9. **Library.** One entry per rudiment name, versions inside (repeats of the same name appear in several places on the PDF: Three-stroke Ruff, Four-stroke Ruff, Five-stroke Roll, Six-stroke Roll, Seven-stroke Roll, etc., as different versions: Standard, Chopbuilder, 16ths, triplets). Grouped by section. Per version: checklist tick and best clean BPM, saved in localStorage. The Ritual plays each version where it appears on the page, in order.
10. Tempo ladder option (+4 BPM per pass), optional.

### Phase 4: data entry from the PDF
Transcribe section by section with a side-by-side check against a crop of Marcelo's page. Order: Three-stroke Ruff (first working piece, with the foot layer), then Flams, then Drags, Rolls, Paradiddles, Swiss, Innovations, Chopbuilders. Sections in the PDF, with page numbers:
- p1: Three-stroke Ruff, Single Drag, Double Drag, Single Ratamacue, Double Ratamacue, Triple Ratamacue
- p2: Drag Paradiddle No. 1 and No. 2, Lesson 25, Flam, Flam Tap, Flam Accent
- p3: Flamacue, Flam Paradiddle, Flamadiddlediddle, Five-, Seven-, Nine-stroke Roll
- p4: Ten-, Six-, Eleven-, Seven-, Thirteen-, Fifteen-stroke Roll, Paradiddle
- p5: Double Paradiddle, Long Roll, Single-stroke Roll, Swiss Rudiments (Tap Flam, Pata Fla Fla)
- p6: Swiss Army Triplets, Berger 25, Dragadiddle, Flam Roll (Five-stroke), Nine-stroke, Single Windmill and Reverse
- p7: Double Windmill, Three-stroke Ruff Paradiddle, Reverse Flam Accent No. 1 and No. 2
- p8: Single, Double, Triple Ratamaflam, Drag Flam, Flam Double Paradiddle, Innovations: Six-stroke Roll
- p9: Paradiddlediddle, Four-stroke Roll, Tripletdiddle, Triplet Paradiddle, Triplet Flamadiddle
- p10: Drag Triplet, Triplet Flamadiddlediddle (and Variation), Chopbuilders: Three-stroke Ruff, Four-stroke Ruff, Five-stroke Roll, Six-stroke Roll
- p11: Chopbuilders continued: Seven-stroke Roll, Eight-stroke Roll, Flam Dada, Dada Flam, Flamflamdiddle, Para Flam Flam, Flamaflamadiddle
- p12: FlamaFlamaFlamFlam, FlamaFlamFlamdiddle, FlamaFlamFlamFlamFlam, Mamadada, Gary's Flams, Arvin's Diddles

Hard-to-read spots to zoom and double check with Marcelo (render at 300 dpi and crop per line): Ratamaflam lines, Drag Flam, Tripletdiddle, Four-stroke Roll variations, buzz slashes on rolls, ties, tiny grace-note stickings.

## Working agreement
- Confirm each phase's plan with Marcelo before building. Show a short plan, then build.
- After each change: run a headless Playwright test (load `index.html`, start a routine, press Play, check no `pageerror`, check `A.ctx.state === "running"`), take a screenshot at iPad sizes (1024x768 and 768x1024), look at it, then deploy.
- Never break: existing routines, limb remap, silent-switch audio fix, full-screen fit, version label.
- Test with real data per phase. Do not claim iPad behavior is verified unless it was tested on an iPad.
