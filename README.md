# Crypto Trading Journal — Obsidian Setup

A structured trading journal built in Obsidian for crypto traders. It enforces pre-trade discipline, captures post-trade reflection, and surfaces patterns across trades using Dataview dashboards and AI critique via the Claude API.

---

## What This System Does

- Forces a structured pre-trade checklist before any position is opened
- Creates linked trade notes tied to that checklist
- Tracks key frontmatter fields (emotion, position sizing, checklist adherence) for pattern analysis
- Aggregates mistakes, emotions, setup types, and P&L across all trades in a live dashboard
- Generates AI-powered trade critiques using Claude that reference both the trade note and the original checklist
- Builds a Trader Profile that analyzes all your trades to find your true edge and identify leaks
- Tracks profit extraction (stabling) to encourage the behavior of moving money to the bank after winning trades

---

## Required Plugins

Install all of these via **Settings → Community Plugins → Browse**:

| Plugin | Purpose |
|--------|---------|
| **Templater** | Runs the scripts that create notes and inject frontmatter |
| **Dataview** | Powers the dashboard queries and pre-trade checklist status banner |
| **Auto Note Mover** | Automatically moves closed trades to Wins or Losses folders based on tags |


---

## Initial Setup

### 1. Copy the vault folder
Place the `Trade Journal` folder anywhere inside your Obsidian vault.

### 2. Configure Templater
- `Settings → Templater`
- Set **Template folder location** to `Trade Journal/Templates`
- Enable **Trigger Templater on new file creation**

### 3. Configure Dataview
- `Settings → Dataview`
- Enable **JavaScript Queries**
- Enable **Inline JavaScript Queries**
- Set **Refresh Interval** to `500` (milliseconds) — makes the checklist banner update in near-real-time

### 4. Configure Auto Note Mover
- `Settings → Auto Note Mover`
- Add two rules:

| Tag | Destination Folder |
|-----|-------------------|
| `win` | `Trade Journal/Trades/Wins` |
| `loss` | `Trade Journal/Trades/Losses` |

- Add `Trade Journal/Templates` to **Excluded Folders** so the plugin ignores template files

