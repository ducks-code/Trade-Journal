# 📊 Trading Dashboard

---

## 🔢 Summary Stats

```dataviewjs
const parsePNL = (val) => {
  if (val == null) return 0;
  const n = parseFloat(String(val).replace(/[^0-9.-]/g, ""));
  return isNaN(n) ? 0 : n;
};

const trades = dv.pages('"Trade Journal/Trades"').where(p => p.result === "win" || p.result === "loss");
const wins = trades.where(p => p.result === "win");
const losses = trades.where(p => p.result === "loss");
const total = trades.length;

const totalPNL = trades.array().reduce((a, p) => a + parsePNL(p.PNL), 0);
const avgWin = wins.length ? wins.array().reduce((a, p) => a + parsePNL(p.PNL), 0) / wins.length : 0;
const avgLoss = losses.length ? losses.array().reduce((a, p) => a + parsePNL(p.PNL), 0) / losses.length : 0;
const winRate = total ? ((wins.length / total) * 100).toFixed(1) : 0;
const profitFactor = avgLoss !== 0 ? Math.abs(avgWin / avgLoss).toFixed(2) : "∞";

dv.table(["Metric", "Value"], [
  ["Total Trades", total],
  ["Wins / Losses", `${wins.length} / ${losses.length}`],
  ["Win Rate", `${winRate}%`],
  ["Total PNL", `$${totalPNL.toFixed(2)}`],
  ["Avg Win", `$${avgWin.toFixed(2)}`],
  ["Avg Loss", `$${avgLoss.toFixed(2)}`],
  ["Profit Factor", profitFactor],
]);
```

---

## 📋 All Closed Trades

```dataviewjs
const parsePNL = (val) => {
  if (val == null) return 0;
  const n = parseFloat(String(val).replace(/[^0-9.-]/g, ""));
  return isNaN(n) ? 0 : n;
};

const trades = dv.pages('"Trade Journal/Trades"')
  .where(p => p.result === "win" || p.result === "loss")
  .sort(p => p.entry_date, "desc");

dv.table(
  ["Date", "Ticker", "Setup", "Size", "Port %", "Result", "PNL", "Emotion", "Followed Plan"],
  trades.map(t => [
    t.entry_date,
    t.file.link,
    t.setup_type,
    t.position_size ? "$" + parsePNL(t.position_size).toLocaleString() : "—",
    t.portfolio_size && t.position_size
      ? ((parsePNL(t.position_size) / parsePNL(t.portfolio_size)) * 100).toFixed(1) + "%"
      : t.position_pct ? t.position_pct + "%" : "—",
    t.result,
    t.PNL ? "$" + parsePNL(t.PNL).toFixed(2) : "—",
    t.emotion_pre,
    t.followed_plan
  ])
);
```

---

## 🎰 Win Rate by Setup Type

```dataviewjs
const parsePNL = (val) => {
  if (val == null) return 0;
  const n = parseFloat(String(val).replace(/[^0-9.-]/g, ""));
  return isNaN(n) ? 0 : n;
};

const trades = dv.pages('"Trade Journal/Trades"')
  .where(p => (p.result === "win" || p.result === "loss") && p.setup_type);

const grouped = {};
for (const t of trades) {
  const key = t.setup_type;
  if (!grouped[key]) grouped[key] = { wins: 0, total: 0, pnl: 0 };
  grouped[key].total++;
  grouped[key].pnl += parsePNL(t.PNL);
  if (t.result === "win") grouped[key].wins++;
}
const rows = Object.entries(grouped)
  .sort((a, b) => b[1].pnl - a[1].pnl)
  .map(([setup, d]) => [
    setup, d.total,
    `${((d.wins / d.total) * 100).toFixed(1)}%`,
    `$${d.pnl.toFixed(2)}`
  ]);
dv.table(["Setup Type", "Trades", "Win Rate", "Total PNL"], rows);
```

---

## 🧠 Emotion vs. Result

