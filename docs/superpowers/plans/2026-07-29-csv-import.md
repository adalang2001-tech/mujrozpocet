# Import CSV výpisu z účtu s automatickou kategorizací Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let the user upload a bank statement CSV, auto-suggest a category per payment from the merchant name, show a review modal to confirm/fix categories and exclude rows (duplicates, internal transfers), then save the confirmed rows as normal transactions.

**Architecture:** Everything lives in the single existing `index.html` file (no build step, no backend — matches the current app architecture). CSV parsing uses PapaParse loaded from CDN, same pattern as the existing Tailwind/Chart.js/MSAL `<script src>` tags. New logic is added as plain functions inside the existing `<script>` block, next to the code they relate to (constants near `CATEGORY_OPTIONS`, storage helpers near `saveTransactions`, render/UI functions near `renderTable`).

**Tech Stack:** Vanilla JS, Tailwind (CDN), PapaParse 5.x (CDN), `localStorage` for persistence, existing OneDrive sync (`scheduleOneDriveSync`/`syncWithOneDrive`) reused unchanged.

## Global Constraints

- Single file: all markup and JS changes go into `index.html`. Do not create new files or a build step.
- No new transaction categories. Every mapping in `MERCHANT_CATEGORY_MAP` must resolve to one of the existing `CATEGORY_OPTIONS` values (`income: Plat, Ostatní` / `expense: Jídlo, Bydlení, Zábava, Doprava, Zdraví, Oblečení, Vzdělání, Investice, Ostatní`).
- **No automated test runner exists in this project** (no `package.json`, no Node available in this environment). "Tests" in this plan mean: open `index.html` in the Browser pane (`mcp__Claude_Browser__preview_start` / `navigate`), then run assertions in the page with `mcp__Claude_Browser__javascript_tool`. A step that says "run the test" means execute the given JS snippet via `javascript_tool` and check the printed result against the stated expectation.
- Keep the visual style consistent with the rest of the dark theme (`bg-slate-800`, `border-slate-700`, `text-slate-100/300/400`, `bg-rose-600` for destructive/expense accents, `bg-amber-900`/`text-amber-200` for warnings — same classes already used in the HTTPS warning box).
- Preserve existing behavior: `addTransaction`, `deleteTransaction`, OneDrive sync, and the 33/33/33 strategy math must keep working exactly as before.

---

### Task 1: `externalId` support in the transaction data model

**Files:**
- Modify: `index.html:542-554` (`normalizeTransactions`)

**Interfaces:**
- Produces: `normalizeTransactions(raw)` now preserves an optional `externalId: string | undefined` field on every returned transaction object, in addition to the existing `id, amount, type, category, date, note`.

- [ ] **Step 1: Write the failing check**

Open `index.html` in the Browser pane and run this via `javascript_tool`:

```js
JSON.stringify(normalizeTransactions([
  { id: 'a1', amount: 10, type: 'expense', category: 'Jídlo', date: '2026-01-01', note: '', externalId: 'ext-123' }
]))
```

Expected right now (FAILS the requirement): the result does **not** contain `"externalId":"ext-123"` because `normalizeTransactions` doesn't copy that field yet.

- [ ] **Step 2: Implement**

In `index.html`, change `normalizeTransactions` from:

```js
    function normalizeTransactions(raw) {
      if (!Array.isArray(raw)) return [];
      return raw.filter(t =>
        t && t.amount > 0 && ['income', 'expense'].includes(t.type) && t.category && t.date
      ).map(t => ({
        id: t.id || generateId(),
        amount: parseFloat(t.amount),
        type: t.type,
        category: t.category,
        date: t.date,
        note: t.note || '',
      }));
    }
```

to:

```js
    function normalizeTransactions(raw) {
      if (!Array.isArray(raw)) return [];
      return raw.filter(t =>
        t && t.amount > 0 && ['income', 'expense'].includes(t.type) && t.category && t.date
      ).map(t => ({
        id: t.id || generateId(),
        amount: parseFloat(t.amount),
        type: t.type,
        category: t.category,
        date: t.date,
        note: t.note || '',
        externalId: t.externalId || undefined,
      }));
    }
```

(`JSON.stringify` drops `undefined` properties automatically, so manually-added transactions — which never set `externalId` — are unaffected in `localStorage`/OneDrive payloads.)

- [ ] **Step 3: Run the check again**

Same snippet as Step 1. Expected: result now contains `"externalId":"ext-123"`.

Also run, to confirm manual transactions are unaffected:

```js
JSON.stringify(normalizeTransactions([
  { id: 'a2', amount: 5, type: 'income', category: 'Plat', date: '2026-01-01', note: '' }
]))
```

Expected: no `externalId` key present at all in the JSON (not even `"externalId":null`).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add externalId field to transaction data model for CSV import dedup"
```

---

### Task 2: CSV amount and date parsing helpers

**Files:**
- Modify: `index.html` — add new functions near `formatDate` (`index.html:562-565`)

**Interfaces:**
- Produces: `parseCsvAmount(raw: string): number` (signed float, e.g. `-129.08`), `csvDateToIso(raw: string): string | null` (returns `YYYY-MM-DD` zero-padded, or `null` if unparsable).

- [ ] **Step 1: Write the failing check**

```js
typeof parseCsvAmount
```

Expected: `"undefined"` (function doesn't exist yet).

- [ ] **Step 2: Implement**

Add these two functions right after `formatDate` in `index.html`:

```js
    function parseCsvAmount(raw) {
      if (raw == null) return NaN;
      const cleaned = String(raw).trim().replace(/\s/g, '').replace(',', '.');
      return parseFloat(cleaned);
    }

    function csvDateToIso(raw) {
      if (!raw) return null;
      const match = String(raw).trim().match(/^(\d{1,2})\.(\d{1,2})\.(\d{4})$/);
      if (!match) return null;
      const [, d, m, y] = match;
      return `${y}-${m.padStart(2, '0')}-${d.padStart(2, '0')}`;
    }
