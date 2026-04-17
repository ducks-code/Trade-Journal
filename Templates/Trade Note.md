---
ticker:
trade_status: open
result:
setup_type: cabal_play | narrative | fundamental | momentum | degen
trade_confidence:
position_size:
portfolio_size:
entry_date:
entry_price:
exit_date:
exit_price:
PNL:
stabled_amount:
followed_plan: true | false
emotion_pre: calm | confident | anxious | FOMO | revenge | bored | neutral
emotion_post: satisfied | frustrated | neutral | regret | relieved
mistake: revenge_trade | social_proof_entry | unconfirmed_catalyst | dca_stalling_thesis | held_past_invalidation | profit_protection_bias | oversized_position | checklist_override | tilt_entry | fomo_entry
pre_trade_checklist:
checklist_passed:
tags:
---

### Thesis


### Catalyst
- 

### Invalidation
- 

### Plan
- Entry:
- Exit targets:

---

### Execution Notes
- 

### Post-Trade Review
- What went well:
- What went wrong:
- What I would do differently:
- What process issue existed in this trade even if it was a win:
### Stabling Decision
- Amount stabled:
- Why this amount:
- If not stabling, why: 
### Lesson
- 
---
### Exit Checks
*Resurfaces your own exit plan and invalidation rules against what's happening now — holds you to the plan you wrote with a clear head.*

*To run while position is open: open Command Palette (`Ctrl/Cmd + P`) → `Templater: Open insert template modal` → select `Exit Check` or press `Alt + X`. Can be run multiple times — each check is timestamped and appended.*

---
### AI Critique
*To generate critique: open Command Palette (`Ctrl/Cmd + P`) → type `Templater: Open insert template modal` → select `Get AI Critique` or press `Alt + A`*

<%*
const checklistFile = app.workspace.getActiveFile();
if (!checklistFile) { new Notice("No active file."); return; }

// ── Regex check: must be an actual checklist note ─────────────
const checklistPattern = /^[A-Z0-9]+ \d{4}-\d{2}-\d{2} \d{2}-\d{2} PRE-TRADE/;
if (!checklistPattern.test(checklistFile.basename)) {
  new Notice("⛔ Open an actual Pre-Trade Checklist note (not the template) and try again.", 8000);
  return;
}

const checklistContent = await app.vault.read(checklistFile);

// ── Gate state ────────────────────────────────────────────────
const proceeding = /^- \[x\] Proceeding with trade$/mi.test(checklistContent);
const sittingOut = /^- \[x\] Sitting this one out$/mi.test(checklistContent);

const hasOneLine = checklistContent.includes("**One sentence on why I am taking this trade:**") &&
  !checklistContent.includes("**One sentence on why I am taking this trade:**\n>\n");

if (proceeding && !hasOneLine) {
  new Notice("⛔ Fill in your one sentence reason in the Final Gate before creating a trade note.", 8000);
  return;
}

if (proceeding && sittingOut) {
  new Notice("⛔ Both Final Gate options are checked — pick one only.", 8000);
  return;
}

if (!proceeding && !sittingOut) {
  new Notice("⛔ Final Gate decision not made — check Proceeding or Sitting this one out.", 8000);
  return;
}

if (sittingOut) {
  new Notice("Final Gate: sitting this one out. No trade note created.");
  return;
}

// ── Extract ticker and confirm ────────────────────────────────
const checklistName = checklistFile.basename;
const ticker = checklistName.split(" ")[0].toUpperCase();
const today = tp.date.now("YYYY-MM-DD");

const confirm = await tp.system.prompt(
  "Creating trade note for " + ticker + " (" + today + "). Confirm ticker or type a new one:",
  ticker
);
if (!confirm) { new Notice("Cancelled."); return; }

const finalTicker = confirm
  .trim()
  .toUpperCase()
  .replace(/[\\/:*?"<>|]/g, "")
  .replace(/\s+/g, "");

if (!finalTicker) { new Notice("Invalid ticker."); return; }

// ── Count unchecked non-gate boxes only ───────────────────────
const rawUnchecked = (checklistContent.match(/- \[ \]/g) || []).length;
const gateUnchecked = (proceeding ? 1 : 0) + (sittingOut ? 1 : 0);
const unchecked = rawUnchecked - (2 - gateUnchecked);

if (unchecked > 0) {
  const proceed = await tp.system.prompt(unchecked + " box(es) still unchecked. Create trade note anyway? (yes/no)");
  if (proceed?.toLowerCase() !== "yes") { new Notice("Cancelled."); return; }
}

const passed = unchecked === 0 ? "true" : "false - " + unchecked + " overrides";

// ── Update checklist frontmatter ──────────────────────────────
const updatedChecklist = checklistContent.replace(
  /checklist_passed:.*/,
  "checklist_passed: " + passed
);
await app.vault.modify(checklistFile, updatedChecklist);

// ── Write frontmatter values after template renders ───────────
setTimeout(async () => {
  const tradeFilePath = "Trade Journal/Trades/" + today + " " + finalTicker + ".md";
  const tradeFile = app.vault.getAbstractFileByPath(tradeFilePath);
  if (!tradeFile) return;
  await app.fileManager.processFrontMatter(tradeFile, fm => {
    fm["ticker"] = finalTicker;
    fm["entry_date"] = today;
    fm["pre_trade_checklist"] = "[[Trade Journal/Trades/Pre-Trades/" + checklistName + "]]";
    fm["checklist_passed"] = passed;
  });
}, 1500);

// ── Rename and move ───────────────────────────────────────────
const tradeFileName = today + " " + finalTicker;
await tp.file.rename(tradeFileName);
await tp.file.move("Trade Journal/Trades/" + tradeFileName);

new Notice("Trade note created: " + tradeFileName);
%>
