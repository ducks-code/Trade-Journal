<%*
// ══════════════════════════════════════════════════════════════
// Exit Check — run on an open Trade Note
// Reads your plan, thesis, invalidation, and execution notes.
// AI resurfaces YOUR rules and asks whether exit conditions
// are met — does NOT tell you to sell or hold.
// Bind to: Alt + X (or any free hotkey)
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

// ── Read the current trade note ───────────────────────────────
const file = app.workspace.getActiveFile();
if (!file) { new Notice("No active file."); return; }

const content = await app.vault.read(file);

// ── Basic validation: must have a thesis and plan ─────────────
const hasThesis = /### Thesis\n[\s\S]*?\S/.test(content);
const hasPlan = /### Plan\n[\s\S]*?\S/.test(content);

if (!hasThesis || !hasPlan) {
  new Notice("⛔ Trade note needs a written Thesis and Plan before running Exit Check.", 8000);
  return;
}

// ── Check trade_status is open ────────────────────────────────
const statusMatch = content.match(/^trade_status:\s*(.+)$/m);
const tradeStatus = statusMatch ? statusMatch[1].trim() : "";

if (tradeStatus === "closed") {
  new Notice("This trade is already closed. Use Get AI Critique instead.", 6000);
  return;
}

// ── Prompt for current context the AI can't see ───────────────
const currentPrice = await tp.system.prompt("Current price (or leave blank):");
const contextNote = await tp.system.prompt("What's happening right now? (brief — e.g. 'volume dying, CT quiet', 'pumping hard everyone bullish', 'thesis catalyst hasn't fired yet')");

if (!contextNote || contextNote.trim().length < 3) {
  new Notice("⛔ Provide at least a brief context note so the check is useful.", 6000);
  return;
}

// ── Try to read linked pre-trade checklist ────────────────────
let checklistContent = "";
const checklistMatch = content.match(/pre_trade_checklist:\s*"?\[\[([^\]]+)\]\]"?/);
if (checklistMatch) {
  const checklistPath = checklistMatch[1].trim() + ".md";
  const checklistFile = app.vault.getAbstractFileByPath(checklistPath);
  if (checklistFile) {
    checklistContent = await app.vault.read(checklistFile);
  }
}

// ── Build user message ────────────────────────────────────────
let userMessage = "TRADE NOTE:\n" + content;
userMessage += "\n\n---\nCURRENT CONTEXT FROM TRADER:";
if (currentPrice && currentPrice.trim()) {
  userMessage += "\nCurrent price: " + currentPrice.trim();
}
userMessage += "\nSituation: " + contextNote.trim();
if (checklistContent) {
  userMessage += "\n\n---\nPRE-TRADE CHECKLIST:\n" + checklistContent;
}

new Notice("Running exit check...");