```

- [ ] **Step 3: Run the checks**

```js
JSON.stringify({
  a1: parseCsvAmount('-129,08'),
  a2: parseCsvAmount('1 234,50'),
  a3: parseCsvAmount('50'),
  d1: csvDateToIso('29.07.2026'),
  d2: csvDateToIso('5.1.2026'),
  d3: csvDateToIso('not a date'),
})
```

Expected: `{"a1":-129.08,"a2":1234.5,"a3":50,"d1":"2026-07-29","d2":"2026-01-05","d3":null}`

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add CSV amount and date parsing helpers"
```

---

### Task 3: Merchant-to-category mapping and confidence scoring

**Files:**
- Modify: `index.html` — add `MERCHANT_CATEGORY_MAP` constant near `CATEGORY_OPTIONS` (`index.html:356-359`), add `categorizeTransaction` function near it.

**Interfaces:**
- Consumes: `CATEGORY_OPTIONS` (existing, `index.html:356-359`).
- Produces: `MERCHANT_CATEGORY_MAP: Record<string, string>` (lowercase keyword → existing category name), `categorizeTransaction(merchant: string, note: string): { category: string, confidence: 'high' | 'low' }`.

- [ ] **Step 1: Write the failing check**

```js
typeof categorizeTransaction
```

Expected: `"undefined"`.

- [ ] **Step 2: Implement**

Add right after the `CATEGORY_OPTIONS` constant in `index.html`:

```js
    const MERCHANT_CATEGORY_MAP = {
      'lidl': 'Jídlo', 'billa': 'Jídlo', 'kaufland': 'Jídlo', 'tesco': 'Jídlo',
      'penny': 'Jídlo', 'coop': 'Jídlo', 'minimarket': 'Jídlo',
      'mcdonald': 'Jídlo', 'kfc': 'Jídlo', 'dráčik': 'Jídlo', 'dracik': 'Jídlo',
      'mol': 'Doprava', 'orlen': 'Doprava', 'shell': 'Doprava', 'omv': 'Doprava',
      'benzina': 'Doprava', 'easypark': 'Doprava', 'multipark': 'Doprava',
      'parkovací dům': 'Doprava', 'parkovaci dum': 'Doprava',
      'e.on': 'Bydlení', 'eon': 'Bydlení', 'čez': 'Bydlení', 'cez': 'Bydlení',
      'pražská plynárenská': 'Bydlení', 'prazska plynarenska': 'Bydlení',
      'google one': 'Zábava', 'youtube': 'Zábava', 'netflix': 'Zábava',
      'spotify': 'Zábava', 'prime video': 'Zábava',
      'decathlon': 'Oblečení', 'sportisimo': 'Oblečení',
      'trading 212': 'Investice', 'trading212': 'Investice', 'revolut': 'Investice',
      'xtb': 'Investice',
      'lékárna': 'Zdraví', 'lekarna': 'Zdraví', 'nemocnice': 'Zdraví',
    };

    function categorizeTransaction(merchant, note) {
      const merchantLower = (merchant || '').toLowerCase();
      const noteLower = (note || '').toLowerCase();

      if (merchantLower.includes('atm') || noteLower.includes('atm') ||
          merchantLower.includes('výběr hotovosti') || noteLower.includes('výběr hotovosti')) {
        return { category: 'Ostatní', confidence: 'low' };
      }

      for (const [keyword, category] of Object.entries(MERCHANT_CATEGORY_MAP)) {
        if (merchantLower.includes(keyword)) {
          return { category, confidence: 'high' };
        }
      }

      for (const [keyword, category] of Object.entries(MERCHANT_CATEGORY_MAP)) {
        if (noteLower.includes(keyword)) {
          return { category, confidence: 'low' };
        }
      }

      return { category: 'Ostatní', confidence: 'low' };
    }
```

- [ ] **Step 3: Run the checks**

```js
JSON.stringify({
  exact: categorizeTransaction('LIDL PRAHA 5', ''),
  fallback: categorizeTransaction('Unknown Shop s.r.o.', 'platba LIDL'),
  atm: categorizeTransaction('ATM WITHDRAWAL', ''),
  none: categorizeTransaction('Totally Unknown Merchant', 'no keywords here'),
})
```

Expected: `{"exact":{"category":"Jídlo","confidence":"high"},"fallback":{"category":"Jídlo","confidence":"low"},"atm":{"category":"Ostatní","confidence":"low"},"none":{"category":"Ostatní","confidence":"low"}}`

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add merchant-to-category mapping with confidence scoring"
```

---

### Task 4: PapaParse CDN + CSV row parsing

**Files:**
- Modify: `index.html:20` (add script tag after the MSAL script tag)
- Modify: `index.html` — add `parseStatementCsv` function near `parseCsvAmount`/`csvDateToIso` (Task 2)

**Interfaces:**
- Consumes: `parseCsvAmount`, `csvDateToIso` (Task 2), `categorizeTransaction` (Task 3), global `Papa` (from CDN script).
- Produces: `parseStatementCsv(csvText: string): { candidates: Array<{externalId, date, amount, type, merchant, note, category, confidence, counterpartyName}>, unparsedCount: number }`.

- [ ] **Step 1: Add the CDN script tag**

In `index.html`, change:

```html
  <script src="https://alcdn.msauth.net/browser/2.38.2/js/msal-browser.min.js"></script>
