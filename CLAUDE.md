# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"Les Loups Garous de Corbelin" — a family Werewolf game management app. Single-page app used live during games at Corbelin. Two audiences share one URL: the **Narrateur** (GM, PIN-protected) and the **Joueurs** (players).

## No build process

The app is a single `index.html` file. There is no bundler, no dependencies, no `package.json`. Edit `index.html` directly and push — Netlify auto-deploys from `main`.

To preview locally: just open `index.html` in a browser, or run `python3 -m http.server` from the repo root.

## Architecture

Everything lives in `index.html` (~1535 lines, but 6MB on disk due to embedded base64 player photos and background audio).

**Global state** — single object `G` (line ~537):
```
G = { players, quests, tour, events, votes, resumes,
      victimId, sorcAction, sorcKillId, potV, potM, cupidPair,
      sorcUnlocked, sorcPlayerId, voyUnlocked, voyPlayerId, locked }
```

**Two views**, toggled by `activateNarrateur()` / `switchToJoueurs()`:
- `#nav-narrateur` — Setup, night phase (victim, sorcière), conseil (votes, elimination), timeline, stats/badges
- `#nav-joueurs` — Résumés, character bios, vote history, badges (read-only)

**Tabs within each view** are shown/hidden via `showMTab(n)` / `showJTab(n)`.

## State persistence

`saveState()` / `loadState()` / `resetState()` handle two layers:
1. **JSONBin** (primary) — stored at `JSONBIN_URL` using `JSONBIN_KEY` hardcoded in the file. Enables cross-device sync (narrateur on one phone, joueurs on another).
2. **localStorage** (`corbelin_v4` key) — fallback if JSONBin is unreachable.

The game only loads saved state when `locked === true` (i.e., setup has been validated).

## AI integration

`callClaude(prompt)` (line ~1233) POSTs to a Cloudflare Worker proxy at `https://claude-proxy.alexandrelemetais.workers.dev` using `claude-sonnet-4-5`. Used for:
- `generateResume(tour)` — narrative summary after each round
- Rumeur generation for the voyante

`netlify/functions/claude.js` is a Netlify Function that proxies `/api/claude` → Anthropic API directly using `process.env.ANTHROPIC_API_KEY`. It is **not currently called** by the frontend (the Cloudflare proxy is used instead), but can serve as a fallback.

## Key constants

- `PRENOMS` — hardcoded list of 13 family player names
- `PLAYER_PHOTOS` — base64-encoded portrait images keyed by name
- `PLAYER_BIOS` — character backstory strings keyed by name
- `RLABELS` — display labels for roles: `loup`, `villageois`, `cupidon`, `amoureux`, `voyante`, `sorciere`

## Déploiement

Les commits sont poussés via **GitHub Desktop** sur le repo GitHub. **Cloudflare Pages** déploie automatiquement à chaque commit sur `main`.

**Ne jamais committer ni pousser sans demander confirmation explicite à l'utilisateur au préalable.**

## Game flow (Narrateur tabs)

1. **Setup** — assign roles, toggle quests (voyante, sorcière unlock)
2. **Nuit** — pick victim, sorcière action (save/kill)
3. **Conseil** — record votes per player, pick elimination
4. **Journal** — timeline of all events; auto-generates AI résumé on `cloreNuit()`
5. **Stats/Badges** — vote matrix, computed badges, scoreboard

The `tour` counter increments; `G.events` and `G.votes` accumulate across the full game.