### 5. Add your Anthropic API key
- Copy `Templates/Config.example.md` to `Templates/Config.md`
- Open `Config.md` and replace the placeholder with your actual API key
- Get one at [console.anthropic.com](https://console.anthropic.com) — costs less than $0.005 per critique
- `Config.md` is gitignored and will never be committed

### 6. Set up the Trader Profile
- Copy `Templates/Trader Profile.example.md` to `Trade Journal/Trader Profile.md` (one level above Templates)
- This file starts with instructions on what the profile does and how to run it
- On first run, the placeholder content is replaced with your AI-generated profile
- `Trader Profile.md` is gitignored — the generated profile stays local, the example template stays tracked

### 7. Set up hotkeys
Bind these in `Settings → Templater → Template Hotkeys`:

| Hotkey | Template | Purpose |
|--------|----------|---------|
| `Alt + P` | `Pre-Trade Checklist.md` | Create a new pre-trade checklist |
| `Alt + N` | `Trade Note.md` | Create a trade note from the active checklist |
| `Alt + A` | `Get AI Critique.md` | Run AI critique on the active trade note |
| `Alt + B` | `Build Trader Profile.md` | Build/update your Trader Profile |

- If `Alt + N` is already prebinded to another hotkey, you can either choose another hotkey to bind or rebind the prebinded one.

---

## Folder Structure

```
Trade Journal/
├── README.md                           ← this file
├── My Rules.md                         ← living rules document, update after every trade
├── Trading Dashboard.md                ← Dataview dashboard, open in Reading View
├── Trader Profile.md                   ← AI-generated trader profile (gitignored)
├── Templates/
│   ├── Config.md                       ← API key (gitignored)
│   ├── Config.example.md              ← template for Config (tracked)
│   ├── Trader Profile.example.md      ← template for Trader Profile (tracked)
│   ├── Pre-Trade Checklist.md         ← checklist template (Alt + P)
│   ├── Trade Note.md                  ← trade note template (Alt + N)
│   └── Scripts/
│       ├── Get AI Critique.md         ← AI critique script (Alt + A)
│       └── Build Trader Profile.md    ← trader profile script (Alt + B)
└── Trades/
    ├── Pre-Trades/                    ← checklists land here automatically
    ├── Wins/                          ← Auto Note Mover moves wins here
    └── Losses/                        ← Auto Note Mover moves losses here
```

---

## The Workflow

### Before every trade

1. **`Alt + P`** → Enter the ticker → Pre-Trade Checklist opens in `Pre-Trades/`
2. Fill out all four sections: Mental State, Thesis, Setup, Risk
3. Check your Final Gate decision and fill in your one-sentence reason
4. The status banner turns green when all checks are passed and a reason is provided
5. **`Alt + N`** while the checklist is open → confirms the ticker → Trade Note opens in `Trades/`

### During the trade

- Fill in entry details in the frontmatter (entry price, position size, portfolio size)
- Add execution notes as the trade develops

### After the trade

- Fill in exit fields: `exit_date`, `exit_price`, `PNL`, `result`, `followed_plan`, `emotion_post`
- Set `mistake` using one or more labels from the controlled vocabulary (see below)
- Set `tags` to match `result` (e.g. `- win` or `- loss`) so Auto Note Mover files it correctly
- Set `trade_status` to `close`
- Fill in **Post-Trade Review** and **Lesson** sections — including the process issue question on wins
- If the trade was a win, fill in `stabled_amount` frontmatter and the **Stabling Decision** section explaining how much was moved to the bank and why
- **`Alt + A`** → AI critique writes itself into the `### AI Critique` section

### Building your Trader Profile

- **`Alt + B`** → analyzes all your closed trade notes, pre-trade checklists, and post-trade reviews
- First run does a full analysis of every trade in your vault
- Subsequent runs only analyze trades added since the last update (incremental), saving API tokens
- You can force a full rebuild when prompted if you want to reprocess everything
- The profile identifies your true edge, leaks, behavioral patterns, and gives actionable recommendations
- Best results come with 15+ closed trades — the more data, the sharper the edge detection

---

## Mistake Labels (Controlled Vocabulary)

Use these standardized labels in the `mistake` frontmatter field. Multiple mistakes can be comma-separated. This is what powers the Recurring Mistakes section of the dashboard.

| Label | Meaning |
|-------|---------|
| `revenge_trade` | Entered to recover a recent loss |
| `social_proof_entry` | Entered because others were buying, not own analysis |
| `unconfirmed_catalyst` | Entered in anticipation of a catalyst, not after confirmation |
| `dca_stalling_thesis` | Averaged down into a position where thesis was weakening |
| `held_past_invalidation` | Held past own invalidation criteria |
| `profit_protection_bias` | Hold/cut decision influenced by another position's P&L |
| `oversized_position` | Position size too large for conviction level or macro conditions |
| `checklist_override` | Entered despite failing checklist items without valid override reason |
| `tilt_entry` | Entered while in an emotional/tilted state |
| `fomo_entry` | Entered due to fear of missing out, not thesis conviction |


---

## Dashboard

Open `Trading Dashboard.md` in **Reading View** (`Ctrl/Cmd + E`).

Sections:

- **Summary Stats** — total trades, win rate, P&L, profit factor
- **All Closed Trades** — full table with position sizing and portfolio %
- **Win Rate by Setup Type** — which setups actually work
- **Emotion vs Result** — how pre-trade emotional state correlates with outcomes
- **Plan Adherence** — win rate and avg P&L when you followed vs broke your plan
- **Position Sizing Analysis** — all trades sorted by % of portfolio
- **Recurring Mistakes** — aggregated mistake labels across all trades including wins
- **Biggest Losses / Wins** — top 10 each
- **Profit Extraction** — total stabled vs total winning PNL, per-trade stabling breakdown, and stabling rate by setup type
- **Open Trades** — current positions

---

## Pathing Issues

All scripts assume the folder is named exactly `Trade Journal` and placed at the **root of your vault**. If you rename the folder or nest it inside another folder, you will need to update the hardcoded paths in the template files.

**If your folder is named differently or not at vault root**, open these files and replace every instance of `Trade Journal` with your actual folder path:

- `Templates/Pre-Trade Checklist.md` — update the `tp.file.move` and `filePath` lines in the Templater script block at the bottom
- `Templates/Trade Note.md` — update the `tradeFilePath`, `pre_trade_checklist`, and `tp.file.move` lines in the Templater script block at the bottom
- `Templates/Scripts/Build Trader Profile.md` — update `PROFILE_PATH` and `TRADES_FOLDER` constants near the top, and the `Config.md` path in `loadApiKey()`

For example if your vault structure is:

```
My Vault/
└── Journals/
    └── Trade Journal/
```

Then replace `"Trade Journal/Trades/..."` with `"Journals/Trade Journal/Trades/..."` in all files.

The easiest way to find all instances is to open each file in Edit mode and use `Ctrl/Cmd + H` (find and replace) to swap `Trade Journal` for your actual path.

---

## Notes for Feedback

A few things that would be helpful to know:

- Does the pre-trade checklist friction feel appropriate, or is it too much before entering a trade?
- Are there checklist items that feel irrelevant to your trading style?
- Is the mistake vocabulary missing any labels that come up in your trades?
- Does the AI critique feel useful and specific, or too generic?
- Does the Trader Profile accurately identify your edge, or does it miss something you know about your trading?
- Does the stabling behavior tracking actually change your extraction habits, or does it just add friction?
- What dashboard sections do you find yourself actually looking at vs ignoring?

The system was built for on-chain narrative plays but the structure should work for any discretionary trading style with some adaptation.