```

to:

```html
  <script src="https://alcdn.msauth.net/browser/2.38.2/js/msal-browser.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/papaparse@5.4.1/papaparse.min.js"></script>
```

- [ ] **Step 2: Verify the library loads**

Reload `index.html` in the Browser pane and run:

```js
typeof Papa
```

Expected: `"object"` (or `"function"` depending on PapaParse's export shape — either way, not `"undefined"`).

- [ ] **Step 3: Write the failing check for the parser function**

```js
typeof parseStatementCsv
```

Expected: `"undefined"`.

- [ ] **Step 4: Implement**

Add right after `csvDateToIso` in `index.html`:

```js
    function parseStatementCsv(csvText) {
      const parsed = Papa.parse(csvText, {
        delimiter: ';',
        header: true,
        skipEmptyLines: true,
        transformHeader: h => h.trim(),
      });

      const fields = parsed.meta?.fields || [];
      if (!fields.includes('Datum provedení') || !fields.includes('Zaúčtovaná částka')) {
        throw new Error('Soubor neobsahuje očekávané sloupce (Datum provedení, Zaúčtovaná částka). Zkontrolujte, že jde o výpis ve správném formátu.');
      }

      const candidates = [];
      let unparsedCount = 0;

      for (const row of parsed.data) {
        const iso = csvDateToIso(row['Datum provedení']);
        const amount = parseCsvAmount(row['Zaúčtovaná částka']);

        if (!iso || Number.isNaN(amount)) {
          unparsedCount++;
          continue;
        }

        const merchant = (row['Název obchodníka'] || '').trim();
        const message = (row['Zpráva'] || row['Poznámka'] || '').trim();
        const { category, confidence } = categorizeTransaction(merchant, message);

        candidates.push({
          externalId: (row['Id transakce'] || '').trim(),
          date: iso,
          amount: Math.abs(amount),
          type: amount < 0 ? 'expense' : 'income',
          merchant,
          note: merchant || message,
          category,
          confidence,
          counterpartyName: (row['Název protiúčtu'] || '').trim(),
        });
      }

      return { candidates, unparsedCount };
    }
```

- [ ] **Step 5: Run the check**

```js
const sample = 'Datum provedení;Zaúčtovaná částka;Název obchodníka;Zpráva;Id transakce;Název protiúčtu\n'
  + '29.07.2026;-129,08;LIDL PRAHA;;tx-001;\n'
  + '28.07.2026;25000,00;;Výplata mzdy;tx-002;\n'
  + 'invalid;abc;Nothing;;tx-003;';
JSON.stringify(parseStatementCsv(sample));
```

Expected: `candidates` has exactly 2 entries (row 1: `expense`, `amount: 129.08`, `category: "Jídlo"`, `confidence: "high"`, `externalId: "tx-001"`; row 2: `income`, `amount: 25000`, `note: "Výplata mzdy"`, `externalId: "tx-002"`) and `unparsedCount` is `1`.

Also verify the header-validation guard:

```js
try {
  parseStatementCsv('Some Other Header;Another Column\nfoo;bar');
  'no error thrown';
} catch (err) {
  err.message;
}
```

Expected: the error message about missing expected columns (not `'no error thrown'`).

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Add PapaParse CDN dependency and statement CSV row parser"
```

---

### Task 5: Duplicate detection and internal-transfer detection

**Files:**
- Modify: `index.html` — add `filterDuplicates` and `isInternalTransfer` functions near `parseStatementCsv` (Task 4)
- Modify: `index.html:317-322` — add `STATEMENT_NAME_KEY` constant next to the other storage keys

**Interfaces:**
- Consumes: transaction objects with `externalId` (Task 1), candidate shape from `parseStatementCsv` (Task 4).
- Produces: `filterDuplicates(candidates, existingTransactions): { toImport: Array, duplicateCount: number }`, `isInternalTransfer(candidate, statementName): boolean`, `STATEMENT_NAME_KEY` constant, `getStatementName(): string` (reads `localStorage`).

- [ ] **Step 1: Write the failing check**

```js
JSON.stringify({ a: typeof filterDuplicates, b: typeof isInternalTransfer, c: typeof getStatementName })
```

Expected: `{"a":"undefined","b":"undefined","c":"undefined"}`.

- [ ] **Step 2: Implement**

In `index.html`, change:

```js
    const STORAGE_KEY = 'muj-rozpocet-transactions';
    const CLIENT_ID_KEY = 'muj-rozpocet-azure-client-id';
    const LAST_SYNC_KEY = 'muj-rozpocet-last-sync';
    const LOCAL_UPDATED_KEY = 'muj-rozpocet-local-updated';
```

to:

```js
    const STORAGE_KEY = 'muj-rozpocet-transactions';
    const CLIENT_ID_KEY = 'muj-rozpocet-azure-client-id';
    const LAST_SYNC_KEY = 'muj-rozpocet-last-sync';
    const LOCAL_UPDATED_KEY = 'muj-rozpocet-local-updated';
    const STATEMENT_NAME_KEY = 'muj-rozpocet-statement-name';
```

Then add near `parseStatementCsv`:

```js
    function getStatementName() {
      return localStorage.getItem(STATEMENT_NAME_KEY) || '';
    }

    function filterDuplicates(candidates, existingTransactions) {
      const existingIds = new Set(existingTransactions.map(t => t.externalId).filter(Boolean));
      const toImport = candidates.filter(c => !c.externalId || !existingIds.has(c.externalId));
      return { toImport, duplicateCount: candidates.length - toImport.length };
    }

    function isInternalTransfer(candidate, statementName) {
      if (!statementName) return false;
      return candidate.counterpartyName.toLowerCase() === statementName.trim().toLowerCase();
    }
```

