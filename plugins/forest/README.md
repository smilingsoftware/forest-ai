# Forest

Kérdezz a Forest adataidról magyarul, és Claude kikeresi a választ: partnerek,
kapcsolattartók, lehetőségek, tevékenységek, számlák.

## Tartalom

- **Skillek** (`skills/`):
  - `adatok`: magyar nyelvű kérdések megválaszolása a Forest adatokból (partnerek,
    kapcsolattartók, lehetőségek, tevékenységek, számlák). Ha több Forest-előfizetéshez is hozzáférsz,
    Claude megkérdezi, melyikről szól a kérdés.
  - `forest-check`: ellenőrzi, hogy a BigQuery connector be van-e kapcsolva, és megvannak-e
    a szükséges jogosultságok.

Használat: egyszerűen kérdezz, például „Hány nyitott feladata van az egyes kereskedőknek?”

A lekérdezések a felhasználó saját **Google Cloud BigQuery** connectorán keresztül futnak.

## Előfeltételek

- A **Google Cloud BigQuery** connector bekapcsolva és bejelentkezve
  (claude.ai: **Settings → Connectors → Google Cloud BigQuery → Connect**).
- A connectorban használt Google-fiók jogosultságai:
  - **BigQuery Job User** a `forestcrm-report-data` projekten (lekérdezés futtatása),
  - **BigQuery Data Viewer** a `forestcrm-report-data.public` adatkészleten (olvasás).

Az első lekérdezés előtt a plugin automatikusan ellenőrzi a hozzáférést, és hiba esetén
megmondja, mit kell beállítani.

## Helyi kipróbálás

```zsh
claude --plugin-dir .
```
