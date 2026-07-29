# Import CSV výpisu z účtu s automatickou kategorizací plateb

Datum: 2026-07-29
Stav: schváleno uživatelem, čeká na implementační plán

## Cíl

Umožnit nahrání CSV výpisu z bankovního účtu, automaticky navrhnout kategorii pro
každou platbu podle názvu obchodníka, ukázat uživateli náhled ke kontrole/opravě
a teprve po potvrzení uložit platby jako běžné transakce (stejně jako ruční zadání).

## Kontext / current state

Aplikace je jeden soubor `index.html` (Tailwind CDN, Chart.js CDN, MSAL CDN,
OneDrive sync). Datový model transakce: `{ id, amount, type, category, date, note }`.
Existující kategorie (pevně dané v kódu, používané i pro 33/33/33 strategii):

- **Příjem:** Plat, Ostatní
- **Výdaj:** Jídlo, Bydlení, Zábava, Doprava, Zdraví, Oblečení, Vzdělání, Investice, Ostatní

`getStrategyBudgets()` počítá 33/33/33 rozdělení podle pevného seznamu kategorií
(`essentialCategories`, `futureCategories`, `personalCategories`) — jakákoli
kategorie mimo tento seznam se nikam nezapočítá. **Proto tento návrh nepřidává
žádné nové kategorie**, vše se mapuje na existující sadu.

Aplikace aktuálně **nemá editaci** uložených transakcí, jen přidání a smazání
(`deleteTransaction`, žádná `editTransaction`).

## Rozhodnutí z brainstormingu

1. Mapovat výhradně na existující kategorie; cokoli bez jasné shody (ATM výběr,
   daně/Finanční správa) spadá do „Ostatní", ale je vždy označeno k ruční kontrole.
2. Detekce interních převodů běží přes nové nastavení „Vaše jméno na výpisu"
   (localStorage), porovnávané s `Název protiúčtu` z CSV.
3. Import v této práci přidá i lehkou editaci kategorie přímo v tabulce
   Historie transakcí (dosud neexistovala vůbec) — jinak by šlo opravit
   kategorii u uloženého importu jen smazáním a ručním přidáním znovu.
4. Náhled před uložením je modal přes celou obrazovku.

## Datový model

Transakce dostane nové volitelné pole:

```js
{ id, amount, type, category, date, note, externalId }
```

- `externalId` = hodnota `Id transakce` z CSV. U ručně zadaných transakcí chybí
  (`undefined`) a nijak jinak se s ním nepracuje.
- `normalizeTransactions()` musí pole zachovávat (dnes by ho tiše zahodilo,
  protože mapuje jen na explicitní whitelist polí).
- `buildPayload()`/OneDrive sync nepotřebuje žádnou změnu — `externalId` prostě
  poplyne skrz `transactions` jako další vlastnost JSON objektu.

Nové nastavení (localStorage, vedle `muj-rozpocet-azure-client-id`):