```dataviewjs
const parsePNL = (val) => {
  if (val == null) return 0;
  const n = parseFloat(String(val).replace(/[^0-9.-]/g, ""));
  return isNaN(n) ? 0 : n;
};

const trades = dv.pages('"Trade Journal/Trades"')
  .where(p => (p.result === "win" || p.result === "loss") && p.emotion_pre);

const grouped = {};
for (const t of trades) {
  const key = t.emotion_pre;
  if (!grouped[key]) grouped[key] = { wins: 0, total: 0, pnl: 0 };
  grouped[key].total++;
  grouped[key].pnl += parsePNL(t.PNL);
  if (t.result === "win") grouped[key].wins++;
}
const rows = Object.entries(grouped)
  .sort((a, b) => b[1].pnl - a[1].pnl)
  .map(([e, d]) => [
    e, d.total,
    `${((d.wins / d.total) * 100).toFixed(1)}%`,
    `$${d.pnl.toFixed(2)}`
  ]);
dv.table(["Pre-Trade Emotion", "Trades", "Win Rate", "Total PNL"], rows);
```

---

## 📏 Plan Adherence Impact

```dataviewjs
const parsePNL = (val) => {
  if (val == null) return 0;
  const n = parseFloat(String(val).replace(/[^0-9.-]/g, ""));
  return isNaN(n) ? 0 : n;
};

const isTrue = (val) => String(val).toLowerCase() === "true";
const isFalse = (val) => String(val).toLowerCase() === "false";

const trades = dv.pages('"Trade Journal/Trades"')
  .where(p => (p.result === "win" || p.result === "loss") && (isTrue(p.followed_plan) || isFalse(p.followed_plan)));
const yes = trades.where(p => isTrue(p.followed_plan));
const no = trades.where(p => isFalse(p.followed_plan));

const avgPNL = (arr) => arr.length ? arr.array().reduce((a, p) => a + parsePNL(p.PNL), 0) / arr.length : 0;
const winRate = (arr) => arr.length ? (arr.where(p => p.result === "win").length / arr.length * 100).toFixed(1) : "—";

dv.table(["Followed Plan?", "Trades", "Win Rate", "Avg PNL"], [
  ["✅ Yes", yes.length, `${winRate(yes)}%`, `$${avgPNL(yes).toFixed(2)}`],
  ["❌ No", no.length, `${winRate(no)}%`, `$${avgPNL(no).toFixed(2)}`],
]);
```

---

## 📊 Position Sizing Analysis

```dataviewjs
const parsePNL = (val) => {
  if (val == null) return 0;
  const n = parseFloat(String(val).replace(/[^0-9.-]/g, ""));
  return isNaN(n) ? 0 : n;
};

const trades = dv.pages('"Trade Journal/Trades"')
  .where(p => (p.result === "win" || p.result === "loss") && p.position_size && p.portfolio_size)
  .sort(p => parsePNL(p.position_size) / parsePNL(p.portfolio_size), "desc");

const rows = trades.map(t => {
  const pct = ((parsePNL(t.position_size) / parsePNL(t.portfolio_size)) * 100).toFixed(1);
  return [
    t.entry_date,
    t.file.link,
    "$" + parsePNL(t.portfolio_size).toLocaleString(),
    "$" + parsePNL(t.position_size).toLocaleString(),
    pct + "%",
    t.result,
    "$" + parsePNL(t.PNL).toFixed(2)
  ];
});

dv.table(["Date", "Ticker", "Portfolio", "Position", "% of Port", "Result", "PNL"], rows);
```

---

## ⚠️ Recurring Mistakes

```dataviewjs
const trades = dv.pages('"Trade Journal/Trades"')
  .where(p => p.result === "loss" && p.mistake);

const mistakeCount = {};
for (const t of trades) {
  const mistakes = String(t.mistake).split(",").map(m => m.trim());
  for (const m of mistakes) {
    if (m) mistakeCount[m] = (mistakeCount[m] ?? 0) + 1;
  }
}
const rows = Object.entries(mistakeCount)
  .sort((a, b) => b[1] - a[1])
  .map(([mistake, count]) => [mistake, count]);
dv.table(["Mistake", "Occurrences"], rows);
```

---

## 💸 Biggest Losses

```dataviewjs
const parsePNL = (val) => {
  if (val == null) return 0;
  const n = parseFloat(String(val).replace(/[^0-9.-]/g, ""));
  return isNaN(n) ? 0 : n;
};

const trades = dv.pages('"Trade Journal/Trades"')
  .where(p => p.result === "loss")
  .sort(p => parsePNL(p.PNL), "asc")
  .slice(0, 10);

dv.table(
  ["Date", "Ticker", "Setup", "PNL", "Emotion", "Mistake"],
  trades.map(t => [t.entry_date, t.file.link, t.setup_type, "$" + parsePNL(t.PNL).toFixed(2), t.emotion_pre, t.mistake])
);
```

