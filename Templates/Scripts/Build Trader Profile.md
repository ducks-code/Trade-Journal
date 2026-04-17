<%*
// ── CONFIG — read API key from Config.md ──────────────────────
async function loadApiKey() {
  const configFile = app.vault.getAbstractFileByPath("Trade Journal/Templates/Config.md");
  if (!configFile) {
    new Notice("⛔ Templates/Config.md not found. Create it with: API_KEY: <your-key>", 10000);
    return null;
  }
  const raw = await app.vault.read(configFile);
  const match = raw.match(/API_KEY:\s*(\S+)/);
  if (!match) {
    new Notice("⛔ API_KEY not found in Config.md. Format: API_KEY: sk-ant-...", 10000);
    return null;
  }
  return match[1].trim();
}

const API_KEY = await loadApiKey();
if (!API_KEY) { return; }
const PROFILE_PATH = "Trade Journal/Trader Profile.md";
const TRADES_FOLDER = "Trade Journal/Trades";

// ── Read existing profile ─────────────────────────────────────
const profileFile = app.vault.getAbstractFileByPath(PROFILE_PATH);
if (!profileFile) {
  new Notice("⛔ Trader Profile.md not found. Place the template file at: " + PROFILE_PATH, 10000);
  return;
}

const profileContent = await app.vault.read(profileFile);
let existingProfile = profileContent;
let lastUpdate = null;

// Extract last_profile_update from frontmatter
const dateMatch = profileContent.match(/last_profile_update:\s*(\d{4}-\d{2}-\d{2})/);
if (dateMatch) {
  lastUpdate = dateMatch[1];
}

// ── Gather all trade notes ────────────────────────────────────
const allFiles = app.vault.getMarkdownFiles();
const tradeFiles = allFiles.filter(f =>
  f.path.startsWith(TRADES_FOLDER + "/") &&
  !f.path.includes("/Templates/") &&
  !f.path.includes("/Pre-Trades/") &&
  f.basename !== "Trading Dashboard" &&
  f.basename !== "Trader Profile"
);

if (tradeFiles.length === 0) {
  new Notice("No trade notes found in " + TRADES_FOLDER);
  return;
}

// ── Filter to new trades only (incremental mode) ──────────────
let tradesToAnalyze = tradeFiles;
let mode = "full";

if (lastUpdate && !existingProfile.includes("*Run the script to generate your profile.")) {
  const newTrades = tradeFiles.filter(f => {
    const content = app.metadataCache.getFileCache(f)?.frontmatter;
    if (!content || !content.entry_date) return false;
    return content.entry_date > lastUpdate;
  });

  if (newTrades.length === 0) {
    const rebuild = await tp.system.prompt(
      "No new trades since last profile update (" + lastUpdate + "). Force full rebuild? (yes/no)"
    );
    if (rebuild?.toLowerCase() !== "yes") {
      new Notice("No new trades to analyze.");
      return;
    }
    // Fall through to full rebuild
  } else {
    tradesToAnalyze = newTrades;
    mode = "incremental";
    new Notice("Analyzing " + newTrades.length + " new trade(s) since " + lastUpdate + "...", 4000);
  }
}

if (mode === "full") {
  new Notice("Building full profile from " + tradesToAnalyze.length + " trade(s)...", 4000);
}

// ── Build trade data payload ──────────────────────────────────
const tradeData = [];

for (const tradeFile of tradesToAnalyze) {
  const content = await app.vault.read(tradeFile);
  const fm = app.metadataCache.getFileCache(tradeFile)?.frontmatter || {};

  // Read linked pre-trade checklist if available
  let checklistContent = "";
  if (fm.pre_trade_checklist) {
    const linkMatch = String(fm.pre_trade_checklist).match(/\[\[(.*?)\]\]/);
    if (linkMatch) {
      let checklistPath = linkMatch[1];
      if (!checklistPath.endsWith(".md")) {
        checklistPath += ".md";
      }
      const checklistFile = app.vault.getAbstractFileByPath(checklistPath);
      if (checklistFile) {
        checklistContent = await app.vault.read(checklistFile);
      }
    }
  }

  tradeData.push({
    ticker: fm.ticker || "unknown",
    result: fm.result || "unknown",
    setup_type: fm.setup_type || "unknown",
    trade_confidence: fm.trade_confidence || "",
    position_size: fm.position_size || "",
    entry_date: fm.entry_date || "",
    exit_date: fm.exit_date || "",
    PNL: fm.PNL || "",
    followed_plan: fm.followed_plan ?? "",
    emotion_pre: fm.emotion_pre || "",
    emotion_post: fm.emotion_post || "",
    mistake: fm.mistake || "",
    checklist_passed: fm.checklist_passed || "",
    trade_note_body: content,
    checklist_body: checklistContent
  });
}

