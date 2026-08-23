# Splint Tracker

Ein einzelnes, eigenständiges HTML/JS-Tool zur Nachverfolgung von [Splint Invest](https://www.splintinvest.com)-Investments (fraktionierte Sachwerte: Trading Cards, Uhren, Kunst, Wein, Autos etc.). Teil der [Munotstadt](https://github.com/Munotstadt)-Plattform-Suite.

Kein eigenes Backend, kein Build-Schritt — läuft als statische Seite über GitHub Pages, hinter Google-Login geschützt. Die Daten selbst liegen nicht mehr im Repo, sondern in einer [Neon](https://neon.com) Postgres-Datenbank und werden über die Neon Data API gelesen/geschrieben.

## Live

`https://splint.gnae.app`

Zugriff nur nach Google-Anmeldung mit einem freigeschalteten Konto (siehe [Zugriff & Login](#zugriff--login)).

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
Alle Transaktionen mit Filter/Suche, sortierbar. Spalten **Day1Profit** und **FX Rate** (jeweils manueller Input, direkt in der Tabelle erfasst und automatisch synchronisiert).

### Wertschriften (Security-Detail)
- Editierbare Stammdaten: Asset-Name, Kategorie, Subkategorie, Asset Category_old, **eROI (%)**, Notizen
- KPI-Kacheln: Splints, Wert, Wert/Splint, Return absolut/relativ, IRR p.a. (Newton-Verfahren), **eROI vs. IRR**
- **Kursverlauf-Chart**: Wert/Splint als Linie (linke Achse, 2 Dezimalstellen) + monatliche Veränderung in % als Säulen (rechte Achse, 1 Dezimalstelle, mit Wert-Label an der Säule). Punkte/Säulen sind antippbar (touch-freundlich statt Hover) und zeigen den exakten Wert in einem Readout unter dem Chart
- Transaktionsliste dieses Assets, inkl. editierbarem Day1Profit-Feld
- **Ähnliche Investments**: Tabelle aller anderen Assets mit identischer Kategorie *und* Subkategorie (gleiche Spalten wie Portfolio)

### Performance-Übersicht
Asset × Monat-Matrix. Neuster Monat links, ältere Monate rechts. Umschaltbar zwischen Kursen und Δ%-zum-Vormonat. Im Kurs-Modus: Hintergrund-Heatmap pro Zeile (tiefster Kurs rot, höchster grün, Rot-Gelb-Grün-Verlauf dazwischen).

### Daten verwalten
1. **Neon-Verbindung**: Statusanzeige, manueller "Reload from Neon"-Button, "Export as CSV" als lokale Sicherung (kein Verbindungs-Setup nötig — läuft automatisch über den eingeloggten Google-Account)
2. **Splint-Exporte hochladen**: zwei Buttons für die Original-CSV-Exporte aus der Splint-App/-Website — zweistufig (Datei wählen → Upload-Button drücken), fügt nur neue Daten hinzu (nie überschreiben)

## Zugriff & Login

- Anmeldung ausschliesslich über **Google-Sign-in** (kein Passwort-Login)
- Authentifizierung läuft über **Neon Auth**, gehostet auf demselben Neon-Projekt wie die Datenbank
- **Nicht jedes Google-Konto hat Zugriff auf die Daten**: Nach dem Login prüft die App per Testabfrage, ob das Konto auf der serverseitigen Allowlist steht (durchgesetzt über eine Row-Level-Security-Policy in Postgres, nicht nur im Frontend). Konten, die nicht freigeschaltet sind, sehen nach dem Google-Login die Meldung *"this account is not authorized to view this data"* und erhalten keine Daten
- Freigeschaltete Konten werden direkt in der Datenbank verwaltet (aktuell: `ph.gnaedinger@gmail.com`). Weitere Konten müssen serverseitig ergänzt werden — kein Self-Service-Signup

## Architektur

- **Persistenz**: Neon Postgres, drei Tabellen (`splint_masterdata`, `splint_prices`, `splint_transactions`), Zugriff via [Neon Data API](https://neon.com/docs/data-api/overview) (PostgREST-artige REST-Schnittstelle)
- **Auth**: [Neon Auth](https://neon.com/docs/auth/overview) (Managed Better Auth), Google OAuth, JWT-basiert; die Data API validiert den JWT bei jedem Request und wertet Row-Level-Security-Policies aus
- **Echtzeit-Sync**: jede Datenänderung (Upload, Stammdaten-Edit, Day1Profit, FX Rate, eROI) synchronisiert sofort alle drei Tabellen
- **Kein lokaler Speicher**: kein localStorage, keine Tokens im Browser — jeder Data-API-Aufruf holt sich vorher ein frisches JWT von Neon Auth
- **Bestände**: werden nicht separat gespeichert, sondern zu jedem Zeitpunkt aus dem Transaktions-Ledger (Durchschnittskosten-Methode) neu berechnet — dadurch bleiben Bestände auch ohne erneuten Opportunities-Upload nach einem reinen Neon-Load korrekt
- **Charts**: reines SVG, keine externen Libraries
- **Design**: Space Grotesk / IBM Plex Mono / Inter, Swiss Red `#E30613`, passend zu den übrigen Munotstadt-Dashboards

## Datenschema (Neon Postgres, Schema `public`)

**`splint_prices`** — ein Kurspunkt pro Asset und Monat (Unique-Constraint auf `asset_id, month`)
```
asset_id, asset_name, month, price_date, price, currency
```

**`splint_transactions`** — alle Transaktionen (Primärschlüssel `transaction_id`)
```
transaction_id, transaction_type, asset_id, transaction_date,
money_amount, money_currency, price_per_splint, price_currency,
fees, fees_currency, confirmation_doc, day1_profit, fx_rate
```

**`splint_masterdata`** — manuell gepflegte Zusatzdaten pro Asset (Primärschlüssel `asset_id`)
```
asset_id, asset_name, asset_category, asset_subcategory, asset_category_old,
notes, release_date, min_horizon, max_horizon, eroi
```

Alle Datumsfelder in Postgres sind ISO-Dates (`YYYY-MM-DD`); im Frontend werden sie zur Anzeige nach `DD.MM.YYYY` konvertiert. Zeitzone für Zeitstempel im UI: Europe/Zurich.

## Setup (für einen Fork/Neuaufbau)

1. Neon-Projekt erstellen, Datenbank + die drei Tabellen oben anlegen
2. **Neon Auth** aktivieren (Google OAuth), Redirect-/Trusted-Origin auf die eigene Hosting-Domain setzen
3. **Neon Data API** aktivieren, RLS auf allen drei Tabellen einschalten und Policies anlegen, die den Zugriff auf die gewünschten Konten beschränken (siehe `neon_auth.user` → E-Mail-Abgleich)
4. `index.html` in einem GitHub-Repo ablegen, **Settings → Pages** aktivieren (Deploy from branch → `main` → `/ (root)`), optional Custom Domain setzen
5. In `index.html` die Konstanten `NEON_AUTH_URL` und `NEON_DATA_API_URL` auf das eigene Neon-Projekt anpassen

## Datenquelle

Es existiert **keine öffentliche/dokumentierte API** von Splint Invest. Die Aktualisierung erfolgt manuell über die zwei CSV-Exporte aus der Splint-App/-Website:
- **Investment Opportunities Export** (aktuelle Positionen inkl. heutigem Kurs)
- **Activities Export** (vollständiges Transaktionsprotokoll)

## Bekannte Grenzen

- Bestandsbewertung zwischen zwei Opportunities-Uploads basiert auf dem letzten bekannten Monatskurs (kein Live-Kurs)
- `eROI`, `Day1Profit`, `FX Rate`, Stammdaten (Kategorie/Notizen/Horizont) sind rein manuelle Felder ohne externe Quelle
- Kein Multi-User-Konflikt-Handling — bei gleichzeitigem Schreiben gilt Last-Write-Wins
- Freischaltung neuer Nutzer:innen ist aktuell ein manueller, serverseitiger Schritt (kein Admin-UI in der App)
