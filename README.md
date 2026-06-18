# Symfony Conference Voting App

Jednoduchá hlasovacia stránka pre výber prednášok, ktorá zbiera hlasy cez Google Apps Script do Google Sheetu.

## Ako to funguje

- Statický `index.html` (GitHub Pages) → Google Apps Script → Google Sheet
- Hlasy sa ukladajú ako riadky: `timestamp | meno | votes (JSON)`
- Výsledky sa počítajú live z listu: `yes=2b, maybe=1b`

## Setup pre nové hlasovanie

### 1. Google Sheet + Apps Script

1. Vytvor nový Google Sheet
2. **Rozšírenia → Apps Script** → vlož kód zo sekcie nižšie
3. **Deploy → New deployment**
   - Type: `Web app`
   - Execute as: `Me`
   - Who has access: **`Anyone`** ← kritické, inak login redirect
4. Skopíruj URL deployu

### 2. HTML

- Zmeň `SHEET_URL` na novú URL
- Zmeň `TALKS` pole s novými prednáškami
- Zmeň názov/rok v headeri

### 3. GitHub Pages

- Pushni `index.html` do repozitára
- Settings → Pages → Deploy from branch `main`

---

## Apps Script kód

```javascript
const SHEET_NAME = 'Votes';

function doGet(e) {
  const action = e.parameter.action;
  const callback = e.parameter.callback;
  let result;

  if (action === 'vote') {
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    let sheet = ss.getSheetByName(SHEET_NAME);
    if (!sheet) sheet = ss.insertSheet(SHEET_NAME);
    sheet.appendRow([e.parameter.timestamp, e.parameter.name, e.parameter.votes]);
    result = { ok: true };

  } else if (action === 'results') {
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    const sheet = ss.getSheetByName(SHEET_NAME);
    if (!sheet) {
      result = { voters: [], scores: {} };
    } else {
      const rows = sheet.getDataRange().getValues();
      const voters = [];
      const scores = {};
      for (const row of rows) {
        const [ts, name, votesJson] = row;
        if (!name) continue;
        voters.push(name);
        const votes = JSON.parse(votesJson || '{}');
        for (const [id, val] of Object.entries(votes)) {
          if (!scores[id]) scores[id] = { yes: 0, maybe: 0, no: 0 };
          if (scores[id][val] !== undefined) scores[id][val]++;
        }
      }
      result = { voters, scores };
    }

  } else {
    result = { error: 'unknown action' };
  }

  const json = JSON.stringify(result);

  // iFrame postMessage výstup (obchádza CORS/redirect problém)
  if (e.parameter.output === 'html') {
    return HtmlService.createHtmlOutput(
      '<script>window.parent.postMessage(' + json + ', "*");<\/script>'
    ).setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL);
  }

  const output = callback ? `${callback}(${json})` : json;
  const mime = callback ? ContentService.MimeType.JAVASCRIPT : ContentService.MimeType.JSON;
  return ContentService.createTextOutput(output).setMimeType(mime);
}
```

---

## Technické poznámky (prečo takto)

### CORS problém s Google Apps Script
GAS `/exec` endpoint robí **302 redirect** na `script.googleusercontent.com`.
Tento redirect má `X-Content-Type-Options: nosniff` + non-JS content-type → browser blokuje.

**Riešenie pre čítanie výsledkov:**
1. Skúsi `fetch` (funguje ak CORS OK)
2. Fallback: skrytý `<iframe>` načíta GAS stránku, tá pošle dáta cez `window.parent.postMessage()`

**Riešenie pre odoslanie hlasov:**
- `fetch` s `mode: 'no-cors'` — dáta prejdú, odpoveď nepotrebujeme

### Persistencia
`localStorage` — ukladá meno, hlasy a stav odovzdania. Prežije refresh stránky.

### Deployment — časté chyby
| Chyba | Príčina | Fix |
|-------|---------|-----|
| Redirect na Google login | "Who has access" nie je `Anyone` | Zmeniť v deployment settings |
| Stará URL po update | Vytvorili ste New deployment namiesto edit | Edit existujúci, New version |
| Zmeny sa neprejavia | Zabudli ste urobiť New version pri deployi | Deploy → Manage → Edit → New version |