// ── Build the prompt ──────────────────────────────────────────
const SYSTEM = `You are an elite trading performance analyst. Your job is to find this trader's TRUE EDGE — not what they think their edge is, but what the data actually shows.

You specialise in CT-driven crypto trading across Base, Solana, and other chains. The trader uses setup types: cabal_play, narrative, fundamental, momentum, degen.

## Your analysis framework:

### 1. EDGE IDENTIFICATION (most important section)
Find where the trader actually makes money. Cross-reference:
- Win rate by setup type
- Average PNL by setup type
- Win rate when checklist is fully passed vs overridden
- Win rate when plan was followed vs not
- Win rate by emotional state at entry
- Which combinations of conditions produce the best results

The edge is the INTERSECTION of conditions where the trader consistently wins. Be specific: "Your edge is [setup type] trades where [conditions], producing [stats]."

### 2. LEAK REPORT
Identify where money is being lost. Be specific with numbers:
- Which setup types are net negative
- What percentage of losses come from checklist overrides
- Emotional states that correlate with losses
- Sizing patterns on losing vs winning trades
- Any chain-specific underperformance

### 3. PROFIT EXTRACTION ANALYSIS
Analyze stabling behavior across all winning trades:
- Overall extraction rate (total stabled / total winning PNL)
- Patterns by PNL size — do they stable small wins but not large ones, or vice versa?
- Patterns by setup type — are certain setups treated as "free money" that gets recycled?
- Patterns by emotional state post-trade — does euphoria correlate with not stabling?
- Common rationalizations in the Stabling Decision sections

### 4. BEHAVIORAL PATTERNS
Extract from post-trade reviews, exit checks, AI critiques, and checklist override reasons:
- Conviction level patterns (derived from exit check notes and position sizing)
- Hold duration patterns (derived from entry_date vs exit_date)
- Exit reasoning patterns (derived from exit checks and post-trade reviews)
- Emotional cycles (what happens after wins vs after losses)
- Revenge trading sequences
- Euphoria-driven entries after wins

### 5. PROFILE SUMMARY
One paragraph: "You are a [type] trader whose edge comes from [specific conditions]. Your biggest leak is [specific pattern]. The single highest-impact change would be [specific action]."

## Output format:

---
## Trader Profile
*Last updated: [today's date] | Trades analyzed: [count] | Mode: [full/incremental]*

### Edge Summary
[2-3 sentences: where the trader actually makes money]

### Setup Type Breakdown
| Setup | Trades | Wins | Losses | Win Rate | Avg Win | Avg Loss | Net PNL |
[table with actual numbers]

### Conditions That Predict Wins
[Ranked list of conditions that correlate with winning trades]

### Conditions That Predict Losses
[Ranked list of conditions that correlate with losing trades]

### Leak Report
[Specific money leaks with numbers, including:] [- Profit recycling — total winning PNL vs total stabled, with extraction rate %] [- Patterns of non-stabling (e.g. "degen wins stabled 0% of the time")]

### Behavioral Patterns
[Patterns extracted from qualitative content — reviews, exit checks, override reasons]

### Trader Archetype
[1 paragraph profile summary]

### Recommendations
[3 specific, actionable changes ranked by expected impact. If extraction rate is low, include a concrete stabling rule — e.g. "Stable minimum 30% of any win over $500" — with specific threshold based on the trader's actual win distribution.]
---

Be brutally honest. Use actual numbers from the data. Do not soften findings. If the data shows a setup type is net negative, say so clearly. If the trader's self-perceived edge doesn't match the data, call that out.`;

// ── Build the user message ────────────────────────────────────
let userMessage = "";

if (mode === "incremental") {
  userMessage = "## EXISTING TRADER PROFILE\n\n" + existingProfile +
    "\n\n---\n\n## NEW TRADES TO INCORPORATE\n\n" +
    "Analyze these " + tradeData.length + " new trade(s) and UPDATE the existing profile. " +
    "Recalculate all stats to include the new data. If patterns have shifted, note the change.\n\n";
} else {
  userMessage = "## FULL TRADE HISTORY\n\n" +
    "Build a complete trader profile from these " + tradeData.length + " trade(s).\n\n";
}

for (const trade of tradeData) {
  userMessage += "### TRADE: " + trade.ticker + " (" + trade.entry_date + ")\n";
  userMessage += "Result: " + trade.result + " | Setup: " + trade.setup_type +
    " | PNL: " + trade.PNL + " | Confidence: " + trade.trade_confidence +
    " | Size: " + trade.position_size + " | Checklist: " + trade.checklist_passed +
    " | Followed Plan: " + trade.followed_plan +
    " | Emotion Pre: " + trade.emotion_pre + " | Emotion Post: " + trade.emotion_post +
    " | Mistake: " + trade.mistake + "\n\n";
  userMessage += "**Trade Note:**\n" + trade.trade_note_body + "\n\n";

  if (trade.checklist_body) {
    userMessage += "**Pre-Trade Checklist:**\n" + trade.checklist_body + "\n\n";
  }

  userMessage += "---\n\n";
}

// ── Call the API ──────────────────────────────────────────────
new Notice("Calling Claude API — this may take 30-60 seconds...", 8000);

let profile = "";
try {
  const response = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "x-api-key": API_KEY,
      "anthropic-version": "2023-06-01",
      "anthropic-dangerous-direct-browser-access": "true"
    },
    body: JSON.stringify({
      model: "claude-sonnet-4-20250514",
      max_tokens: 3000,
      system: SYSTEM,
      messages: [{ role: "user", content: userMessage }]
    })
  });

  if (!response.ok) {
    const err = await response.json();
    new Notice("API error: " + (err.error?.message || response.status));
    return;
  }

  const data = await response.json();
  profile = (data.content || []).map(b => b.text || "").join("").trim();

} catch(e) {
  new Notice("Failed to build profile: " + e.message);
  return;
}

if (!profile) {
  new Notice("API returned empty profile.");
  return;
}

// ── Build the final note content ──────────────────────────────
const today = tp.date.now("YYYY-MM-DD HH:mm");
const finalContent = [
  "---",
  "last_profile_update: " + today,
  "trades_analyzed: " + (mode === "incremental" ? "incremental +" + tradeData.length : tradeData.length),
  "mode: " + mode,
  "---",
  "",
  profile
].join("\n");

// ── Write profile into existing note ──────────────────────────
await app.vault.modify(profileFile, finalContent);
new Notice("Trader Profile updated (" + mode + " — " + tradeData.length + " trades).", 6000);

// ── Open the profile ─────────────────────────────────────────
const leaf = app.workspace.getLeaf();
await leaf.openFile(profileFile);
%>