- [ ] **Step 3: Run the checks**

```js
const existing = [{ id: 'x', amount: 1, type: 'expense', category: 'Jídlo', date: '2026-01-01', note: '', externalId: 'tx-001' }];
const candidates = [
  { externalId: 'tx-001', counterpartyName: '' },
  { externalId: 'tx-002', counterpartyName: '' },
  { externalId: '', counterpartyName: '' },
];
JSON.stringify({
  dedup: filterDuplicates(candidates, existing),
  transferYes: isInternalTransfer({ counterpartyName: 'Adam Lang' }, 'Adam Lang'),
  transferNo: isInternalTransfer({ counterpartyName: 'Lidl' }, 'Adam Lang'),
  transferNoSetting: isInternalTransfer({ counterpartyName: 'Adam Lang' }, ''),
})
```

Expected: `dedup.toImport` has 2 entries (`tx-002` and the empty-externalId one), `dedup.duplicateCount` is `1`; `transferYes` is `true`, `transferNo` is `false`, `transferNoSetting` is `false`.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add duplicate and internal-transfer detection for CSV import"
```

---

### Task 6: "Statement name" setting field (UI)

**Files:**
- Modify: `index.html` — add a small settings field in the "Nová transakce" section, right after `</form>` (`index.html:119`)
- Modify: `index.html` — wire the field to `localStorage` near where `client-id` input is wired

**Interfaces:**
- Produces: `<input id="statement-name">` element, pre-filled from `getStatementName()` on load, saved to `localStorage` on blur.

- [ ] **Step 1: Add the markup**

In `index.html`, change:

```html
            <button type="submit"
              class="w-full bg-primary-600 hover:bg-primary-700 text-white font-medium py-2.5 px-4 rounded-lg transition-colors text-sm">
              Přidat transakci
            </button>
          </form>
        </section>
```

to:

```html
            <button type="submit"
              class="w-full bg-primary-600 hover:bg-primary-700 text-white font-medium py-2.5 px-4 rounded-lg transition-colors text-sm">
              Přidat transakci
            </button>
          </form>

          <div class="mt-6 pt-6 border-t border-slate-700 space-y-3">
            <div>
              <label for="statement-name" class="block text-sm font-medium text-slate-300 mb-1">
                Vaše jméno na výpisu
              </label>
              <input type="text" id="statement-name" placeholder="např. Adam Lang"
                class="w-full rounded-lg border border-slate-600 bg-slate-700 text-slate-100 px-3 py-2.5 text-sm focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-transparent placeholder-slate-500">
              <p class="text-xs text-slate-500 mt-1">Používá se při CSV importu k rozpoznání převodů mezi vlastními účty.</p>
            </div>
            <button type="button" id="csv-upload-btn"
              class="w-full bg-slate-700 hover:bg-slate-600 text-slate-300 font-medium py-2.5 px-4 rounded-lg transition-colors text-sm border border-slate-600">
              Nahrát CSV výpis
            </button>
            <input type="file" id="csv-input" accept=".csv,text/csv" class="hidden">
          </div>
        </section>
```

- [ ] **Step 2: Wire persistence**

Add near the other DOM-reference constants (`index.html:353-354`, right after `const importInput = ...`):

```js
    const statementNameInput = document.getElementById('statement-name');
    const csvUploadBtn = document.getElementById('csv-upload-btn');
    const csvInput = document.getElementById('csv-input');
```

Add near `init()` (inside it, right after `setDefaultDate();`):

```js
      statementNameInput.value = getStatementName();
```

Add near the bottom, next to the other `addEventListener` wiring (`index.html:1093-1097`):

```js
    statementNameInput.addEventListener('blur', () => {
      localStorage.setItem(STATEMENT_NAME_KEY, statementNameInput.value.trim());
    });
```

- [ ] **Step 3: Manual check in the browser**

Open `index.html` in the Browser pane, type `Adam Lang` into the new "Vaše jméno na výpisu" field, click elsewhere to blur, then run:

```js
localStorage.getItem('muj-rozpocet-statement-name')
```

Expected: `"Adam Lang"`. Reload the page and confirm the field still shows `Adam Lang`.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add statement name setting for internal transfer detection"
```

---

### Task 7: Import review modal — markup and open/close

**Files:**
- Modify: `index.html` — add modal markup right before `</body>` (after `</footer>`, `index.html:314`)
- Modify: `index.html` — add `openImportModal`/`closeImportModal` functions and wire `csvUploadBtn`/`csvInput`

**Interfaces:**
- Consumes: `csvUploadBtn`, `csvInput` (Task 6), `parseStatementCsv` (Task 4), `filterDuplicates`, `isInternalTransfer`, `getStatementName` (Task 5).
- Produces: module-level `let pendingImportRows = []`; `openImportModal(rows, meta)`, `closeImportModal()`; DOM ids `import-modal`, `import-modal-body`, `import-modal-summary`, `import-cancel-btn`, `import-confirm-btn`.

- [ ] **Step 1: Add modal markup**

In `index.html`, change:

```html
  <footer class="max-w-6xl mx-auto px-4 py-6 text-center text-xs text-slate-500">
    Lokální cache v prohlížeči + automatická synchronizace do OneDrive po přihlášení.
  </footer>
```

to:

