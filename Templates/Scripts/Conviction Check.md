<%*
// ══════════════════════════════════════════════════════════════
// Conviction Check — run on an open Pre-Trade Checklist
// Reads checklist sections + Final Gate reasoning, sends to AI
// AI challenges your thesis and asks hard questions — does NOT
// tell you whether to buy.
// Bind to: Alt + Q (or any free hotkey)
// ══════════════════════════════════════════════════════════════

const API_KEY = await (async function loadApiKey() {
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
})();
if (!API_KEY) { return; }

// ── Read the current checklist note ───────────────────────────
const file = app.workspace.getActiveFile();
if (!file) { new Notice("No active file."); return; }

// ── Must be an actual checklist note, not the template ────────
const checklistPattern = /^[A-Z0-9]+ \d{4}-\d{2}-\d{2} \d{2}-\d{2} PRE-TRADE/;
if (!checklistPattern.test(file.basename)) {
  new Notice("⛔ Open an actual Pre-Trade Checklist note (not the template) and try again.", 8000);
  return;
}

const content = await app.vault.read(file);

// ── Check Final Gate has a one-sentence reason ────────────────
const hasOneLine = content.includes("**One sentence on why I am taking this trade:**") &&
  !content.includes("**One sentence on why I am taking this trade:**\n>\n");

if (!hasOneLine) {
  new Notice("⛔ Write your one-sentence reason in Final Gate before running Conviction Check.", 8000);
  return;
}

// ── Check for override reasons ────────────────────────────────
const overrideBlocks = content.match(/Override reason \(if any box unchecked\):\s*\n>\s*(.+)/gi) || [];
const overrideReasons = overrideBlocks
  .map(block => {
    const match = block.match(/>\s*(.+)/);
    return match ? match[1].trim() : "";
  })
  .filter(r => r.length > 0);

new Notice("Running conviction check...");

// ── System prompt ─────────────────────────────────────────────
const SYSTEM = `You are a sharp trading coach. The trader is about to enter a crypto trade and has filled out a pre-trade checklist. Your job is NOT to tell them whether to buy. Your job is to stress-test their reasoning so THEY can decide with more clarity.

Read the checklist carefully — every checked/unchecked box, every override reason, and the Final Gate reasoning.

=== FRAMEWORK ===

THESIS STRESS TEST:
- Is the one-sentence reason a real thesis or a vague feeling? A thesis names a specific catalyst and why the market is mispricing it. "It's going up" is not a thesis.
- Does the thesis depend on something the trader controls, or on hope that someone else does something (team launches X, KOL shills it, market rotates)?
- If the thesis hinges on hope, flag it directly: "This reads like a hope trade, not a thesis trade."
- Could this thesis have been written BEFORE the price moved, or is it post-hoc rationalization of a pump they saw?

OVERRIDE SCRUTINY:
- For each unchecked box with an override reason, assess: is this a genuine edge case where the rule doesn't apply, or is the trader rationalizing their way past a guardrail?
- Mental State overrides are the most dangerous — trading while tilted, FOMOing, or revenge trading with a written justification doesn't make it disciplined.
- Multiple overrides compound risk. Flag this explicitly if present.

WHAT YOU'RE MISSING:
- Point out what information the trader DOESN'T have. What assumptions are they making? What could invalidate this in the next 24 hours?
- If no invalidation criteria are written in the checklist, flag that hard.

DEVIL'S ADVOCATE:
- Give the strongest 2-3 sentence bear case against this trade. Not generic risk warnings — specific to what they've written.

Return your response in this EXACT format with no extra text before or after:

## Conviction Check — {ticker}

#### Thesis Assessment
<2-3 sentences: is this a real thesis or rationalization? Is conviction built on research or on price action they saw? Is this a thesis trade or a hope trade?>

#### Override Review
<If overrides exist: 2-3 sentences per override assessing validity. If no overrides: "No overrides — checklist fully passed.">

#### What You're Not Seeing
<2-3 sentences: what assumptions are baked in? What information is missing? What could go wrong that isn't accounted for?>

#### Bear Case
> <2-3 sentence strongest argument AGAINST this trade, specific to their thesis>

#### Questions to Answer Before Entry
> 1. <Hard question #1 — something they need to honestly answer>
> 2. <Hard question #2>
> 3. <Hard question #3>

Do NOT say "buy" or "don't buy." Do NOT give a score. Do NOT make a recommendation. Your job is to make the trader think harder, not to think for them.`;

// ── Call the API ──────────────────────────────────────────────
let response_text = "";
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
      max_tokens: 1200,
      system: SYSTEM,
      messages: [{ role: "user", content: "PRE-TRADE CHECKLIST:\n" + content }]
    })
  });

  if (!response.ok) {
    const err = await response.json();
    new Notice("API error: " + (err.error?.message || response.status));
    return;
  }

  const data = await response.json();
  response_text = (data.content || []).map(b => b.text || "").join("").trim();

} catch(e) {
  new Notice("Failed to fetch conviction check: " + e.message);
  return;
}

// ── Write into checklist — below Final Gate, above the script ─
const sectionHeader = "\n\n---\n\n### AI Conviction Check\n";
const placeholder = "*Run Conviction Check after completing your Final Gate reasoning.*";

if (content.includes("### AI Conviction Check")) {
  // ── Already exists — ask to overwrite ───────────────────────
  const overwrite = await tp.system.prompt("Conviction check already exists. Overwrite? (yes/no)");
  if (overwrite?.toLowerCase() !== "yes") { new Notice("Cancelled."); return; }

  const updatedContent = content.replace(
    /(### AI Conviction Check\n)([\s\S]*?)(\n---|\n<%|$)/,
    "$1" + response_text + "\n$3"
  );
  await app.vault.modify(file, updatedContent);
} else {
  // ── Insert before the Templater script block ────────────────
  const scriptStart = content.indexOf("<%*");
  if (scriptStart > -1) {
    const updatedContent = content.slice(0, scriptStart) + sectionHeader + response_text + "\n\n" + content.slice(scriptStart);
    await app.vault.modify(file, updatedContent);
  } else {
    // No script block — append at end
    const updatedContent = content + sectionHeader + response_text + "\n";
    await app.vault.modify(file, updatedContent);
  }
}

new Notice("✅ Conviction check written to checklist.", 6000);
%>
