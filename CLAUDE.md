# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A single-player, browser-based, fully Persian (RTL) educational escape-room game. The player escapes the "mind" of an AI called هاتف (Hatef) by solving five layers that teach how an LLM works: Tokenization → Embedding → Attention → Generation → Prompt Engineering. Ten-minute timer, one key per layer. It is built for live classes of up to ~60 students, each on their own device. Deployed on Vercel: every push to `master` auto-deploys.

## Commands

```bash
npm install
npm run dev      # http://localhost:3000 (falls back to 3001+ if busy)
npm run build
npm start
npx tsc --noEmit # type-check
```

There is no test suite, and ESLint is not installed (`npm run lint` / `next lint` will prompt to set it up). The `npm audit` vulnerability warnings are known and come from build tooling, so they don't need fixing. Requires Node >= 18.18.

## Architecture

Next.js 15 App Router, TypeScript (strict, `@/*` path alias to repo root), Tailwind, Framer Motion, Zustand, Tone.js.

- **All game content lives in [lib/puzzles.ts](lib/puzzles.ts)**: Persian narration, per-layer metadata (`layers[id]`: title, intro/outro lines, 3 hints, base score), each puzzle's data and answers (lock code `7391`, password `نگار`, pronoun→referent map, generation steps, acrostic target `رها`), and timing constants (`TOTAL_TIME_SECONDS`, `HINT_PENALTY_SECONDS`). Change content here, not in the components.
- **State: [lib/store.ts](lib/store.ts)** is one Zustand store (`useGame`), persisted to localStorage under the key `hatef-escape-room` so a page refresh resumes the game. The timer is derived, not ticked: `remainingSeconds()` = total − (now − `startTimestamp`) − `penaltySeconds`. Each hint adds a time penalty. `solveLayer(5)` sets `phase: "win"` and freezes `finalRemaining`.
- **Flow:** `/` ([app/page.tsx](app/page.tsx), Intro screen) calls `startGame()` and navigates to `/game`. [app/game/page.tsx](app/game/page.tsx) waits for mount to avoid hydration mismatch with the persisted state, redirects to `/` if no game has started, and renders the Win or Lose screen or the current layer.
- **Layer pattern:** each puzzle in `components/puzzles/LayerN*.tsx` is wrapped by [components/LayerShell.tsx](components/LayerShell.tsx), which runs the stages intro narration → puzzle → outro. The shell passes `onSolved` to the puzzle as a render-prop. A puzzle only calls `onSolved()`; the shell plays the unlock SFX, shows the outro, and then calls `solveLayer` to advance. To add or reorder a layer, update `LayerId`, `layers`, the switch in `app/game/page.tsx`, and the `id >= 5` win check in the store.
- **Layer 5 / LLM backend:** [components/puzzles/Layer5Prompt.tsx](components/puzzles/Layer5Prompt.tsx) POSTs `{prompt}` to [app/api/oracle/route.ts](app/api/oracle/route.ts), which tries three modes in order:
  1. `OPENROUTER_API_KEY`: walks a chain of free OpenRouter models (the first one can be overridden by `OPENROUTER_MODEL`). On a 429 or error it falls through to the next model, which spreads class load across models.
  2. `ANTHROPIC_API_KEY`: Claude via `@anthropic-ai/sdk`.
  3. Offline (no key): `promptCoversRequirements()` regex-checks the player's *prompt* and returns the canned `fallbackPoem`.

  The live modes judge the model's *output* with `checkAcrostic()` in [lib/acrostic.ts](lib/acrostic.ts): the first letter of each line, after stripping diacritics and normalizing alef variants, must spell `رها`. The game must stay completable with no API key.
- **Assets degrade gracefully:** [components/SceneBackground.tsx](components/SceneBackground.tsx) loads `public/images/<name>.png` and falls back to a CSS gradient if the image is missing. [lib/audio.ts](lib/audio.ts) plays `public/audio/ambient.mp3` if it exists; otherwise it generates ambient music with Tone.js (lazy-imported, started only after user interaction). Its tempo rises via `updateTension()` as time runs out.
- **Scoring/formatting:** [lib/scoring.ts](lib/scoring.ts) computes the score as base + 2 per remaining second − hint penalty. Use `toFa()` / `formatTime()` for any displayed numbers (Persian digits).

## Conventions

- All user-facing text is Persian, and the root layout sets `lang="fa" dir="rtl"` with the Vazirmatn font. Code comments are also written in Persian, so match that.
- Custom Tailwind colors and utilities (`cyanGlow`, `amberGlow`, `ink-900`, `text-glow`, `shadow-glow`, `scanlines`, `grain`) are defined in [tailwind.config.ts](tailwind.config.ts) and [app/globals.css](app/globals.css).
- Env vars go in `.env.local` (copy from `.env.example`). `.env*` is gitignored.
