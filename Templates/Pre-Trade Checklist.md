---
date: <% tp.date.now("YYYY-MM-DD HH:mm") %>
ticker: 
chain: 
setup_type: 
checklist_passed: 
tags:
  - pre-trade
---


# Pre-Trade Checklist — <% tp.date.now("YYYY-MM-DD") %>

---

```dataviewjs
// ── Get current file and content ──────────────────────────────
const file = app.workspace.getActiveFile();
if (!file) return;
const content = await app.vault.read(file);

// ── Final Gate checks ─────────────────────────────────────────
const proceeding = /^- \[x\] Proceeding with trade$/mi.test(content);
const sittingOut  = /^- \[x\] Sitting this one out$/mi.test(content);

// ── One sentence reason check ─────────────────────────────────
const hasOneLine = content.includes("**One sentence on why I am taking this trade:**") &&
  !content.includes("**One sentence on why I am taking this trade:**\n>\n");

// ── Checkbox counts ───────────────────────────────────────────
const allChecked   = (content.match(/- \[x\]/gi) || []).length;
const allUnchecked = (content.match(/- \[ \]/g)  || []).length;

// Always subtract both gate checkboxes from total
const gateChecked   = (proceeding ? 1 : 0) + (sittingOut ? 1 : 0);
const gateUnchecked = 2 - gateChecked;

const relevantChecked   = allChecked - gateChecked;
const relevantUnchecked = allUnchecked - gateUnchecked;
const total = relevantChecked + relevantUnchecked;

let pct = 0;
if (total > 0) {
  pct = Math.round((relevantChecked / total) * 100);
}

// ── Status logic ──────────────────────────────────────────────
let bg, color, icon, message;

if (proceeding && sittingOut) {
  bg      = "rgba(255,60,60,0.12)";
  color   = "#ff3c3c";
  icon    = "✗";
  message = "Both Final Gate options are checked — pick one only";

} else if (!proceeding && !sittingOut) {
  bg      = "rgba(255,60,60,0.12)";
  color   = "#ff3c3c";
  icon    = "✗";
  message = relevantUnchecked + " items unchecked — Final Gate decision not set";

} else if (sittingOut) {
  bg      = "rgba(255,170,0,0.12)";
  color   = "#ffaa00";
  icon    = "⚠";
  message = "Sitting this one out — no trade note needed";

} else if (proceeding && !hasOneLine) {
  bg      = "rgba(255,170,0,0.12)";
  color   = "#ffaa00";
  icon    = "⚠";
  message = "Fill in your reasoning on why you're taking this trade before entering";

} else if (relevantUnchecked === 0 && proceeding && hasOneLine) {
  bg      = "rgba(0,180,80,0.12)";
  color   = "#00b450";
  icon    = "✓";
  message = "All checks passed — green light to trade";

} else {
  const remaining = relevantUnchecked;
  let urgency = "amber";
  if (remaining > 2) {
    urgency = "red";
  }

  if (urgency === "red") {
    bg    = "rgba(255,60,60,0.12)";
    color = "#ff3c3c";
    icon  = "✗";
  } else {
    bg    = "rgba(255,170,0,0.12)";
    color = "#ffaa00";
    icon  = "⚠";
  }

  message = remaining + " item";
  if (remaining > 1) {
    message = message + "s";
  }
  message = message + " unchecked — review override reasons before entering";
}

// ── Render ────────────────────────────────────────────────────
const progressBar = [
  "<div style=\"height:4px;background:rgba(128,128,128,0.15);border-radius:2px;margin:10px 0 4px;\">",
  "<div style=\"height:4px;width:" + pct + "%;background:" + color + ";border-radius:2px;\"></div>",
  "</div>"
].join("");

const counter = [
  "<span style=\"font-family:monospace;font-size:13px;color:" + color + ";opacity:0.8;\">",
  relevantChecked + "/" + total,
  "</span>"
].join("");

const label = [
  "<span style=\"font-size:15px;font-weight:500;color:" + color + ";\">",
  icon + " " + message,
  "</span>"
].join("");

const banner = [
  "<div style=\"background:" + bg + ";border:1px solid " + color + "33;border-radius:8px;padding:14px 18px;margin:4px 0 16px;\">",
  "<div style=\"display:flex;align-items:center;justify-content:space-between;\">",
  label,
  counter,
  "</div>",
  progressBar,
  "</div>"
].join("");

this.container.innerHTML = banner;
```

> **Tip:** Set Dataview refresh interval to 500ms in settings for near-live updates.

---

## 🧠 Mental State

- [ ] I have not closed a losing trade in the last 24 hours
- [ ] I am not trading to recover a loss
- [ ] I am not entering because I feel like I am missing out
- [ ] I can sit out this trade without feeling anxious

**Override reason (if any box unchecked):**
>

---

## 📐 Thesis

- [ ] I can explain the thesis in one sentence
- [ ] I have identified a specific catalyst — not vibes or narrative momentum alone
- [ ] I know why this asset is mispriced right now
- [ ] The thesis is mine — not copied from CT

**Override reason (if any box unchecked):**
>

---

## 🔍 Setup

- [ ] I have checked the chart and PA supports the thesis
- [ ] I know exactly what invalidates this trade before I enter
- [ ] The invalidation is specific and enforceable, not vague

**Override reason (if any box unchecked):**
>

---

## 💰 Risk
- [ ] My position size matches my actual conviction, not my desired outcome
- [ ] My position size is appropriate for current macro conditions — not oversized in risk off environment
- [ ] I know my exact exit targets before entering
- [ ] I can afford to lose this entire position without affecting my next trade
- [ ] I am not sizing up to recover a previous loss