```html
  <footer class="max-w-6xl mx-auto px-4 py-6 text-center text-xs text-slate-500">
    Lokální cache v prohlížeči + automatická synchronizace do OneDrive po přihlášení.
  </footer>

  <div id="import-modal" class="hidden fixed inset-0 bg-black/60 z-50 flex items-center justify-center p-4">
    <div class="bg-slate-800 border border-slate-700 rounded-2xl shadow-lg max-w-4xl w-full max-h-[85vh] flex flex-col">
      <div class="px-6 py-4 border-b border-slate-700">
        <h2 class="text-lg font-semibold text-slate-100">Náhled importu</h2>
        <p id="import-modal-summary" class="text-sm text-slate-400 mt-1"></p>
      </div>
      <div class="overflow-y-auto flex-1 px-6 py-4">
        <table class="w-full text-sm">
          <thead>
            <tr class="text-left text-slate-400 uppercase text-xs tracking-wide">
              <th class="py-2 pr-2"></th>
              <th class="py-2 pr-2">Datum</th>
              <th class="py-2 pr-2">Obchodník / popis</th>
              <th class="py-2 pr-2 text-right">Částka</th>
              <th class="py-2 pr-2">Kategorie</th>
              <th class="py-2 pr-2">Stav</th>
            </tr>
          </thead>
          <tbody id="import-modal-body"></tbody>
        </table>
      </div>
      <div class="px-6 py-4 border-t border-slate-700 flex justify-end gap-3">
        <button type="button" id="import-cancel-btn"
          class="bg-slate-700 hover:bg-slate-600 text-slate-300 font-medium py-2.5 px-4 rounded-lg transition-colors text-sm border border-slate-600">
          Zrušit
        </button>
        <button type="button" id="import-confirm-btn"
          class="bg-primary-600 hover:bg-primary-700 text-white font-medium py-2.5 px-4 rounded-lg transition-colors text-sm">
          Potvrdit a uložit vše
        </button>
      </div>
    </div>
  </div>
```

- [ ] **Step 2: Write the failing check**

```js
JSON.stringify({ a: typeof openImportModal, b: typeof closeImportModal })
```

Expected: `{"a":"undefined","b":"undefined"}`.

- [ ] **Step 3: Implement open/close + file handling**

Add near the other DOM-reference constants (after the ones added in Task 6):

```js
    const importModal = document.getElementById('import-modal');
    const importModalBody = document.getElementById('import-modal-body');
    const importModalSummary = document.getElementById('import-modal-summary');
    const importCancelBtn = document.getElementById('import-cancel-btn');
    const importConfirmBtn = document.getElementById('import-confirm-btn');

    let pendingImportRows = [];
```

Add a new function near `parseStatementCsv`:

```js
    function closeImportModal() {
      importModal.classList.add('hidden');
      pendingImportRows = [];
    }

    function openImportModal(csvText) {
      let candidates, unparsedCount;
      try {
        ({ candidates, unparsedCount } = parseStatementCsv(csvText));
      } catch (err) {
        alert('Chyba při čtení CSV: ' + err.message);
        return;
      }

      const { toImport, duplicateCount } = filterDuplicates(candidates, transactions);
      const statementName = getStatementName();

      pendingImportRows = toImport.map(c => ({
        ...c,
        checked: !isInternalTransfer(c, statementName),
        isInternalTransfer: isInternalTransfer(c, statementName),
      }));

      const parts = [`${pendingImportRows.length} nových transakcí k importu`];
      if (duplicateCount > 0) parts.push(`${duplicateCount} už bylo dříve naimportováno (přeskočeno)`);
      if (unparsedCount > 0) parts.push(`${unparsedCount} řádků se nepodařilo přečíst`);
      importModalSummary.textContent = parts.join(' · ');

      if (pendingImportRows.length === 0) {
        importModalBody.innerHTML = '<tr><td colspan="6" class="py-8 text-center text-slate-500">Žádné nové transakce k importu.</td></tr>';
      }

      importModal.classList.remove('hidden');
    }
```

Wire the button/input near the other event listeners (after the `importInput.addEventListener` block, `index.html:1094-1097`):

```js
    csvUploadBtn.addEventListener('click', () => csvInput.click());
    csvInput.addEventListener('change', (e) => {
      const file = e.target.files[0];
      if (!file) return;
      const reader = new FileReader();
      reader.onload = (ev) => openImportModal(ev.target.result);
      reader.readAsText(file);
      e.target.value = '';
    });
    importCancelBtn.addEventListener('click', closeImportModal);
```

- [ ] **Step 4: Manual check in the browser**

