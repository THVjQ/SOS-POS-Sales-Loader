# SOS POS Walk-in Sales Loader

**Version:** 1.0 · **Site:** app.sospos.com.au

Paste rows straight from your sales sheet and bulk-build **Walk-in sales** in SOS POS. Rows are grouped by ticket number, each row becomes a line item, and the script stops at Checkout so you take payment yourself. Saves the manual click-through when you're entering a batch of walk-in transactions.

---

## What It Does

1. Adds a floating **🏷️ button** to SOS POS, docked next to the app's own bottom-left buttons.
2. Copy your rows from the sheet (`Ctrl+C`).
3. Paste into the panel — rows are **grouped by ticket #**.
4. Click **Build** — for each ticket it switches to the **Sale** tab, clicks **Walk-in**, and adds every row as a line item with its price.
5. It **stops at Checkout**. You review the total and take payment manually.
6. After payment, click **Next Ticket →** to build the following group.

> No money is ever moved automatically. The script only builds the cart up to the Checkout step.

---

## How Rows Are Read

Paste is tab-separated (straight from a spreadsheet). Columns map like this:

| Field            | Source                                          |
| ---------------- | ----------------------------------------------- |
| Ticket # (group) | Col C                                           |
| Cash amount      | Col E                                            |
| EFTPOS amount    | Col F                                            |
| Description      | Last cell in the row (the text after `PIN`)     |
| Line price       | Cash + EFTPOS added together *(configurable)*   |
| Payment method   | Cash / EFTPOS / Split — shown as a tag only     |

**Example** — these five rows become **two tickets**:

```
17.06.26    A2889   Paid & Collected        40   ...   Walkin - H/G
17.06.26    A2892   Paid & Collected        50   ...   Walkin C T/G x2
17.06.26    A2892   Paid & Collected        60   ...   walkin PP zfold 6 front screen
17.06.26    A2892   Paid & Collected   55        ...   Walkin 0- Cable + wal plug
17.06.26    A2892   Paid & Collected        20   ...   Walkin - cable 1m
```

- **A2889** → 1 item, $40
- **A2892** → 4 items, $50 + $60 + $55 + $20 = **$185**

---

## Install

1. Install [Tampermonkey](https://www.tampermonkey.net/) in Chrome.
2. Open the Tampermonkey dashboard → **Create a new script**.
3. Paste the contents of `sos-pos-walkin-loader.user.js` and save (`Ctrl+S`).
4. Open SOS POS — the **🏷️ button** appears in the bottom-left, just right of the existing buttons.

---

## Settings

Open the panel → **⚙ Settings**:

- **Line item price** — Cash + EFTPOS added together (default), EFTPOS only, or Cash only.
- **Step delay (ms)** — pause between actions. Raise it if the page is slow and fields get skipped.
- **Strip "Walkin" prefix** — removes a leading `Walkin` / `Walk-in` from each description.

Settings are saved across sessions via Tampermonkey storage.

---

## Notes

- The **ticket number in the paste is only used for grouping** — SOS POS assigns its own new number to each walk-in sale.
- The **payment method tag** (Cash / EFTPOS / Split) is a reminder for when you check out. The script does not select a method or process payment.
- The **$60 in the description** and the **LR / DP codes** are ignored.
- Wrong columns? Edit the `COL` map at the top of the script.

---

## Troubleshooting

- **A step fails / a field is skipped** — open the console (`F12`) and look for `[SOS Walk-in]` messages; they name the selector that didn't match. Raising the step delay in Settings often fixes timing issues.
- **Button is in the wrong spot** — it measures the app's bottom-left buttons at load and on resize. If detection misses, it falls back to a fixed position.
- **Nothing parses** — check the paste is tab-separated and the column map matches your sheet.

---

## Requirements

- `GM_setValue` / `GM_getValue` grants (already in the script header) — used to remember your settings.