**Override reason (if any box unchecked):**
>

---

## ✅ Final Gate

**Decision:**
- [ ] Proceeding with trade
- [ ] Sitting this one out

**One sentence on why I am taking this trade:**
>

**One sentence on why (I AM NOT) taking this trade (if applicable):**
>
---
### AI Conviction Check
*Stress-tests your thesis and override reasons without telling you to buy or not — forces you to think harder before entry.*
*To run: complete your Final Gate reasoning, then open Command Palette (`Ctrl/Cmd + P`) → `Templater: Open insert template modal` → select `Conviction Check` or press `Alt + Q`*

<%*
// ── Euphoria detection — scan for recent wins & open winners ──
const tradeFolders = ["Trade Journal/Trades", "Trade Journal/Trades/Wins"];
const now = new Date();
const cutoffMs = 48 * 60 * 60 * 1000; // 48 hours
let recentWins = [];
let openWinners = [];

for (const folderPath of tradeFolders) {
  const folder = app.vault.getAbstractFileByPath(folderPath);
  if (!folder || !folder.children) { continue; }

  for (const f of folder.children) {
    if (!f.name.endsWith(".md")) { continue; }

    const meta = app.metadataCache.getFileCache(f);
    if (!meta || !meta.frontmatter) { continue; }
    const fm = meta.frontmatter;

    // Recent closed wins (within 48h)
    if (fm.result === "win" && fm.exit_date) {
      const exitDate = new Date(fm.exit_date);
      if (now - exitDate < cutoffMs) {
        recentWins.push(fm.ticker + " (closed " + fm.exit_date + ", PNL: " + fm.PNL + ")");
      }
    }

    // Open trades (potential unrealized gains)
    if (fm.trade_status === "open") {
      openWinners.push(fm.ticker + " (open since " + fm.entry_date + ")");
    }
  }
}

// ── Show euphoria warning if triggers detected ────────────────
let euphoriaSectionTriggered = false;

if (recentWins.length > 0 || openWinners.length > 0) {
  euphoriaSectionTriggered = true;

  let warningMsg = "⚠️ EUPHORIA CHECK — potential elevated risk state detected:\n\n";

  if (recentWins.length > 0) {
    warningMsg += "Recent wins (last 48h):\n";
    for (const w of recentWins) {
      warningMsg += "  • " + w + "\n";
    }
  }
  if (openWinners.length > 0) {
    warningMsg += "Open positions:\n";
    for (const o of openWinners) {
      warningMsg += "  • " + o + "\n";
    }
  }

  warningMsg += "\nEuphoria leads to oversizing, random coins, and round-tripping. Pay extra attention to the Euphoria Check section.\n\nContinue? (yes/no)";

  const proceed = await tp.system.prompt(warningMsg);
  if (proceed?.toLowerCase() !== "yes") {
    new Notice("Checklist creation cancelled. Take a breath.", 6000);
    return;
  }
}

// ── Ticker input ──────────────────────────────────────────────
const tickerInput = await tp.system.prompt("Ticker");
if (!tickerInput) { new Notice("No ticker entered."); return; }

const ticker = tickerInput
  .trim()
  .toUpperCase()
  .replace(/[\\/:*?"<>|]/g, "")
  .replace(/\s+/g, "");

if (!ticker) { new Notice("Ticker is empty after cleaning."); return; }

const ts = tp.date.now("YYYY-MM-DD HH-mm");
const newName = ticker + " " + ts + " PRE-TRADE";
await tp.file.rename(newName);
await tp.file.move("Trade Journal/Trades/Pre-Trades/" + newName);

// ── Inject Euphoria Check section if triggered ────────────────
if (euphoriaSectionTriggered) {
  setTimeout(async () => {
    const filePath = "Trade Journal/Trades/Pre-Trades/" + newName + ".md";
    const noteFile = app.vault.getAbstractFileByPath(filePath);
    if (!noteFile) { return; }

    const noteContent = await app.vault.read(noteFile);

    const euphoriaSection = [
      "",
      "## 🔥 Euphoria Check",
      "*Triggered: recent wins or open positions detected. This section only appears when you're in a potential euphoria state.*",
      "",
      "- [ ] I am not sizing bigger than usual because I'm on a hot streak",
      "- [ ] This trade came from my watchlist — I did not find it in the last hour",
      "- [ ] I would still take this trade if my last trade had been a loss",
      "- [ ] I am not entering a random coin outside my usual edge because I feel invincible",
      "",
      "**Override reason (if any box unchecked):**",
      ">",
      "",
      "---",
      ""
    ].join("\n");

    // Insert after Mental State section (after its ---)
    const thesisIndex = noteContent.indexOf("## 📐 Thesis");
    if (thesisIndex > -1) {
      const updated = noteContent.slice(0, thesisIndex) + euphoriaSection + noteContent.slice(thesisIndex);
      await app.vault.modify(noteFile, updated);
    }
  }, 2000);
}

// ── Update frontmatter ────────────────────────────────────────
setTimeout(async () => {
  const filePath = "Trade Journal/Trades/Pre-Trades/" + newName + ".md";
  const file = app.vault.getAbstractFileByPath(filePath);
  if (!file) return;
  await app.fileManager.processFrontMatter(file, fm => {
    fm["ticker"] = ticker;
    fm["date"] = tp.date.now("YYYY-MM-DD HH:mm");
  });
}, 1500);
%>