---

## 🏆 Biggest Wins

```dataviewjs
const parsePNL = (val) => {
  if (val == null) return 0;
  const n = parseFloat(String(val).replace(/[^0-9.-]/g, ""));
  return isNaN(n) ? 0 : n;
};

const trades = dv.pages('"Trade Journal/Trades"')
  .where(p => p.result === "win")
  .sort(p => parsePNL(p.PNL), "desc")
  .slice(0, 10);

dv.table(
  ["Date", "Ticker", "Setup", "PNL", "Emotion"],
  trades.map(t => [t.entry_date, t.file.link, t.setup_type, "$" + parsePNL(t.PNL).toFixed(2), t.emotion_pre])
);
```

---

## 💰 Profit Extraction

```dataviewjs
const parsePNL = (val) => {
  if (val == null) return 0;
  const n = parseFloat(String(val).replace(/[^0-9.-]/g, ""));
  return isNaN(n) ? 0 : n;
};

const wins = dv.pages('"Trade Journal/Trades"').where(p => p.result === "win");
const totalWinPNL = wins.array().reduce((a, p) => a + parsePNL(p.PNL), 0);
const totalStabled = wins.array().reduce((a, p) => a + parsePNL(p.stabled_amount), 0);
const leftOnTable = totalWinPNL - totalStabled;
const extractionRate = totalWinPNL > 0 ? ((totalStabled / totalWinPNL) * 100).toFixed(1) : 0;

dv.table(["Metric", "Value"], [
  ["Total Winning PNL", `$${totalWinPNL.toFixed(2)}`],
  ["Total Stabled", `$${totalStabled.toFixed(2)}`],
  ["Left on Table", `$${leftOnTable.toFixed(2)}`],
  ["Extraction Rate", `${extractionRate}%`],
]);
```

### Per-Trade Stabling

```dataviewjs
const parsePNL = (val) => {
  if (val == null) return 0;
  const n = parseFloat(String(val).replace(/[^0-9.-]/g, ""));
  return isNaN(n) ? 0 : n;
};

const wins = dv.pages('"Trade Journal/Trades"').where(p => p.result === "win");
const rows = wins.array()
  .map(p => {
    const pnl = parsePNL(p.PNL);
    const stabled = parsePNL(p.stabled_amount);
    const pct = pnl > 0 ? ((stabled / pnl) * 100).toFixed(1) + "%" : "—";
    return [p.file.link, p.ticker, `$${pnl.toFixed(2)}`, `$${stabled.toFixed(2)}`, pct];
  })
  .sort((a, b) => parsePNL(b[3]) - parsePNL(a[3]));
dv.table(["Note", "Ticker", "PNL", "Stabled", "Extraction %"], rows);
```

### Stabling Rate by Setup Type

```dataviewjs
const parsePNL = (val) => {
  if (val == null) return 0;
  const n = parseFloat(String(val).replace(/[^0-9.-]/g, ""));
  return isNaN(n) ? 0 : n;
};

const wins = dv.pages('"Trade Journal/Trades"').where(p => p.result === "win" && p.setup_type);
const grouped = {};
for (const t of wins) {
  const key = t.setup_type;
  if (!grouped[key]) grouped[key] = { pnl: 0, stabled: 0, count: 0 };
  grouped[key].pnl += parsePNL(t.PNL);
  grouped[key].stabled += parsePNL(t.stabled_amount);
  grouped[key].count++;
}
const rows = Object.entries(grouped)
  .sort((a, b) => b[1].pnl - a[1].pnl)
  .map(([setup, d]) => {
    const pct = d.pnl > 0 ? ((d.stabled / d.pnl) * 100).toFixed(1) + "%" : "—";
    return [setup, d.count, `$${d.pnl.toFixed(2)}`, `$${d.stabled.toFixed(2)}`, pct];
  });
dv.table(["Setup Type", "Wins", "Total PNL", "Total Stabled", "Extraction %"], rows);
```
---

## 🔓 Open Trades

```dataview
TABLE entry_date, ticker, setup_type, trade_confidence, entry_price, position_size, position_pct
FROM "Trade Journal/Trades"
WHERE trade_status = "open"
SORT entry_date DESC
```