- klíč `muj-rozpocet-statement-name`, textová hodnota jména uživatele tak, jak
  se objevuje ve výpisu (např. „Adam Lang").

## CSV parsování

- Přidat PapaParse přes CDN (`<script src="https://cdn.jsdelivr.net/npm/papaparse@5/papaparse.min.js"></script>"`),
  konzistentní s existujícím vzorem načítání knihoven (Chart.js, MSAL).
- Konfigurace parseru: `delimiter: ';'`, `header: true`, `skipEmptyLines: true`,
  `transformHeader` ořeže bílé znaky. PapaParse si poradí s BOM, uvozovkami
  i středníky uvnitř quoted hodnot automaticky.
- Číselný parser částky: odebrat mezery (tisícové oddělovače), nahradit `,` za
  `.`, `parseFloat`. Záporná částka = `expense` (`amount = Math.abs(...)`),
  kladná = `income`.
- Datum `DD.MM.YYYY` → převod na `YYYY-MM-DD` (formát používaný zbytkem appky).
- Použité sloupce CSV: `Datum provedení`, `Zaúčtovaná částka`, `Název obchodníka`,
  `Zpráva`/`Poznámka` (fallback popis, pokud obchodník chybí), `Typ transakce`,
  `Id transakce`, `Číslo protiúčtu`, `Název protiúčtu`.
- Řádek bez parsovatelného data nebo částky se přeskočí a započítá do souhrnu
  „X řádků se nepodařilo přečíst" zobrazeného nahoře v modalu.

## Kategorizace a confidence

Nová konstanta `MERCHANT_CATEGORY_MAP` (samostatný objekt v kódu, blízko
`CATEGORY_OPTIONS`, snadno rozšiřitelný o další obchodníky):

```js
const MERCHANT_CATEGORY_MAP = {
  // Jídlo
  'lidl': 'Jídlo', 'billa': 'Jídlo', 'kaufland': 'Jídlo', 'tesco': 'Jídlo',
  'penny': 'Jídlo', 'coop': 'Jídlo', 'minimarket': 'Jídlo',
  'mcdonald': 'Jídlo', 'kfc': 'Jídlo', 'dráčik': 'Jídlo', 'dracik': 'Jídlo',
  // Doprava
  'mol': 'Doprava', 'orlen': 'Doprava', 'shell': 'Doprava', 'omv': 'Doprava',
  'benzina': 'Doprava', 'easypark': 'Doprava', 'multipark': 'Doprava',
  'parkovací dům': 'Doprava', 'parkovaci dum': 'Doprava',
  // Bydlení
  'e.on': 'Bydlení', 'eon': 'Bydlení', 'čez': 'Bydlení', 'cez': 'Bydlení',
  'pražská plynárenská': 'Bydlení', 'pre ': 'Bydlení',
  // Zábava / předplatné
  'google one': 'Zábava', 'youtube': 'Zábava', 'netflix': 'Zábava',
  'spotify': 'Zábava', 'prime video': 'Zábava',
  // Oblečení / sport vybavení
  'decathlon': 'Oblečení', 'sportisimo': 'Oblečení',
  // Investice
  'trading 212': 'Investice', 'trading212': 'Investice', 'revolut': 'Investice',
  'xtb': 'Investice',
  // Zdraví
  'lékárna': 'Zdraví', 'lekarna': 'Zdraví', 'nemocnice': 'Zdraví',
};
```

- Klíče jsou lowercase podřetězce hledané v lowercased `Název obchodníka`.
- **Vysoká jistota:** podřetězec z `MERCHANT_CATEGORY_MAP` nalezen v `Název obchodníka`.
- **Nízká jistota:** shoda nalezena jen ve `Zpráva`/`Poznámka` (stejný slovník,
  fallback zdroj), NEBO žádná shoda vůbec (kategorie se přednastaví na „Ostatní").
- ATM/výběr hotovosti: rozpoznat klíčovým slovem `atm` nebo `výběr hotovosti`
  v obchodníkovi/zprávě → kategorie „Ostatní", ale **vždy** nízká jistota
  (nucená kontrola), i když je shoda „jistá" v tom smyslu, že víme, že jde o výběr.
- Typ transakce se nepoužívá pro kategorizaci samotnou, jen pro detekci
  interních převodů (viz níže).

## Detekce interních převodů

- Pokud `Název protiúčtu` (case-insensitive, ořezané mezery) odpovídá hodnotě
  uloženého nastavení `muj-rozpocet-statement-name`, řádek se v náhledu označí
  štítkem „Interní převod" a jeho checkbox je **defaultně odškrtnutý**.
- Pokud nastavení není vyplněné, žádná automatická detekce neběží — všechny
  řádky mají checkbox zaškrtnutý jako obvykle, uživatel je může sám odškrtnout.
- Nejde o novou kategorii — jde jen o výchozí stav checkboxu + vizuální štítek.

## Detekce duplicit

- Před zobrazením modalu porovnat `Id transakce` (→ bude `externalId`) každého
  parsovaného řádku s `externalId` všech existujících `transactions`.
- Řádky se shodou se **nezobrazují** v editovatelném seznamu k importu; nahoře
  modalu se zobrazí souhrnná hláška „N záznamů už bylo dříve naimportováno,
  přeskočeno".
- Pokud po odfiltrování duplicit a neparsovatelných řádků nezbyde nic k importu,
  modal to řekne rovnou a nabídne jen zavření (žádná prázdná tabulka).

## UI a flow

1. V kartě „Nová transakce" (`index.html` cca řádek 76+), pod tlačítkem
   „Přidat transakci" a oddělené tenkou linkou (`border-t border-slate-700`),
   přibude sekundární tlačítko „Nahrát CSV výpis" + skryté
   `<input type="file" accept=".csv">`.
2. Klik na tlačítko → otevře file picker → po výběru souboru se přečte jako
   text (`FileReader`/`file.text()`), naparsuje PapaParse, zpracuje kategorizace
   a duplicity, výsledek naplní modal.
3. **Modal** (nový `<dialog>` nebo overlay `div` se stejným vizuálním stylem
   jako zbytek appky — `bg-slate-800`, `border-slate-700`, tmavý theme):
   - Hlavička: „Náhled importu" + souhrn (kolik řádků, kolik přeskočeno jako
     duplicity, kolik nerozpoznáno).
   - Tabulka řádků: checkbox | datum | obchodník/popis | částka (barva podle
     typu) | dropdown kategorie (možnosti filtrované podle `income`/`expense`
     stejně jako `CATEGORY_OPTIONS`) | štítek jistoty/interního převodu.
   - Řádek s nízkou jistotou: žlutý/amber rámeček (`border-amber-500` nebo
     obdoba existující `bg-amber-900`/`text-amber-200` palety použité u
     HTTPS varování) + štítek „Zkontrolovat".
   - Patička modalu: „Zrušit" (zavře modal, nic neuloží) a „Potvrdit a uložit
     vše" (uloží jen zaškrtnuté řádky).
4. Uložení: pro každý zaškrtnutý řádek vytvořit transakci stejnou cestou jako
   `addTransaction` (generovat `id` přes `generateId()`, uložit `externalId`),
   naráz vložit do `transactions`, zavolat `saveTransactionsLocal()` +
   `render()` a **jednou** `scheduleOneDriveSync()` po celé dávce (ne per řádek).

## Editace kategorie v historii transakcí

- V `renderTable()` (řádek ~955) přidat možnost kliknout na buňku kategorie u
  libovolné transakce → přepne se na `<select>` s možnostmi podle typu dané
  transakce (stejný zdroj jako `CATEGORY_OPTIONS`), výběr rovnou uloží
  (`t.category = ...; saveTransactions(); render();`).
- Netýká se to jen importovaných záznamů — funguje to pro všechny transakce,
  ruční i importované.

## Error handling

- Nečitelný/prázdný soubor, špatný formát (chybí klíčové sloupce) → `alert()`
  se srozumitelnou hláškou, modal se neotevře (konzistentní s existujícím
  stylem chybových hlášek v appce, např. `syncWithOneDrive`).
- Řádky s nevalidní částkou/datem se tiše přeskočí a připočtou do souhrnu
  „nerozpoznaných" řádků v hlavičce modalu (viz sekce Parsování).

## Testing

- Ruční test v prohlížeči s reálným/anonymizovaným vzorkem CSV: ověřit BOM,
  desetinnou čárku, kategorizaci podle slovníku i fallback, low-confidence
  zvýraznění, duplicitní re-import (nahrát stejný soubor podruhé → vše
  přeskočeno), interní převod (pokud je nastavené jméno) a uložení do
  `localStorage`/`transactions`.
- Ověřit, že `externalId` přežije `normalizeTransactions()` a je vidět v
  `localStorage` po uložení.
- Ověřit editaci kategorie v historii na ručně zadané i importované transakci.

## Mimo rozsah

- Žádné nové kategorie ani úprava 33/33/33 logiky.
- Žádný plný edit řádku (datum/částka/poznámka) v historii — jen kategorie.
- Žádné strojové učení / fuzzy matching nad rámec substring + keyword hledání.