Open `index.html` in the Browser pane. Run this to simulate picking a file (there's no native file picker to drive via the automation tools, so we set `input.files` directly with `DataTransfer`, which is a standard technique and behaves exactly like a real file pick for the `change` handler):

```js
const csv = 'Datum provedení;Zaúčtovaná částka;Název obchodníka;Zpráva;Id transakce;Název protiúčtu\n'
  + '29.07.2026;-129,08;LIDL PRAHA;;tx-100;\n';
const file = new File([csv], 'test.csv', { type: 'text/csv' });
const dt = new DataTransfer();
dt.items.add(file);
document.getElementById('csv-input').files = dt.files;
document.getElementById('csv-input').dispatchEvent(new Event('change', { bubbles: true }));
document.getElementById('import-modal').classList.contains('hidden');
```

Expected: `false` (modal is now visible). Then run:

```js
document.getElementById('import-modal-summary').textContent
```

Expected: contains `"1 nových transakcí k importu"`.

Also verify the invalid-format path stays closed and alerts instead of opening:

```js
document.getElementById('import-modal').classList.add('hidden'); // reset from previous check
let alertMessage = null;
const originalAlert = window.alert;
window.alert = (msg) => { alertMessage = msg; };
const badFile = new File(['Foo;Bar\n1;2'], 'bad.csv', { type: 'text/csv' });
const dt3 = new DataTransfer();
dt3.items.add(badFile);
document.getElementById('csv-input').files = dt3.files;
document.getElementById('csv-input').dispatchEvent(new Event('change', { bubbles: true }));
window.alert = originalAlert;
JSON.stringify({ alertShown: alertMessage, modalHidden: document.getElementById('import-modal').classList.contains('hidden') });
```

Expected: `alertShown` contains `"Chyba při čtení CSV"`, `modalHidden` is `true`.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Add CSV import review modal with open/close and file handling"
```

---

### Task 8: Render import rows in the modal (checkbox, category dropdown, confidence/transfer badges)

**Files:**
- Modify: `index.html` — add `renderImportModalRows` function next to `openImportModal` (Task 7), call it from `openImportModal`

**Interfaces:**
- Consumes: `pendingImportRows` (Task 7), `CATEGORY_OPTIONS` (existing), `formatCurrency`, `formatDate`, `escapeHtml` (existing).
- Produces: `renderImportModalRows()` — renders `pendingImportRows` into `#import-modal-body`; each row's checkbox has `data-index`, each category `<select>` has `data-index`.

- [ ] **Step 1: Write the failing check**

```js
typeof renderImportModalRows
```

Expected: `"undefined"`.

- [ ] **Step 2: Implement**

Add right after `openImportModal` in `index.html`:

```js
    function renderImportModalRows() {
      if (pendingImportRows.length === 0) return;

      importModalBody.innerHTML = pendingImportRows.map((row, index) => {
        const options = CATEGORY_OPTIONS[row.type] || CATEGORY_OPTIONS.expense;
        const optionsHtml = options.map(opt =>
          `<option value="${opt}" ${opt === row.category ? 'selected' : ''}>${opt}</option>`
        ).join('');

        const amountClass = row.type === 'income' ? 'text-emerald-400' : 'text-rose-400';
        const amountPrefix = row.type === 'income' ? '+' : '−';

        let statusBadge = '';
        if (row.isInternalTransfer) {
          statusBadge = '<span class="inline-flex items-center px-2 py-0.5 rounded-full text-xs font-medium bg-sky-900 text-sky-300">Interní převod</span>';
        } else if (row.confidence === 'low') {
          statusBadge = '<span class="inline-flex items-center px-2 py-0.5 rounded-full text-xs font-medium bg-amber-900 text-amber-200">Zkontrolovat</span>';
        }

        const rowClass = row.confidence === 'low' && !row.isInternalTransfer
          ? 'border-t border-amber-700'
          : 'border-t border-slate-700';

        return `
          <tr class="${rowClass}">
            <td class="py-2 pr-2">
              <input type="checkbox" class="import-row-checkbox" data-index="${index}" ${row.checked ? 'checked' : ''}>
            </td>
            <td class="py-2 pr-2 whitespace-nowrap text-slate-300">${formatDate(row.date)}</td>
            <td class="py-2 pr-2 text-slate-300 max-w-[200px] truncate" title="${escapeHtml(row.note)}">${escapeHtml(row.note) || '—'}</td>
            <td class="py-2 pr-2 text-right font-medium ${amountClass} whitespace-nowrap">${amountPrefix}${formatCurrency(row.amount)}</td>
            <td class="py-2 pr-2">
              <select class="import-row-category bg-slate-700 border border-slate-600 text-slate-100 text-xs rounded px-2 py-1" data-index="${index}">
                ${optionsHtml}
              </select>
            </td>
            <td class="py-2 pr-2">${statusBadge}</td>
          </tr>`;
      }).join('');

      document.querySelectorAll('.import-row-checkbox').forEach(cb => {
        cb.addEventListener('change', () => {
          pendingImportRows[Number(cb.dataset.index)].checked = cb.checked;
        });
      });
      document.querySelectorAll('.import-row-category').forEach(sel => {
        sel.addEventListener('change', () => {
          pendingImportRows[Number(sel.dataset.index)].category = sel.value;
        });
      });
    }
```

Then call it at the end of `openImportModal`, right before `importModal.classList.remove('hidden');`:

```js
      renderImportModalRows();
      importModal.classList.remove('hidden');
```

(Remove the old inline "no rows" `innerHTML` assignment from Task 7's `openImportModal` — `renderImportModalRows` now owns rendering, including the empty case. Update it to:)

```js
      if (pendingImportRows.length === 0) {
        importModalBody.innerHTML = '<tr><td colspan="6" class="py-8 text-center text-slate-500">Žádné nové transakce k importu.</td></tr>';
      } else {
        renderImportModalRows();
      }

      importModal.classList.remove('hidden');
```

- [ ] **Step 3: Manual check in the browser**

Repeat the Task 7 Step 4 file-injection snippet, then run:

```js
JSON.stringify({
  rows: document.querySelectorAll('#import-modal-body tr').length,
  checkbox: document.querySelector('.import-row-checkbox').checked,
  category: document.querySelector('.import-row-category').value,
})
```

Expected: `{"rows":1,"checkbox":true,"category":"Jídlo"}` (LIDL matches high-confidence, so no amber badge; checkbox defaults checked since it's not an internal transfer).

Then test an internal transfer: set the statement name first, then re-inject a CSV row whose counterparty matches it:

```js
localStorage.setItem('muj-rozpocet-statement-name', 'Adam Lang');
const csv2 = 'Datum provedení;Zaúčtovaná částka;Název obchodníka;Zpráva;Id transakce;Název protiúčtu\n'
  + '29.07.2026;-500,00;;Vlastní převod;tx-200;Adam Lang\n';
const file2 = new File([csv2], 'test2.csv', { type: 'text/csv' });
const dt2 = new DataTransfer();
dt2.items.add(file2);
document.getElementById('csv-input').files = dt2.files;
document.getElementById('csv-input').dispatchEvent(new Event('change', { bubbles: true }));
JSON.stringify({
  checked: document.querySelector('.import-row-checkbox').checked,
  badge: document.querySelector('#import-modal-body td:last-child').textContent,
})
```

Expected: `{"checked":false,"badge":"Interní převod"}`.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Render CSV import rows with editable category and status badges"
```

---

### Task 9: Confirm & save imported rows

**Files:**
- Modify: `index.html` — add `confirmImport` function next to `renderImportModalRows` (Task 8), wire `importConfirmBtn`

**Interfaces:**
- Consumes: `pendingImportRows` (Task 7), `generateId`, `saveTransactionsLocal`, `scheduleOneDriveSync`, `render` (all existing).
- Produces: `confirmImport()` — appends checked rows to `transactions`, persists once, re-renders, closes modal.

- [ ] **Step 1: Write the failing check**

```js
typeof confirmImport
```

Expected: `"undefined"`.

- [ ] **Step 2: Implement**

Add right after `renderImportModalRows`:

```js
    function confirmImport() {
      const toSave = pendingImportRows.filter(r => r.checked);

      for (const row of toSave) {
        transactions.unshift({
          id: generateId(),
          amount: row.amount,
          type: row.type,
          category: row.category,
          date: row.date,
          note: row.note,
          externalId: row.externalId || undefined,
        });
      }

      if (toSave.length > 0) {
        saveTransactionsLocal();
        scheduleOneDriveSync();
        render();
      }

      alert(`Naimportováno ${toSave.length} transakcí.`);
      closeImportModal();
    }
```

Wire the button near the other listeners added in Task 7:

```js
    importConfirmBtn.addEventListener('click', confirmImport);
```

- [ ] **Step 3: Manual check in the browser**

Using the Task 8 Step 3 setup (one pending row for LIDL, `tx-001` or `tx-100` depending on which sample is still loaded — re-run the first CSV injection snippet from Task 7 Step 4 to get a clean single pending row), run:

```js
const before = transactions.length;
document.getElementById('import-confirm-btn').click();
JSON.stringify({
  added: transactions.length - before,
  lastTx: transactions[0],
  modalHidden: document.getElementById('import-modal').classList.contains('hidden'),
})
```

Expected: `added` is `1`, `lastTx.externalId` is `"tx-100"` (or whichever id was used), `lastTx.category` is `"Jídlo"`, `modalHidden` is `true`.

Then confirm re-importing the same file is now a no-op (duplicate detection): re-inject the exact same CSV file from Task 7 Step 4 and check the summary:

```js
document.getElementById('import-modal-summary').textContent
```

Expected: contains `"0 nových transakcí k importu"` and `"1 už bylo dříve naimportováno"`.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Save confirmed CSV import rows as transactions in one batch"
```

---

### Task 10: Inline category edit in the transaction history table

**Files:**
- Modify: `index.html:972-989` (`renderTable`'s row template — the category `<td>`)

**Interfaces:**
- Consumes: `CATEGORY_OPTIONS`, `CATEGORY_COLORS`, `saveTransactions`, `render` (all existing).
- Produces: clicking a category badge in "Historie transakcí" turns it into a `<select>`; changing the select saves immediately and re-renders. Works for both manual and imported transactions (no code path distinguishes them).

- [ ] **Step 1: Write the failing check**

Add at least one transaction (via the normal form), then run:

```js
document.querySelectorAll('.category-cell').length
```

Expected: `0` (the class doesn't exist yet — category cells aren't individually targetable).

- [ ] **Step 2: Implement**

In `index.html`, change the category `<td>` inside `renderTable`'s template from:

```js
          <td class="px-6 py-3.5">
            <span class="inline-flex items-center gap-1.5 text-slate-300">
              <span class="w-2 h-2 rounded-full" style="background:${CATEGORY_COLORS[t.category] || '#94a3b8'}"></span>
              ${escapeHtml(t.category)}
            </span>
          </td>
```

to:

```js
          <td class="px-6 py-3.5 category-cell" data-id="${t.id}" data-type="${t.type}">
            <span class="category-display inline-flex items-center gap-1.5 text-slate-300 cursor-pointer hover:underline" title="Kliknutím změníte kategorii">
              <span class="w-2 h-2 rounded-full" style="background:${CATEGORY_COLORS[t.category] || '#94a3b8'}"></span>
              ${escapeHtml(t.category)}
            </span>
          </td>
```

Then, right after the existing `.delete-btn` wiring at the end of `renderTable` (`index.html:992-994`), add:

```js
      document.querySelectorAll('.category-cell').forEach(cell => {
        cell.addEventListener('click', () => {
          const id = cell.dataset.id;
          const type = cell.dataset.type;
          const current = transactions.find(t => t.id === id)?.category;
          const options = CATEGORY_OPTIONS[type] || CATEGORY_OPTIONS.expense;
          const select = document.createElement('select');
          select.className = 'bg-slate-700 border border-slate-600 text-slate-100 text-xs rounded px-2 py-1';
          select.innerHTML = options.map(opt =>
            `<option value="${opt}" ${opt === current ? 'selected' : ''}>${opt}</option>`
          ).join('');
          select.addEventListener('change', () => {
            const tx = transactions.find(t => t.id === id);
            if (tx) {
              tx.category = select.value;
              saveTransactions();
              render();
            }
          });
          select.addEventListener('blur', () => render());
          cell.innerHTML = '';
          cell.appendChild(select);
          select.focus();
        });
      });
```

- [ ] **Step 3: Manual check in the browser**

With at least one transaction present (add one via the form if needed), run:

```js
const cell = document.querySelector('.category-cell');
cell.click();
JSON.stringify({ hasSelect: !!cell.querySelector('select') })
```

Expected: `{"hasSelect":true}`.

Then change the category and confirm it saved:

```js
const select = document.querySelector('.category-cell select');
const id = document.querySelector('.category-cell').dataset.id;
const otherOption = Array.from(select.options).find(o => o.value !== select.value);
select.value = otherOption.value;
select.dispatchEvent(new Event('change', { bubbles: true }));
transactions.find(t => t.id === id).category === otherOption.value
```

Expected: `true`. Also confirm it round-trips through storage:

```js
JSON.parse(localStorage.getItem('muj-rozpocet-transactions')).find(t => t.id === id).category === otherOption.value
```

Expected: `true`.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add inline category editing in transaction history table"
```

---

### Task 11: End-to-end verification pass

**Files:**
- No code changes expected — this task only verifies Tasks 1-10 work together. If it finds a bug, fix it in the relevant file/location from the task above that owns that code, then re-run this task's checks.

- [ ] **Step 1: Reset local state and reload**

In the Browser pane, run:

```js
localStorage.clear();
location.reload();
```

- [ ] **Step 2: Set the statement name**

Type `Adam Lang` into "Vaše jméno na výpisu" and blur the field (or via `javascript_tool`, set the input's value, dispatch `blur`).

- [ ] **Step 3: Build a realistic multi-row sample CSV and import it**

```js
const csv = [
  'Datum provedení;Zaúčtovaná částka;Název obchodníka;Zpráva;Typ transakce;Id transakce;Číslo protiúčtu;Název protiúčtu',
  '29.07.2026;-129,08;LIDL PRAHA 5;;Platba kartou;tx-001;;',
  '28.07.2026;-45,00;ORLEN;;Platba kartou;tx-002;;',
  '27.07.2026;25000,00;;Výplata mzdy;Odchozí okamžitá úhrada;tx-003;;',
  '26.07.2026;-1000,00;;Vlastní převod;Odchozí okamžitá úhrada;tx-004;123456/0100;Adam Lang',
  '25.07.2026;-500,00;Finanční správa;;Trvalý příkaz;tx-005;;',
].join('\n');
const file = new File([csv], 'statement.csv', { type: 'text/csv' });
const dt = new DataTransfer();
dt.items.add(file);
document.getElementById('csv-input').files = dt.files;
document.getElementById('csv-input').dispatchEvent(new Event('change', { bubbles: true }));
JSON.stringify({
  summary: document.getElementById('import-modal-summary').textContent,
  rowCount: document.querySelectorAll('#import-modal-body tr').length,
})
```

Expected: `rowCount` is `5`, summary mentions `5 nových transakcí`.

- [ ] **Step 4: Verify categorization and flags per row**

```js
JSON.stringify(pendingImportRows.map(r => ({ note: r.note, category: r.category, confidence: r.confidence, checked: r.checked, transfer: r.isInternalTransfer })))
```

Expected:
- `LIDL PRAHA 5` → `Jídlo`, `high`, `checked: true`
- `ORLEN` → `Doprava`, `high`, `checked: true`
- `Výplata mzdy` → `Ostatní`, `low`, `checked: true` (no merchant, no keyword match — falls to catch-all)
- `Vlastní převod` (counterparty "Adam Lang") → `checked: false`, `transfer: true`
- `Finanční správa` → `Ostatní`, `low`, `checked: true` (no dictionary entry — falls to catch-all, flagged for review)

- [ ] **Step 5: Confirm import and verify saved state**

```js
document.getElementById('import-confirm-btn').click();
JSON.stringify({
  count: transactions.length,
  ids: transactions.map(t => t.externalId).filter(Boolean).sort(),
})
```

Expected: `count` is `4` (5 rows minus the unchecked internal transfer), `ids` is `["tx-001","tx-002","tx-003","tx-005"]`.

- [ ] **Step 6: Re-import the same file and confirm full dedup**

Repeat Step 3's file injection with the identical CSV content, then check the summary:

```js
document.getElementById('import-modal-summary').textContent
```

Expected: contains `"0 nových transakcí"` and `"4 už bylo dříve naimportováno"` (the 4 previously-saved rows are recognized; the 1 unchecked transfer was never saved so it isn't a "duplicate" — it will show up again as a fresh candidate available to import if desired).

- [ ] **Step 7: Verify inline category edit works on an imported row**

```js
const importedId = transactions.find(t => t.externalId === 'tx-005').id;
document.querySelector(`.category-cell[data-id="${importedId}"]`).click();
const select = document.querySelector(`.category-cell[data-id="${importedId}"] select`);
select.value = 'Vzdělání';
select.dispatchEvent(new Event('change', { bubbles: true }));
transactions.find(t => t.id === importedId).category
```

Expected: `"Vzdělání"`.

- [ ] **Step 8: Confirm no console errors**

Use `mcp__Claude_Browser__read_console_messages` with `onlyErrors: true` after the steps above. Expected: no errors.

- [ ] **Step 9: Final commit (if any fixes were needed)**

```bash
git add index.html
git commit -m "Fix issues found in CSV import end-to-end verification"
```

(Skip this commit if Steps 1-8 required no code changes.)