// ── System prompt ─────────────────────────────────────────────
const SYSTEM = `You are a trading coach helping a crypto trader evaluate whether to exit a position. You have access to their original trade note (thesis, plan, invalidation, execution notes) and a brief context update they've provided about what's happening right now.

YOUR ROLE: Resurface THEIR OWN RULES and help them evaluate whether those rules say it's time to act. You do NOT tell them to sell or hold. You hold them accountable to the plan they wrote when they were thinking clearly.

PRIOR EXIT CHECKS: The trade note may contain previous exit checks under "### Exit Checks" with timestamps. If present, use them as critical context:
- Track how the trader's reasoning has evolved. Are they becoming more hopeful or more honest?
- Flag if the same concerns were raised before and nothing changed — "This is the second check where volume is flagged as dying. What's different now?"
- Note if the trader's context descriptions are shifting to justify holding (rationalizing) vs honestly reporting conditions.
- If a previous check asked hard questions the trader clearly hasn't answered or acted on, call that out directly.

=== EXIT PRINCIPLES TO APPLY ===

PLAN ENFORCEMENT:
- Read their "Exit targets" from the Plan section. Have any been hit? If the trader hasn't provided specific targets, flag this as a structural problem.
- Read their "Invalidation" section. Based on the current context they've described, has the invalidation been triggered or is it close to triggering?
- If the original thesis catalyst hasn't fired and no timeline was set, flag this: "No deadline on your catalyst = no way to know when the thesis is dead."

EMOTIONAL SIGNALS:
- "Would I buy the same amount at this price?" — if the trader's context note suggests they wouldn't, that's a sell signal by their own framework.
- If the context sounds stressed, fearful, or euphoric, flag which emotional state you're detecting and how it historically affects this trader's decisions.
- Charts almost never make you bullish. If the trader mentions chart-watching, screen time, or candle-by-candle updates — flag high-frequency watching as decision quality erosion.

EXIT FRAMEWORK:
- Sell when the reason for buying is gone.
- Sell when volume dies.
- Sell when you start to stress (emotional = irrational).
- You will not catch the top. Capturing most of the move beats trying to perfectly time it.
- If drivers are intact, the trade is intact. If a driver breaks, the trade changes. Measure against catalysts, not candle color.

DANGER SIGNALS:
- Hope-based holding: "maybe tomorrow" or "I think it could still..." without citing specific drivers.
- Anchoring to entry price or ATH — refusing to sell below a mental number.
- Re-evaluating the thesis to justify staying in rather than honestly assessing whether the thesis still holds.
- Bag-swapping temptation: wanting to exit into something "hotter" rather than because the trade's thesis is done.

Return your response in this EXACT format with no extra text before or after:

## Exit Check — {ticker}

#### Your Plan Says
<2-3 sentences: restate their specific exit targets and invalidation criteria from the trade note. If targets are vague or missing, call this out.>

#### Current Situation vs Plan
<2-3 sentences: based on the context they provided, how does the current situation map to their exit conditions? Which conditions are met, close to being met, or not yet triggered?>

#### Thesis Status
<2-3 sentences: is the original thesis still intact based on what they've described? Have the catalysts fired, partially fired, or failed? Is this still a thesis trade or has it become a hope trade?>

#### Emotional Read
<1-2 sentences: what emotional state does their context note suggest? How does that map to their known patterns?>

#### Questions to Answer Honestly
> 1. <Hard question about whether they'd buy this same position at this price today>
> 2. <Hard question about whether they're holding based on thesis or hope>
> 3. <Hard question specific to what they described — e.g. if volume is dying, "What volume level would confirm your thesis is dead?">

Do NOT say "sell" or "hold." Do NOT give a recommendation. Your job is to confront the trader with their own rules so they can make a clear-headed decision.`;

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
      messages: [{ role: "user", content: userMessage }]
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
  new Notice("Failed to fetch exit check: " + e.message);
  return;
}

// ── Write into trade note — append as a dated entry ───────────
const now = tp.date.now("YYYY-MM-DD HH:mm");
const sectionHeader = "### Exit Checks\n";
const entry = "\n#### " + now + "\n" + response_text + "\n";

if (content.includes("### Exit Checks")) {
  // ── Section exists — insert right after header (newest first) ─
  const exitChecksIndex = content.indexOf("### Exit Checks");
  const afterHeader = content.indexOf("\n", exitChecksIndex) + 1;

  // Skip the instruction placeholder line if present
  let insertPoint = afterHeader;
  const nextLine = content.slice(afterHeader, content.indexOf("\n", afterHeader));
  if (nextLine.startsWith("*")) {
    insertPoint = content.indexOf("\n", afterHeader) + 1;
  }

  const updatedContent = content.slice(0, insertPoint) + entry + "\n" + content.slice(insertPoint);
  await app.vault.modify(file, updatedContent);
} else {
  // ── No Exit Checks section yet — create it before AI Critique
  const aiCritiqueIndex = content.indexOf("### AI Critique");
  if (aiCritiqueIndex > -1) {
    const updatedContent = content.slice(0, aiCritiqueIndex) + sectionHeader + entry + "\n---\n\n" + content.slice(aiCritiqueIndex);
    await app.vault.modify(file, updatedContent);
  } else {
    // No AI Critique section either — append at end
    const updatedContent = content + "\n\n---\n\n" + sectionHeader + entry;
    await app.vault.modify(file, updatedContent);
  }
}

new Notice("✅ Exit check added to trade note.", 6000);
%>
