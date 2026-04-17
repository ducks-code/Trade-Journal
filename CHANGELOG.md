# Changelog

All notable changes to the Trade Journal system.
Format follows [Keep a Changelog](https://keepachangelog.com/).

## [Unreleased]

## [0.5.0] — 2026-04-17 — Repo init + Euphoria handling

### Added
- `Config.md` pattern: API key moved out of scripts into a single gitignored config file
- `loadApiKey()` helper in all four scripts (Conviction Check, Exit Check, Get AI Critique, Trade Research)
- Automatic euphoria detection in Pre-Trade Checklist: scans for recent wins (48h window) and open positions at creation time
- Dynamic injection of Euphoria Check section — only appears when triggers are detected, keeping the checklist clean on cold streaks
- Exit Check third prompt: free-text "How are you feeling about this position right now?"
- Euphoria detection in Exit Check system prompt: reads emotional tone from free text, activates heightened scrutiny (moved goalposts, "would I buy this amount at this price" test, round-tripping risk)

### Changed
- Exit Check entries now insert newest-first under `### Exit Checks` (reverse chronological log)
- Exit Check system prompt includes prior exit check context — tracks evolution of reasoning across multiple checks
- `.gitignore` excludes real trade data (`Trades/`, `Pre-Trades/`, `Wins/`, `Losses/`) and `Config.md`

### Security
- Removed all hardcoded API keys from tracked scripts
- If you have prior commits containing a key, revoke it at `console.anthropic.com/settings/keys`

## [0.4.0] — 2026-04 — AI Critique learning framework

### Added
- Trading principles reference block in Get AI Critique system prompt (profit taking, position management, thesis vs hope trades, cognitive biases, emotional control, two-trades-two-questions)
- "Principles Check" section in critique output connects trades back to the learning framework

## [0.3.0] — 2026-04 — Conviction Check + Exit Check

### Added
- Conviction Check script (`Alt + Q`) — stress-tests thesis and override reasons on the checklist without making buy/no-buy recommendation
- Exit Check script (`Alt + X`) — resurfaces trader's own plan and asks whether exit conditions are met, repeatable with timestamp log
- Both scripts explicitly designed to never say buy/sell/hold — questions only

## [0.2.0] — 2026-04 — AI Critique with checklist context

### Added
- Get AI Critique script reads linked pre-trade checklist and passes it to the API alongside the trade note
- Checklist Adherence scoring category replacing generic Process Discipline
- Missing field warnings for `followed_plan`, `emotion_post`, `result`

## [0.1.0] — 2026-03 — Core workflow

### Added
- Pre-Trade Checklist template with live Dataview status banner
- Trade Note template with linked checklist frontmatter
- Create Trade Note script — validates Final Gate before spawning trade note via `app.vault.create` + `processFrontMatter`
- Trading Dashboard with Dataview queries scoped to `result` field
