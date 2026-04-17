<%*
// ── Load API key from Templates/Config.md (gitignored) ────────
const CLAUDE_API_KEY = await (async function loadApiKey() {
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
if (!CLAUDE_API_KEY) { return; }

// ── Validate active file is a pre-trade checklist ─────────────
const checklistFile = app.workspace.getActiveFile();
if (!checklistFile) { new Notice("No active file."); return; }

const checklistPattern = /^[A-Z0-9]+ \d{4}-\d{2}-\d{2} \d{2}-\d{2} PRE-TRADE$/;
if (!checklistPattern.test(checklistFile.basename)) {
  new Notice("⛔ Open a Pre-Trade Checklist note first (Alt + P), then run research.", 8000);
  return;
}

const ticker = checklistFile.basename.split(" ")[0].toUpperCase();
const inputNotePath = "Trade Journal/Trades/Pre-Trades/" + ticker + " TWEET INPUT.md";

// ── PASS 1: input note doesn't exist yet — create it ─────────
const inputFileExists = app.vault.getAbstractFileByPath(inputNotePath);

if (!inputFileExists) {
  const inputNoteContent = [
    "# " + ticker + " — Tweet Input",
    "",
    "> Paste your tweet(s) or thread text below the divider.",
    "> Save this note, navigate back to the checklist, then press Alt + R again to generate research.",
    "> This note will be deleted automatically after research is generated.",
    "> Leave everything below the divider blank if you have no tweets to add.",
    "",
    "---",
    "",
    ""
  ].join("\n");

  const inputFile = await app.vault.create(inputNotePath, inputNoteContent);
  await app.workspace.getLeaf().openFile(inputFile);
  new Notice("📋 Paste your tweets below the divider, save, then navigate back to the checklist and press Alt + R again.", 8000);
  return;
}

// ── PASS 2: input note exists — read it and generate research ──

// ── Check research hasn't already been inserted ───────────────
const existingContent = await app.vault.read(checklistFile);
if (existingContent.includes("## Research Note —")) {
  const overwrite = await tp.system.prompt("Research already exists in this checklist. Overwrite? (yes/no)");
  if (overwrite?.toLowerCase() !== "yes") {
    await app.vault.delete(app.vault.getAbstractFileByPath(inputNotePath));
    new Notice("Cancelled.");
    return;
  }
}

// ── Read tweet content from input note ────────────────────────
const inputFileRef = app.vault.getAbstractFileByPath(inputNotePath);
const rawInput = await app.vault.read(inputFileRef);
let tweetText = "";
const separatorIndex = rawInput.indexOf("---\n\n");
if (separatorIndex !== -1) {
  tweetText = rawInput.slice(separatorIndex + 5).trim();
}

// ── Delete input note ─────────────────────────────────────────
await app.vault.delete(inputFileRef);

// ── Get contract address ──────────────────────────────────────
const caInput = await tp.system.prompt("Contract address (CA):");
if (!caInput) { new Notice("Cancelled."); return; }
const ca = caInput.trim().replace(/\s+/g, "");
if (!ca) { new Notice("Invalid contract address."); return; }

// ── Get portfolio size ────────────────────────────────────────
const portfolioInput = await tp.system.prompt("Your portfolio size in USD (e.g. 7500):");
if (!portfolioInput) { new Notice("Cancelled."); return; }
const portfolioSize = parseFloat(portfolioInput.replace(/[^0-9.]/g, ""));
if (isNaN(portfolioSize) || portfolioSize <= 0) { new Notice("Invalid portfolio size."); return; }

new Notice("Fetching token data from Dexscreener...");

// ── Fetch Dexscreener data by CA ──────────────────────────────
let tokenData = null;
let dexError = null;

try {
  const searchUrl = "https://api.dexscreener.com/latest/dex/tokens/" + ca;
  const response = await fetch(searchUrl);
  const data = await response.json();

  if (data.pairs && data.pairs.length > 0) {
    const pairs = [...data.pairs];
    pairs.sort((a, b) => parseFloat(b.volume?.h24 || 0) - parseFloat(a.volume?.h24 || 0));
    tokenData = pairs[0];
  }
} catch(e) {
  dexError = e.message;
}

// ── Build token summary ───────────────────────────────────────
let tokenSummary = "";
let chain = "unknown";

if (tokenData) {
  chain = (tokenData.chainId || "unknown").toLowerCase();
  const mc = tokenData.marketCap ? "$" + parseFloat(tokenData.marketCap).toLocaleString() : "N/A";
  const price = tokenData.priceUsd ? "$" + parseFloat(tokenData.priceUsd).toFixed(8) : "N/A";
  const vol24h = tokenData.volume?.h24 ? "$" + parseFloat(tokenData.volume.h24).toLocaleString() : "N/A";
  const liq = tokenData.liquidity?.usd ? "$" + parseFloat(tokenData.liquidity.usd).toLocaleString() : "N/A";
  const change5m = tokenData.priceChange?.m5 != null ? tokenData.priceChange.m5 + "%" : "N/A";
  const change1h = tokenData.priceChange?.h1 != null ? tokenData.priceChange.h1 + "%" : "N/A";
  const change6h = tokenData.priceChange?.h6 != null ? tokenData.priceChange.h6 + "%" : "N/A";
  const change24h = tokenData.priceChange?.h24 != null ? tokenData.priceChange.h24 + "%" : "N/A";
  const buys = tokenData.txns?.h24?.buys || 0;
  const sells = tokenData.txns?.h24?.sells || 0;
  const tokenName = tokenData.baseToken?.name || ticker;
  const tokenSymbol = tokenData.baseToken?.symbol || ticker;

  tokenSummary = [
    "DEXSCREENER DATA — " + tokenName + " (" + tokenSymbol + ") on " + chain.toUpperCase(),
    "CA: " + ca,
    "Price: " + price,
    "Market Cap: " + mc,
    "24h Volume: " + vol24h,
    "Liquidity: " + liq,
    "Price Change — 5m: " + change5m + " | 1h: " + change1h + " | 6h: " + change6h + " | 24h: " + change24h,
    "24h Transactions — Buys: " + buys + " / Sells: " + sells,
    "Pair Address: " + (tokenData.pairAddress || "N/A")
  ].join("\n");
} else {
  tokenSummary = "Dexscreener data unavailable for CA: " + ca + (dexError ? " (error: " + dexError + ")" : " — CA may be incorrect or token not yet listed.");
}

// ── Build Claude prompt ───────────────────────────────────────
const tweetSection = tweetText.length > 0
  ? "TWEET / THREAD CONTENT:\n" + tweetText
  : "No tweet content provided.";

const userMessage = [
  "TICKER: " + ticker,
  "CA: " + ca,
  "CHAIN: " + chain,
  "PORTFOLIO SIZE: $" + portfolioSize.toLocaleString(),
  "",
  tokenSummary,
  "",
  tweetSection
].join("\n");

const SYSTEM = `You are a crypto trading analyst specialising in micro-cap narrative and momentum plays on Solana and Base. You help traders assess whether a trade is worth taking based on on-chain data and CT context.

Given token data and optional tweet context, produce a structured research note in this EXACT markdown format with no extra text before or after:

## Research Note — {TICKER} // {DATE}

### Token Snapshot
| Metric | Value |
|--------|-------|
| Market Cap | ... |
| 24h Volume | ... |
| Liquidity | ... |
| 5m / 1h / 6h / 24h | ... |
| 24h Buys / Sells | ... |

### Thesis Assessment
<2-3 sentences: what is the core narrative or edge here based on the data and tweets provided? Is there a real reason this asset could move?>

### Catalyst
<1-2 sentences: what is the specific trigger for a move? Is it confirmed or anticipated?>

### Invalidation
<1-2 sentences: what specific, enforceable condition means the thesis is wrong?>

### Checklist Pre-Assessment
Rate each area as ✅ Clear / ⚠️ Caution / ❌ Concern and give one sentence of reasoning:

- **Thesis clarity:** 
- **Catalyst confirmation:** 
- **Chart / PA support:** 
- **Risk conditions:** 
- **Emotional suitability:** (note: always flag if this should be verified by the trader)

### Position Sizing Suggestion
Based on thesis conviction, token liquidity, and portfolio size of $` + portfolioSize.toLocaleString() + `:

- **Suggested allocation:** X% of portfolio
- **Dollar amount:** $X
- **Reasoning:** <1-2 sentences on why this size is appropriate given liquidity and conviction>

### Go / No-Go
**Recommendation: GO / NO-GO / CONDITIONAL**
> <2-3 sentences: clear verdict and the single most important reason for it. If CONDITIONAL, state exactly what condition needs to be met before entering.>

Scoring guidance:
- GO = strong thesis, confirmed catalyst, healthy on-chain metrics
- CONDITIONAL = thesis exists but catalyst unconfirmed or metrics weak — state the condition
- NO-GO = no clear edge, low liquidity, or thesis is pure speculation`;

new Notice("Generating research...");

// ── Call Claude API ───────────────────────────────────────────
let research = "";
try {
  const response = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "x-api-key": CLAUDE_API_KEY,
      "anthropic-version": "2023-06-01",
      "anthropic-dangerous-direct-browser-access": "true"
    },
    body: JSON.stringify({
      model: "claude-sonnet-4-20250514",
      max_tokens: 1500,
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
  research = (data.content || []).map(b => b.text || "").join("").trim();

} catch(e) {
  new Notice("Failed to fetch research: " + e.message);
  return;
}

// ── Build research block ──────────────────────────────────────
const researchBlock = [
  "---",
  "",
  research,
  "",
  "### Raw Token Data",
  "```",
  tokenSummary,
  "```",
  "",
  "> **Research generated by AI — verify independently before trading. Fill out the checklist below based on your own assessment.**",
  "",
  "---",
  ""
].join("\n");

// ── Insert research at top of checklist after frontmatter ─────
let currentContent = await app.vault.read(checklistFile);

// ── Remove existing research block if overwriting ─────────────
if (currentContent.includes("## Research Note —")) {
  const start = currentContent.indexOf("\n---\n\n## Research Note —");
  const end = currentContent.indexOf("\n---\n", start + 5);
  if (start !== -1 && end !== -1) {
    currentContent = currentContent.slice(0, start) + currentContent.slice(end + 5);
  }
}

// ── Find insertion point after frontmatter ────────────────────
const frontmatterEnd = currentContent.indexOf("---", 3);
if (frontmatterEnd === -1) { new Notice("Could not find frontmatter in checklist."); return; }

let insertAt = frontmatterEnd + 3;
while (insertAt < currentContent.length && currentContent[insertAt] === "\n") {
  insertAt++;
}

const updatedContent = currentContent.slice(0, insertAt) + researchBlock + currentContent.slice(insertAt);
await app.vault.modify(checklistFile, updatedContent);

new Notice("✅ Research inserted into checklist for " + ticker, 6000);
%>
