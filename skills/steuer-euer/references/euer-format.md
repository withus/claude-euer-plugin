# EÜR-Dateiformat: Referenz

## Dateistandort

```
STEUER_ROOT/[Jahr]/EÜR [Jahr].xlsx
```

`STEUER_ROOT` steht in `config.json` (Feld `steuer_root`).

Beispiel: `<STEUER_ROOT>/2024/EÜR 2024.xlsx`

**Achtung:** Es können auch ältere Versionen existieren (`EÜR 2022_1.xlsx`, `EÜR 2022_2.xlsx`). Immer die Datei OHNE Suffix verwenden, das ist die aktuellste.

**Sonderfall:** Ältere Jahres-EÜRs können zusätzlich direkt im `STEUER_ROOT` liegen. Die im Jahresordner ist maßgeblich.

## Spaltenstruktur: Sheet "Gesamt"

| Spalte | Header | Typ | Beschreibung |
|--------|--------|-----|--------------|
| A | Datum | Datum (DD.MM.YYYY) | Buchungsdatum (= Zahlungsdatum, nicht Rechnungsdatum) |
| B | Belegnr. | Text | Rechnungsnummer oder Referenz |
| C | Beschreibung | Text | Lieferant/Kunde + Kurzbeschreibung |
| D | Einnahmen Brutto | Zahl (EUR) | Brutto-Einnahmen (nur bei Einnahmen) |
| E | Einnahmen Netto | Zahl (EUR) | Netto-Einnahmen |
| F | Einnahmen USt | Zahl (EUR) | Umsatzsteuer der Einnahmen |
| G | Ausgaben Brutto | Zahl (EUR) | Brutto-Ausgaben (nur bei Ausgaben) |
| H | Ausgaben Netto | Zahl (EUR) | Netto-Ausgaben |
| I | Ausgaben USt | Zahl (EUR) | Vorsteuer der Ausgaben |
| J | Kategorie | Text | Buchungskategorie (optional) |

**Zeile 1:** Header-Zeile
**Ab Zeile 2:** Buchungsdaten, chronologisch sortiert
**Letzte Zeile:** Summen (SUM-Formeln)

## Spaltenstruktur: Sheet "Unklar"

Spalten A bis J identisch mit "Gesamt", zusätzlich:

| Spalte | Header | Typ | Beschreibung |
|--------|--------|-----|--------------|
| K | Quelle | Text | `Bank-<id>` (id des Kontos aus `config.json`, z. B. `Bank-geschaeftskonto`), `KK`, `Dump`, `Gmail` |
| L | Grund | Text | Warum unklar (s. u.) |

### Mögliche Gründe im Unklar-Tab

- `Unbekannter Lieferant`: Lieferant nicht in der Datenbank
- `Empfänger nicht Rechnungsempfänger`: Rechnung auf jemand anders ausgestellt
- `Zahlungsdienst ohne Empfänger`: Zahlungsdienst-Buchung ohne erkennbaren Händler
- `Betrag unleserlich`: OCR konnte den Betrag nicht lesen
- `Datum unleserlich`: OCR konnte das Datum nicht lesen
- `Privat oder geschäftlich unklar`: z. B. bei einem Online-Versandhändler, könnte beides sein
- `Doppelbuchung möglich`: Ähnliche Buchung bereits vorhanden
- `Neuer Lieferant`: Erstmalig aufgetaucht, bitte kategorisieren

## Berechnungsregeln

### Einnahmen (selbst gestellte Rechnungen)

```
Brutto = Rechnungsbetrag
Netto  = ROUND(Brutto / 1.19, 2)
USt    = Brutto - Netto
```

Alle Ausgangsrechnungen des Nutzers haben in dieser Vorlage 19% USt.

### Ausgaben Deutschland (19% USt)

```
Brutto = Rechnungsbetrag
Netto  = ROUND(Brutto / 1.19, 2)
USt    = Brutto - Netto     ← Vorsteuer, abzugsfähig
```

### Ausgaben Deutschland (7% USt)

```
Brutto = Rechnungsbetrag
Netto  = ROUND(Brutto / 1.07, 2)
USt    = Brutto - Netto     ← Vorsteuer, abzugsfähig
```

Gilt z. B. für: Fachliteratur (Beispiel-Lieferant "Beispiel Fachverlag GmbH" in `lieferanten.json`)

### Ausgaben Ausland (0% USt)

```
Brutto = Rechnungsbetrag
Netto  = Brutto              ← IDENTISCH!
USt    = 0,00 EUR            ← Kein Vorsteuerabzug
```

Gilt für: Lieferanten mit Sitz im Ausland, die keine deutsche Umsatzsteuer ausweisen (Beispiel-Lieferant "Muster Software AG" in `lieferanten.json`).

## Formatierung

- Währung: EUR, 2 Dezimalstellen
- Schriftart: Calibri 11pt (Standard)
- Header: Fett
- Summenzeile: Fett, mit SUM()-Formeln
- Beträge: Rechtsbündig
- Datumsformat in Zellen: DD.MM.YYYY

## Summenformeln

Die letzte Zeile enthält Summen. **Beim Hinzufügen neuer Zeilen MUSS der Bereich der SUM-Formeln angepasst werden!**

Beispiel (wenn Daten in Zeilen 2 bis 220):
```
D[Summe] = SUM(D2:D220)
E[Summe] = SUM(E2:E220)
...usw. für alle Spalten D bis I
```

**KRITISCH:** Decken die SUM-Formeln nicht alle Datenzeilen ab, wird der Gewinn falsch berechnet. Nach jedem Einfügen prüfen, dass die Formeln den vollen Bereich bis zur letzten Datenzeile verwenden!

## Backup-Strategie

Vor jeder Änderung:
```
Kopiere: EÜR [Jahr].xlsx → EÜR [Jahr]_backup_[YYYYMMDD].xlsx
```

Die Backup-Datei im selben Ordner speichern.

## Bekannte Probleme

1. **Unicode-Dateinamen:** Das Dateisystem-Werkzeug hat manchmal Probleme mit `ü` in `EÜR`. Falls die Datei nicht gefunden wird, nach allen .xlsx-Dateien im Ordner suchen.
2. **Verwaiste Dateien:** In manchen Jahresordnern liegt die EÜR auch im `STEUER_ROOT`. Die im Jahresordner ist maßgeblich.
3. **Mehrere Versionen:** `EÜR 2022.xlsx`, `EÜR 2022_1.xlsx`, `EÜR 2022_2.xlsx`: immer die ohne Suffix verwenden.
4. **Temp-Dateien:** `~$EÜR 2022_1.xlsx` sind Lock-Dateien von Excel, ignorieren.
