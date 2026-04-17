---
last_profile_update:
trades_analyzed:
mode:
---

# Trader Profile

*This note is generated and updated by the **Build Trader Profile** script. It analyzes all your closed trade notes, pre-trade checklists, and post-trade reviews to find your true edge — where you actually make money vs where you think you do.*

---

## How to Run

1. Make sure you have at least a few closed trades with filled-in post-trade reviews
2. Open the Command Palette (`Ctrl/Cmd + P`)
3. Type `Templater: Insert template` → select **Build Trader Profile**
4. Wait 30-60 seconds for the API to process your trades
5. This note will be updated with your full profile

**First run:** Analyzes all trades (full rebuild).
**Subsequent runs:** Only analyzes trades added since the last update (incremental), saving API tokens. You can force a full rebuild if prompted.

---

## What It Analyzes

- **Win rate by setup type** — which setups actually make money
- **Checklist adherence vs outcome** — does following the checklist predict wins
- **Emotional state correlations** — which pre-trade emotions lead to losses
- **Position sizing patterns** — are you oversizing on losers
- **Behavioral patterns** — revenge trading, euphoria entries, exit reasoning
- **Hold duration** — derived from entry/exit dates
- **Conviction signals** — derived from exit checks and sizing

---

*Run the script to generate your profile. The content below this line will be replaced.*
