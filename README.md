# SOS POS Sales Loader

A Tampermonkey userscript for **app.sospos.com.au** that turns a pasted block of rows from your daily sales sheet into finished tickets in SOSPOS — creating customers, adding line items, taking payment, dismissing the receipt prompt, and handing you back a tidy list of ticket numbers to paste into your sheet.

- **Version:** 2.5
- **Runs on:** `https://app.sospos.com.au/*`
- **Requires:** [Tampermonkey](https://www.tampermonkey.net/) (or a compatible userscript manager) with `GM_setValue` / `GM_getValue` permissions

---

## What it does, in one breath

You copy a day's worth of rows out of your spreadsheet, paste them into the loader, and it sorts them into named customers, walk-ins, rows that need a human, and rows that already have a ticket. It then builds each one in SOSPOS — making the customer, entering the device/repair line items and prices, checking out, paying, and skipping the receipt dialog — and collects every ticket number into a copy-ready list.

---

## Installation

1. Install the Tampermonkey browser extension.
2. Open the Tampermonkey dashboard → **Create a new script** (or **Utilities → Import**).
3. Paste in the full contents of `sos-pos-sales-loader.user.js`, replacing anything already there.
4. Save (Ctrl/Cmd+S).
5. Open or hard-refresh **app.sospos.com.au** (Ctrl+Shift+R).

You should see a round teal **🏷️ button** in the bottom-left corner of the page. Click it to open the panel.

> **Updating:** when you replace the script with a newer version, hard-refresh the page afterwards. You can confirm the new build loaded by checking the version number in Tampermonkey — it should read **2.5**.

---

## The interface

Clicking the floating 🏷️ button opens a panel with three tabs.

### 🛠 Build
Where you paste rows and run the job.
- **Clear all** (top-right) — wipes the current paste, preview, and results at any time.
- **Paste area** — click it, then press **Ctrl+V** to paste your rows.
- **Preview** — shows how each row was interpreted, grouped into sections.
- **Start / Next** — builds the tickets.
- **Progress bar + status line** — shows what it's doing.

### 📋 Results
The pile of ticket numbers captured during the run (see [Results](#results) below).

### ⚙ Settings
All the knobs (see [Settings](#settings) below). Settings are saved to your browser and persist between sessions.

---

## How to use it

1. In your sheet, select and copy the rows you want to load (tab-separated — a normal spreadsheet copy).
2. Open the loader → **Build** tab → click the paste box → **Ctrl+V**.
3. Review the **preview**. Rows are grouped into four buckets:
   - **Named tickets** (built first)
   - **Walk-ins** (built after named)
   - **Needs manual description** (deferred — you type the description)
   - **Existing tickets** (already have a number — skipped, shown for reference)
4. For any **manual** rows, type the device + repair into the purple input field.
5. Click **Start**. Behaviour from here depends on your [payment settings](#settings).
6. When it finishes, open **Results**, fix any ticket numbers if needed, and **Copy for Sheets**.

---

## How rows are interpreted

### Column map

The script reads tab-separated columns by position. By spreadsheet letter:

| Column | Meaning            |
|--------|--------------------|
| **C**  | Ticket number      |
| **E**  | Cash amount        |
| **F**  | EFTPOS amount      |

**Description** is taken from the cell immediately **after** a cell containing the word `PIN`. If no `PIN` cell is found, it falls back to the last non-empty cell in the row.

Header rows and junk are ignored automatically (rows whose description is things like `Date`, `Ticket`, `Status`, `Customer`, day-of-week abbreviations, etc.).

### The four buckets

**1. Existing tickets** — if column C already contains a real ticket number (a letter followed by 3+ digits, e.g. `A1234`), the row is **skipped** and shown in orange. It is *not* rebuilt, but it still appears in Results so your list stays complete.

**2. Walk-ins** — if the description starts with `walk-in` / `walkin`, it's treated as a walk-in sale. No customer is created. The "Walkin" prefix can be stripped from the description (see Settings).

**3. Named tickets** — everything the parser can resolve into a customer + device. The smart parser pulls out the name, phone, and email and keeps only the device + repair in the line-item description.

**4. Needs manual review** — rows the parser can't safely resolve are deferred for you to complete. A row lands here when:
   - no device/repair text could be extracted, or
   - the note contains a **password** (kept out of the description for safety), or
   - the note is very long with no recognisable device.

   These show in purple with the raw note and an input box. Fill in the description, then build.

### The smart note parser

When the parser is **on**, it cleans messy notes into a clean device + repair description and lifts out the contact details:

- **Phone numbers** — normalised to Australian formats. Mobiles become `04xx xxx xxx`; landlines become `(0x) xxxx xxxx`. Bare 8-digit landlines are given a default `02` area code. `61…` international prefixes are converted to `0…`.
- **Emails** — detected and removed from the description.
- **Name** — taken from the text before the first dash or phone number (with labels like `CALL`, `TEXT ONLY`, `MOB`, `PH` stripped).
- **Devices** — recognises iPhone, Samsung/Galaxy (S/A/Z Flip/Fold/Note), Oppo, Pixel, MacBook, iPad, iPod, and Flip/Fold, including tiers like Pro, Pro Max, Plus, mini, FE, Ultra.
- **Multiple devices in one note** are split into separate line items on the same ticket.

When the parser is **off**, the script uses a simple fallback: it splits the note on the first phone-like number into name / phone / description, and otherwise uses the raw text as the description.

---

## Settings

| Setting | Options | What it does |
|---|---|---|
| **Payment** | *Stop at Checkout — I take payment* / *Auto-pay each ticket* | Whether the script pays for you or stops so you can handle payment manually. |
| **After each ticket** | *Pause — wait before the next one* / *Don't wait — run them all in a row* | In auto-pay mode, whether it pauses for you to click **Next** between tickets, or runs the whole batch unattended. |
| **Line item price** | *Cash + EFTPOS added together* / *EFTPOS column only* / *Cash column only* | Which column(s) become the line-item price. |
| **Step delay (ms)** | number (min 100, default 350) | Pause between each automated action. Increase if SOSPOS is slow to respond and the script gets ahead of the page. |
| **Strip "Walkin" prefix** | Yes / No | Remove a leading `Walkin` from walk-in descriptions. |
| **Smart note parser** | On / Off | Toggle the description parser described above. |

Click **Save settings** to persist. If you change settings while rows are loaded, the preview re-parses automatically.

> **Tip:** the safe way to start is *Auto-pay each ticket* + *Pause after each*. You watch each one complete and click **Next**, so you can catch any oddities before committing to a full unattended run. Once you trust it, switch to *Don't wait*.

---

## What happens during a build

For each ticket the script:

1. Opens the **Sale** tab.
2. **Named / manual:** creates the customer (name, phone, email). **Walk-in:** clicks the Walk-in button.
3. Enters each line item — description and price — using **Add another item** for additional lines.
4. **Auto-pay mode only:** clicks **Checkout**, fills the Cash and/or EFTPOS split (based on your price setting), clicks **Complete Payment**, then automatically clicks **Skip** on the "Would you like a receipt?" dialog.
5. Captures the resulting ticket number into Results.

### Duplicate customers

If SOSPOS shows a "possible duplicate" dialog when creating a customer, the script handles it automatically without pausing — it prefers **Skip this customer**, otherwise uses the first match, otherwise **Create new anyway**. It never stops the run to ask.

> If you'd rather it *reuse* the matched customer instead of skipping, that's a small change — see the note in the changelog.

### Build order

Named tickets are built first, then walk-ins, then any manual rows you've filled in. Manual rows with an empty description are skipped (and listed as blanks in Results so you can finish them later).

---

## Results

Every row from your paste ends up in the Results tab, even skipped ones:

- **Built tickets** — with the ticket number the script detected.
- **Existing tickets** — shown in yellow, carrying the number from column C.
- **Skipped / manual rows** — shown in purple with a **blank, editable** ticket field for you to fill in.

From here:

- Each ticket number is **editable** — fix anything the auto-capture got wrong before copying.
- Each row has its own **⧉ copy button** to copy just that one ticket number.
- **📋 Copy for Sheets** copies the whole list, tab-separated as `ticket ⇥ name`, ready to paste back into your spreadsheet.
- **Clear** empties the list.

---

## Troubleshooting

**The 🏷️ button doesn't appear.**
Confirm the script is enabled in Tampermonkey and that you're on `app.sospos.com.au`. Hard-refresh. If it still doesn't show, open the browser console and run `document.getElementById('sost-fab')` — if that returns `null`, the script isn't running (check the version/match); if it returns an element, it exists and is just a positioning issue.

**The Settings list looks cut off.**
The panel is capped to your window height and scrolls internally, with the header and tabs pinned — scroll down inside the panel to reach everything, including **Save settings**. If your browser window is very short, make it taller.

**It builds too fast / clicks before the page is ready.**
Increase **Step delay (ms)** in Settings (try 500–700).

**A build stops with an error.**
The status line shows the reason (e.g. a button it couldn't find). Fix the situation in SOSPOS, then click the button again to resume from where it stopped. Full errors are also logged to the browser console under `[SOS Loader]`.

**Ticket numbers in Results look wrong or blank.**
The capture looks for ticket numbers in the form *letter + digits* (e.g. `A1234`). The receipt dialog shows a **tax-invoice** number like `INV-2606-AAA-0023`, which is a different thing — if your build is picking up the wrong value, the ticket number may live in a part of the page the grabber isn't reading. You can always type the correct number into the editable Results field.

---

## Privacy & data

The script runs entirely in your browser. The only thing it stores is your settings, saved locally via Tampermonkey's storage (`GM_setValue`). Pasted sheet data lives only in memory for the current session and is cleared when you click **Clear all** or reload the page. Nothing is sent anywhere except the normal actions you'd perform in SOSPOS yourself.

---

## Changelog

- **2.5** — Auto-skip the "Would you like a receipt?" dialog after payment; defensive receipt-dialog clearing at the start of each job.
- **2.4** — Panel scrolls as a unit with sticky header/tabs so Settings can never be clipped off-screen.
- **2.3** — All rows (built, existing, skipped) listed in Results; per-row copy buttons; auto-handle duplicate-customer dialog ("Skip this customer", no pausing); simplified payment to Manual vs Auto-pay plus a "wait after each" toggle; always-visible "Clear all" on the Build tab.
- **2.1** — Hardened the floating button so it re-mounts if the app re-renders, and raised its stacking order so it can't be hidden.
- **2.0** — Smart note parser (strips name/phone/email from the description), skips rows with existing tickets, defers unresolvable rows for manual entry.

> **Customisation note:** to make the duplicate-customer dialog *reuse* the matched customer instead of skipping, change the handler to prefer the "Use this customer" button over "Skip this customer".
