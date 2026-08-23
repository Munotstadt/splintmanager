# Splint Tracker

Ein einzelnes, eigenständiges HTML/JS-Tool zur Nachverfolgung von [Splint Invest](https://www.splintinvest.com)-Investments (fraktionierte Sachwerte: Trading Cards, Uhren, Kunst, Wein, Autos etc.). Teil der [Munotstadt](https://github.com/Munotstadt)-Plattform-Suite.

Keine Datenbank, kein Backend, kein Build-Schritt — läuft direkt als statische Seite über GitHub Pages und speichert seine Daten als CSV-Dateien im selben Repo (via GitHub Contents API).

## Live

`https://<owner>.github.io/splintdatacollector/` (nach Aktivierung von GitHub Pages, siehe unten)

## Funktionen

### Dashboard
- KPI-Kacheln: aktueller Wert, offene Investitionen, unrealisierte/realisierte Performance, Portfolio-IRR p.a.
- **Portfolio Value Over Time**: zwei Linien — Wert (Bestand × Kurs zum jeweiligen Monatsende) und Kosten (offene Kostenbasis zum jeweiligen Monatsende), historisch korrekt berechnet aus dem Transaktions-Ledger
- **By Category**: Value/Cost/Return/IRR gruppiert nach Asset-Kategorie
- Top Positions by Value
- Cash Flow-Tabelle (neuster Monat zuoberst, Investments positiv/grün, Exits/Sales negativ/rot)

### Portfolio
Vollständige Tabelle aller Assets (inkl. verkaufter): Splints, Wert, Kosten, Return absolut/relativ, IRR p.a., **eROI** (manuell erfassbar) und **eROI vs. IRR**, 1. Kauf, Exit-Datum, **Horizont bis** (Release-Jahr + Ø Min/Max-Investitionshorizont, `n/a` bei verkauften Assets), Kategorie/Subkategorie. Durchsuchbar, sortierbar, Filter Alle/Gehalten/Verkauft, Total-/Subtotal-Zeile.

### Transaktionen
Alle Transaktionen mit Filter/Suche, sortierbar. Spalte **Day1Profit** (EUR, manueller Input, wird direkt in der Tabelle erfasst und automatisch synchronisiert). Spalten **Price CHF** (editierbarer FX-Kurs, siehe Abschnitt "FX / CHF-Umrechnung") und **Value CHF** (berechnet). Ganz rechts: **Booking Entries** — nur bei Kauftransaktionen ein Excel-Icon ("Kaufbeleg" beim Hover), das eine `splint_entries.xlsx` mit zwei Tabs (`splint_accounting_entry`, `splint_security_entry`) für genau diese Transaktion generiert, gemäss fixer Buchungsvorlage (siehe unten).

### Wertschriften (Security-Detail)
- Editierbare Stammdaten: Asset-Name, Kategorie, Subkategorie, Asset Category_old, **eROI (%)**, Notizen
- KPI-Kacheln: Splints, Wert, Wert/Splint, Return absolut/relativ, IRR p.a. (Newton-Verfahren), **eROI vs. IRR**
- **Kursverlauf-Chart**: Wert/Splint als Linie (linke Achse, 2 Dezimalstellen) + monatliche Veränderung in % als Säulen (rechte Achse, 1 Dezimalstelle, mit Wert-Label an der Säule). Punkte/Säulen sind antippbar (touch-freundlich statt Hover) und zeigen den exakten Wert in einem Readout unter dem Chart
- Transaktionsliste dieses Assets, inkl. editierbarem Day1Profit-Feld
- **Ähnliche Investments**: Tabelle aller anderen Assets mit identischer Kategorie *und* Subkategorie (gleiche Spalten wie Portfolio)

### Performance-Übersicht
Asset × Monat-Matrix. Neuster Monat links, ältere Monate rechts. Umschaltbar zwischen Kursen und Δ%-zum-Vormonat. Im Kurs-Modus: Hintergrund-Heatmap pro Zeile (tiefster Kurs rot, höchster grün, Rot-Gelb-Grün-Verlauf dazwischen).

### Daten verwalten
1. **GitHub-Verbindung**: Owner/Repo/Branch/Fine-grained PAT, optional im Browser gemerkt (localStorage, standardmässig aus)
2. **Splint-Exporte hochladen**: zwei Buttons für die Original-CSV-Exporte aus der Splint-App/-Website — zweistufig (Datei wählen → Upload-Button drücken), fügt nur neue Daten hinzu (nie überschreiben)
3. **Erweitert/Fallback**: manueller CSV-Up-/Download, Reset

## Architektur

- **Persistenz**: 3 CSV-Dateien im Repo unter `data/`, via GitHub Contents API (base64 + SHA-Versionierung) gelesen/geschrieben
- **Echtzeit-Sync**: jede Datenänderung (Upload, Stammdaten-Edit, Day1Profit, eROI) synchronisiert sofort alle 3 Dateien, falls verbunden
- **Auto-Load**: bei gemerkter Verbindung wird der letzte GitHub-Stand automatisch beim Öffnen geladen
- **Bestände**: werden nicht separat gespeichert, sondern zu jedem Zeitpunkt aus dem Transaktions-Ledger (Durchschnittskosten-Methode) neu berechnet — dadurch bleiben Bestände auch ohne erneuten Opportunities-Upload nach einem reinen GitHub-Load korrekt
- **Charts**: reines SVG, keine externen Libraries
- **Design**: Space Grotesk / IBM Plex Mono / Inter, Swiss Red `#E30613`, passend zu den übrigen Munotstadt-Dashboards

## Datenschema (`data/*.csv`)

**`prices.csv`** — ein Kurspunkt pro Asset und Monat
```
AssetID, Asset Name, Month, Date, Price, Currency
```

**`transactions.csv`** — alle Transaktionen (dedupliziert nach Transaction ID)
```
Transaction ID, Transaction Type, Asset ID, Date Of Transaction,
Money Amount, Purchase/Sale Price Per Splint, Fees,
Purchase/Sale Confirmation, Day1Profit, FX Rate
```

**`stammdaten.csv`** — manuell gepflegte Zusatzdaten pro Asset
```
AssetID, Asset Name, Asset Category, Asset SubCategory, Asset Category_old,
Notes, Release Date, Min Horizon, Max Horizon, eROI
```

Alle Daten folgen der Munotstadt-Konvention: Datum `DD.MM.YYYY`, bei Bedarf mit Zeit `HH:MM:SS`; Zeitzone Europe/Zurich.

## FX / CHF-Umrechnung

Splint liefert keine CHF-Kurse — die Umrechnung kommt aus einem separaten Munotstadt-Repo:

```
https://raw.githubusercontent.com/Munotstadt/accountingdatacollector/refs/heads/main/fx_rates.csv
```

**Schema dieser externen Datei** (Fremdquelle, nicht Teil von `splintdatacollector`):
```
Date, Currency, Rate, CollectedAt
```
`Date` = `DD.MM.YYYY` (ohne Zeit), `Currency` = 3-Buchstaben-Code (z. B. `EUR`, `USD`, `GBP` — direkt der Kurs zu CHF, kein Paar-Name wie `EUR/CHF`), `Rate` = Kurs, `CollectedAt` = Erfassungszeitpunkt.

**Ablauf im Splint Tracker:**
1. Beim Öffnen wird diese CSV einmalig per `fetch()` direkt im Browser geladen (`loadFxRates()`), pro Währung nach Datum sortiert im Speicher gehalten.
2. Für jede Transaktion ohne eigenen `FX Rate`-Wert wird automatisch der **letzte bekannte Kurs an oder vor dem Transaktionsdatum** gesucht (`chfRateForCurrencyAtDate()`) und in die Spalte `FX Rate` eingetragen.
3. Danach ist `FX Rate` ein **normales, persistiertes und manuell editierbares Feld** in `transactions.csv` — es wird nie erneut automatisch überschrieben, auch nicht bei künftigen Ladevorgängen.
4. Liegt eine Transaktion **vor dem ältesten bekannten Datenpunkt** dieser Währung, bleibt `FX Rate` (und damit `Value CHF`) bewusst **leer** — es wird nie mit einem zeitlich zu weit entfernten Kurs geschätzt.
5. Der Auto-Fill läuft bei jedem Rendern erneut (self-healing), betrifft aber immer nur Zeilen, die noch keinen Wert haben.

`Value CHF` selbst wird nie gespeichert, sondern immer live aus `Money Amount × FX Rate` berechnet.

## Booking Entries (`splint_entries.xlsx`)

Nur bei **Kauftransaktionen** (Primary market purchase, Marketplace purchase) lässt sich über das Excel-Icon ganz rechts in der Transaktionen-Tabelle eine Buchungsvorlage generieren (via [SheetJS](https://sheetjs.com/), geladen von cdnjs.cloudflare.com — reine Client-Erzeugung, kein Server). Bei Verkäufen wird kein Icon angezeigt. Struktur exakt nach der von Philipp gelieferten Vorlage:

**Tab `splint_accounting_entry`** — 2 Zeilen (Doppelbuchung):
| Feld | Kauf: MainID 1060 (Cash) | Kauf: MainID 1816 (Investment) |
|---|---|---|
| AmtCHF / AmtLC | negativ (Geldabfluss) | positiv (Bestandszunahme) |

Bei einem Verkauf werden die Vorzeichen gespiegelt (1060 positiv/Geldzufluss, 1816 negativ/Bestandsabnahme) — **Annahme**, da die Vorlage nur ein Kaufbeispiel enthielt.

**Tab `splint_security_entry`** — 1 Zeile: Quantity positiv bei Kauf, negativ bei Verkauf; AmtCHF/AmtLC immer als Betrag.

Konstante Werte (nicht `[in eckigen Klammern]` in der Vorlage, daher unverändert für jede Transaktion übernommen): `MainID 1060/1816/21000`, `EntryNo 8352`, `TrxArt 13946`, `PartyNo "Splint Real Assets / Art"`, `ProjectID "INV_Splint_2023 ff."` bzw. `10`, `TrxTypeID 222`. **Diese Codes stammen alle aus dem einen Kauf-Beispiel der Vorlage** — falls Verkäufe einen anderen `TrxTypeID` oder eine andere `EntryNo`-Logik brauchen, bitte Bescheid geben.

Variable Felder: `Comment`/`Comments` = `Kategorie: Asset-Name`, `AmtCHF`/`Price_CHF` = `Money Amount × FX Rate` (2 Dezimalstellen), `AmtLC` = `Money Amount` (2 Dezimalstellen), `Valuta`/`Date` = Transaktionsdatum.

## Setup

1. Neues **privates oder öffentliches** Repo `splintdatacollector` erstellen
2. `index.html` als Datei im Root hochladen
3. **Settings → Pages** → Deploy from branch → `main` → `/ (root)`
4. Fine-grained Personal Access Token generieren: github.com → Settings → Developer settings → Personal access tokens → Fine-grained tokens, Scope auf dieses Repo, Permission **Contents: Read and write**
5. Im Tool unter "Daten verwalten" → GitHub-Verbindung Owner/Repo/Branch/Token eintragen, optional "Verbindung merken" aktivieren

## Datenquelle

Es existiert **keine öffentliche/dokumentierte API** von Splint Invest. Die Aktualisierung erfolgt manuell über die zwei CSV-Exporte aus der Splint-App/-Website:
- **Investment Opportunities Export** (aktuelle Positionen inkl. heutigem Kurs)
- **Activities Export** (vollständiges Transaktionsprotokoll)

## Bekannte Grenzen

- Bestandsbewertung zwischen zwei Opportunities-Uploads basiert auf dem letzten bekannten Monatskurs (kein Live-Kurs)
- `eROI`, `Day1Profit`, Stammdaten (Kategorie/Notizen/Horizont) sind rein manuelle Felder ohne externe Quelle
- `FX Rate`/`Value CHF` bleiben leer für Transaktionen vor dem ältesten Datenpunkt in `accountingdatacollector/fx_rates.csv`
- Kein Multi-User-Konflikt-Handling über die einfache SHA-Versionierung hinaus (Last-Write-Wins bei gleichzeitigem Schreiben)
- Speicherort für `splint_entries.xlsx`: aus Sicherheitsgründen kann keine Website einen exakten Ordnerpfad vorgeben. In Chrome/Edge öffnet sich der native "Speichern unter"-Dialog (Ordner selbst wählen — der Browser merkt sich diesen meist für spätere Speicherungen von dieser Seite); in Browsern ohne File System Access API (Safari, Firefox) landet die Datei automatisch im Standard-Downloads-Ordner
