# Forest AI

Kérdezz a Forest adataidról magyarul, és Claude kikeresi a választ: partnerek,
kapcsolattartók, lehetőségek, tevékenységek, számlák.

## Telepítés (claude.ai és Claude Desktop)

1. Nyisd meg a **Customize → Plugins** oldalt, és válaszd az **Add marketplace** lehetőséget.
2. Írd be: `smilingsoftware/forest-ai`
3. A megjelenő listában telepítsd a **forest** plugint.
4. Kapcsold be a **Sync automatically** opciót, így az új verziók maguktól megérkeznek.

A plugin a fiókodhoz kötődik, ezért a Claude Desktopban is megjelenik.

## BigQuery-kapcsolat beállítása

A Forest adatai a Google BigQuery-ben vannak. Claude a saját Google-fiókoddal éri el őket:

1. **Settings → Connectors → Google Cloud BigQuery → Connect**
2. Jelentkezz be azzal a Google-fiókkal, amelyhez a Forest hozzáférést adott.

A fióknak két jogosultság kell (ezt a Forest állítja be neked):

- **BigQuery Job User** a `forestcrm-report-data` projekten,
- **BigQuery Data Viewer** a `forestcrm-report-data.public` adatkészleten.

Az első kérdés előtt Claude ellenőrzi a hozzáférést, és ha valami hiányzik, megmondja, mit kell
beállítani.

## Használat

Egyszerűen kérdezz, például:

- „Hány nyitott feladata van az egyes kereskedőknek?”
- „Melyik partnerünktől jött a legtöbb bevétel idén?”

## Claude Code

```
/plugin marketplace add smilingsoftware/forest-ai
/plugin install forest@forest-ai
```
