---
name: steuer-euer
description: >
  Verwaltet die Einnahmenüberschussrechnung (EÜR) für Einzelunternehmer und Freelancer.
  Verarbeitet den zentralen Dump-Ordner (Rechnungs-PDFs, Bank-CSVs, Kreditkartenabrechnungen: alles rein,
  Claude sortiert automatisch), scannt Gmail nach fehlenden Rechnungen und aktualisiert die EÜR-Excel-Dateien
  direkt im lokalen Steuerordner. IMMER verwenden bei: Steuern, EÜR, Einnahmenüberschussrechnung, Buchhaltung, Buchungen,
  Rechnungen sortieren, Belege verarbeiten, Steuer-Update, "Steuern aktualisieren", "EÜR updaten",
  "Rechnungen buchen", "Dump verarbeiten", "fehlende Rechnungen", "Belege einsortieren", "Buchhaltung machen",
  "was fehlt noch in der EÜR", "Steuerordner aufräumen", Betriebsausgaben, Eingangsrechnungen,
  Ausgangsrechnungen, Kreditkartenabrechnung auslesen, Bankdaten einlesen, Bank-CSV.
---

# Steuer-EÜR Skill

Automatisierte Buchhaltung für Einzelunternehmer: vom Beleg bis zur fertigen EÜR.

## Konfiguration

- Konfigurationsordner: `${CLAUDE_PLUGIN_DATA}` (laut Plugin-Dokumentation bleibt dieser Ordner bei Plugin-Updates erhalten).
- `config.json` in diesem Ordner enthält Basispfad, Rechnungsempfänger, Konten, CSV-Formate und Kreditkarten-Erkennung. Vorlage: `${CLAUDE_PLUGIN_ROOT}/config.example.json`.
- Die eigene Lieferantenliste liegt als `lieferanten.json` im selben Ordner. Vorlage mit erfundenen Beispieldaten: `references/lieferanten.json`.
- Fehlt eine der beiden Dateien: Vorlage in den Konfigurationsordner kopieren, die Platzhalter in spitzen Klammern gemeinsam mit dem Nutzer ausfüllen, erst danach weiterarbeiten. Mit unausgefüllten Platzhaltern nichts verschieben und nichts buchen.
- Bleibt `${CLAUDE_PLUGIN_DATA}` unaufgelöst im Text stehen: den Nutzer nach dem Ordner fragen, nicht raten.
- Im Folgenden steht `lieferanten.json` für die eigene Liste im Konfigurationsordner.

## Basispfad

```
STEUER_ROOT = <STEUER_ROOT>    (Feld steuer_root in config.json)
```

Alle Pfade in diesem Skill sind relativ zu `STEUER_ROOT`.

---

## Grundprinzipien

### Zufluss-/Abfluss-Prinzip
Die EÜR wird nach dem **Zufluss-/Abfluss-Prinzip** gebucht. Das bedeutet:
- Eine Rechnung vom Dezember 2024, die erst im Januar 2025 bezahlt wird, wird in **2025** gebucht
- Maßgeblich ist immer das **Datum der Zahlung**, NICHT das Rechnungsdatum
- Bei Bank-CSVs: Buchungsdatum = Zahlungsdatum → bestimmt das Jahr
- Bei Dump-PDFs: Wenn ein Zahlungsdatum erkennbar ist, dieses verwenden. Sonst Rechnungsdatum als Fallback, mit Vermerk im Unklar-Tab

### KFZ-Leasing (Beispiel für eine Dauerbuchung mit Deduplizierung)
- Im Beispiel 100% Betriebsausgabe, weil das Fahrzeug ausschließlich betrieblich genutzt wird. Bei privater Mitnutzung ist ein Privatanteil anzusetzen (1-%-Regel oder Fahrtenbuch); das vor der Buchung steuerlich klären
- Monatliche Rate (z. B. 400,00 EUR), gelegentlich abweichende Beträge (Nachzahlungen/Sonderraten)
- Wird über Bankkonto UND Kreditkarte bezahlt → Deduplizierung beachten!
- Der zugehörige Lieferant steht mit Kategorie `KFZ` in `lieferanten.json`

### Interne Umbuchungen (Unterkonten)
- Interne Umbuchungen zwischen eigenen Konten und Unterkonten (z. B. regelmäßige Sparraten, in `config.json` unter `ignoriere_umbuchungen` je Konto beschrieben) → **IMMER IGNORIEREN**
- Diese sind keine echten Ein-/Ausgaben

### Reverse Charge / Auslands-Dienste
- Auslandsdienste (z. B. Cloud-, Software- und KI-Dienste mit Sitz außerhalb Deutschlands) werden ganz normal als Betriebsausgabe gebucht
- Brutto = Netto, USt = 0,00 EUR, wie bisher in den bestehenden EÜR-Dateien
- Kein separates Reverse-Charge-Tracking durch dieses Plugin

### Bestehende Buchungen
- Wenn eine Buchung bereits in der EÜR steht: **Betrag prüfen**
- Bei Abweichung (±0,02 EUR): in den **Unklar-Tab** melden mit Grund "Betragsdifferenz"
- Bei Übereinstimmung: überspringen

### Kategorien
- Kategorien aus den **bestehenden EÜR-Dateien** übernehmen
- Beim ersten Lauf: vorhandene Kategorien pro Lieferant erfassen und in `lieferanten.json` ergänzen
- Für neue Lieferanten ohne bestehende Kategorie: Nutzer fragen

### Automatische EÜR-Erstellung
- Wenn für ein Jahr noch keine EÜR-Datei existiert → **automatisch anlegen** mit korrekter Spaltenstruktur
- Sheets: "Gesamt" + "Unklar"
- Header, Formatierung und leere Summenzeile sofort einrichten
- Siehe `references/euer-format.md` für die exakte Struktur

---

## Ordnerstruktur

```
STEUER_ROOT/
├── Dump/                          ← ZENTRALER EINGANG: Alles hier reinwerfen!
│                                     PDFs, CSVs, KK-Abrechnungen, Claude sortiert
├── [Jahr]/
│   ├── EÜR [Jahr].xlsx            ← Die Einnahmenüberschussrechnung
│   ├── Bank/                      ← Kontoauszüge (PDFs und CSVs)
│   ├── Kreditkarte/               ← KK-Abrechnungen (gescannte PDFs)
│   ├── Eingangsrechnungen/        ← Ausgaben, nach Lieferant sortiert
│   │   ├── Beispiel-Hosting/
│   │   ├── Muster-Software/
│   │   └── ...
│   ├── Ausgangsrechnungen/        ← Einnahmen (selbst gestellt, Nummernformat laut config.json)
│   └── Steuerbescheide/
├── [weitere Jahre, gleiche Struktur]
└── [diverse alte Dateien im Root]  ← Werden bei Bedarf auch verarbeitet
```

---

## Workflow-Übersicht

Der Skill hat 5 Module, die einzeln oder zusammen ausgeführt werden können:

| # | Modul | Trigger-Beispiele |
|---|-------|-------------------|
| 1 | **Dump verarbeiten** | "Dump verarbeiten", "Belege sortieren" |
| 2 | **Bankdaten verarbeiten** | "Bankdaten einlesen", "Bank-CSV abgleichen" |
| 3 | **Kreditkarte auslesen** | "Kreditkarte verarbeiten", "KK auslesen" |
| 4 | **Gmail scannen** | "Gmail nach Rechnungen durchsuchen" |
| 5 | **EÜR-Report** | "EÜR Status", "Was fehlt noch?" |

Bei "Steuern aktualisieren" oder "EÜR updaten" → alle Module der Reihe nach ausführen.

---

## Modul 1: Dump-Ordner verarbeiten (ZENTRALES MODUL)

Der Dump-Ordner (`STEUER_ROOT/Dump/`) ist der zentrale Eingang. Der Nutzer wirft hier ALLES rein:
- Rechnungs-PDFs (Eingangs- und Ausgangsrechnungen)
- Bank-CSVs
- Kreditkartenabrechnungen (gescannte PDFs)
- Kontoauszüge (PDFs)
- Sonstige Belege

Claude muss **automatisch erkennen**, was jede Datei ist, und entsprechend verarbeiten.

### Schritt 1: Dump-Ordner lesen und Dateien klassifizieren

```
Pfad: STEUER_ROOT/Dump/
```

Alle Dateien auflisten. Falls der Ordner nicht existiert, erstellen. Dann jede Datei nach Typ einordnen:

| Erkennungsmerkmal | Dateityp | Verarbeitung |
|-------------------|----------|--------------|
| `.csv` + Spaltenköpfe passen zu einem Format aus `csv_formate` in `config.json` (`erkennung_header_enthaelt`) oder Dateiname enthält einen Wert aus `csv_dateiname_enthaelt` | **Bank-CSV** | → Modul 2 (Parser dieses Formats, Konto laut `konten`) |
| `.pdf` + Dateiname enthält einen Wert aus `kreditkarte.abrechnung_dateiname_enthaelt` oder Inhalt zeigt KK-Abrechnungsformat | **Kreditkartenabrechnung** | → Modul 3 (KK-OCR) |
| `.pdf` + Dateiname enthält einen Wert aus `kontoauszug_dateiname_enthaelt` | **Kontoauszug** | → Verschieben nach `[Jahr]/Bank/` |
| `.pdf` + Rechnungsinhalt erkennbar | **Rechnung (Eingang oder Ausgang)** | → Schritt 2 (PDF-Klassifizierung) |
| `.pdf` + Steuerbescheid/Finanzamt erkennbar | **Steuerbescheid** | → Verschieben nach `[Jahr]/Steuerbescheide/` |
| Unbekannt | **Unklar** | → Im Dump belassen, melden |

Passt eine CSV zu keinem beschriebenen Format: nicht raten, Datei im Dump belassen und melden (siehe Modul 2, Abschnitt "CSV-Formate").

### Schritt 2: PDFs lesen und klassifizieren

Für jede Rechnungs-PDF:

1. **Lesen:** ein Dateisystem-Werkzeug nutzen, das PDFs als Bild liefert (z. B. `Filesystem:read_media_file`) → Claude kann PDFs direkt als Bild lesen
2. **Falls unleserlich** (gescannte Bilder): ein Bild-/OCR-Werkzeug nutzen (z. B. `nano-banana-pro:describe_image`)
3. **Extrahieren:**
   - Rechnungsdatum
   - Zahlungsdatum (falls erkennbar → maßgeblich für EÜR-Jahr, Zufluss-Prinzip!)
   - Rechnungsnummer
   - Rechnungssteller (wer hat die Rechnung geschrieben?)
   - Empfänger (an wen ist die Rechnung adressiert?)
   - Beträge: Brutto, Netto, USt (19% / 7% / 0%)
   - Ob auf den Rechnungsempfänger aus `config.json` (`unternehmer.rechnungsempfaenger`) ausgestellt (→ sonst NICHT buchen, Unklar-Tab)
   - Ob es eine Ausgangsrechnung ist (VOM Rechnungsempfänger aus `config.json` AN jemanden)

### Schritt 3: Klassifizieren

| Feld | Logik |
|------|-------|
| **Typ** | Eingangsrechnung (Ausgabe) oder Ausgangsrechnung (Einnahme)? |
| **Jahr** | Aus ZAHLUNGSDATUM → Zielordner bestimmen (Zufluss-Prinzip!) |
| **Lieferant** | Gegen `lieferanten.json` abgleichen |
| **USt-Behandlung** | Deutsch (19%/7%) → VSt-Abzug. Ausland → Netto = Brutto, USt = 0 |
| **Kategorie** | Aus bestehender EÜR oder `lieferanten.json` übernehmen |

**Entscheidungsbaum:**
- Rechnungssteller = bekannter Lieferant → automatisch zuordnen
- Empfänger ≠ Rechnungsempfänger aus `config.json` und ≠ leer → **Unklar-Tab** (könnte einen anderen Rechnungsempfänger betreffen)
- Rechnungssteller steht unter `ignorieren` in `lieferanten.json` → **IGNORIEREN** (nicht die eigene Firma)
- Betrag oder Datum nicht lesbar → **Unklar-Tab**
- Unbekannter Lieferant → **Unklar-Tab** mit Hinweis "Neuer Lieferant"

### Schritt 4: Datei verschieben

```
Bank-CSV           → STEUER_ROOT/[Jahr]/Bank/[Dateiname]  (nach Verarbeitung)
KK-Abrechnung      → STEUER_ROOT/[Jahr]/Kreditkarte/[Dateiname]  (nach Verarbeitung)
Kontoauszug        → STEUER_ROOT/[Jahr]/Bank/[Dateiname]
Eingangsrechnung   → STEUER_ROOT/[Jahr]/Eingangsrechnungen/[Lieferant]/[Dateiname]
Ausgangsrechnung   → STEUER_ROOT/[Jahr]/Ausgangsrechnungen/[Dateiname]
Steuerbescheid     → STEUER_ROOT/[Jahr]/Steuerbescheide/[Dateiname]
Unklar             → Datei im Dump belassen, in Unklar-Tab eintragen
```

Bei Eingangsrechnungen: Falls Lieferanten-Ordner nicht existiert → erstellen.
Dateiname beibehalten, aber wenn es ein UUID-Name ist → umbenennen zu `[Lieferant]_[Datum]_[Betrag].pdf`

**Jahr bestimmen:**
- Bei CSVs: aus dem Inhalt (Buchungsdatum der Transaktionen)
- Bei KK-PDFs: aus dem Abrechnungszeitraum
- Bei Rechnungs-PDFs: aus dem Zahlungsdatum (Zufluss-Prinzip), Fallback Rechnungsdatum
- Bei Kontoauszügen: aus dem Dateinamen (enthält Datum)

### Schritt 5: In EÜR buchen

→ Siehe Abschnitt "EÜR-Buchungslogik"

### Schritt 6: Auch lose PDFs in Jahresordnern verarbeiten

**WICHTIG:** In jedem Jahresordner (`[Jahr]/`) liegen oft noch lose PDFs direkt auf der Ebene (nicht in Unterordnern). Diese genauso verarbeiten wie den Dump-Ordner:
- Lesen, klassifizieren, in den richtigen Unterordner verschieben, buchen.
- `.DS_Store`, `.xlsx`-Dateien und Ordner ignorieren.
- CSVs: prüfen, ob es Bank-Daten sind → verarbeiten wie in Modul 2

---

## Modul 2: Bankdaten verarbeiten

Wird automatisch von Modul 1 aufgerufen, wenn CSVs im Dump erkannt werden. Kann auch separat über Bank-CSVs in den `[Jahr]/Bank/`-Ordnern laufen.

### Datenquellen

Die Konten stehen in `config.json` unter `konten`. Jedes Konto hat eine `art`:

**`geschaeftlich` (Geschäftskonto):**
- Alle Geschäftsbuchungen sind relevant.
- Interne Umbuchungen (siehe Grundprinzipien) ignorieren.

**`privat_gemischt` (Privatkonto mit geschäftlichen Anteilen):**
- **ACHTUNG:** Das ist ein PRIVATKONTO. Nur geschäftsrelevante Buchungen extrahieren!
- Sind mehrere Kontonummern in `kontonummern` eingetragen, alle prüfen.

### CSV-Formate

Jedes Bank-CSV-Format ist als austauschbare Parser-Beschreibung in `config.json` unter `csv_formate` hinterlegt. Eine Beschreibung enthält:

| Feld | Bedeutung |
|------|-----------|
| `erkennung_header_enthaelt` | Spaltenköpfe, an denen das Format erkannt wird |
| `trennzeichen` | z. B. `,` oder `;` |
| `datumsformat` | z. B. `YYYY-MM-DD` oder `DD.MM.YYYY` |
| `dezimaltrenner` | `.` oder `,` |
| `spalten` | Zuordnung der Felder `buchungsdatum`, `partner`, `verwendungszweck`, `betrag` (optional `kontoname`) zu den Spaltenköpfen der Datei |

Mitgelieferte Beispiele (erfunden, an die eigene Bank anpassen):
- `format_a`: Komma-getrennt, ISO-Datum, Punkt als Dezimaltrenner, englische Spaltenköpfe. Beispielkopf: `Date,Counterparty,Reference,Amount,Account`
- `format_b`: Semikolon-getrennt, deutsche Spaltenköpfe, Datum `DD.MM.YYYY`, Betrag mit Komma. Beispielkopf: `Buchungstag;Name Zahlungsbeteiligter;Verwendungszweck;Betrag`

**Weiteres Format hinzufügen:**
1. Kopfzeile und zwei bis drei Beispielzeilen der CSV ansehen (Werte nicht weitergeben oder speichern).
2. Trennzeichen, Datumsformat, Dezimaltrenner und Spaltenzuordnung als neuen Eintrag in `csv_formate` beschreiben.
3. Das Format dem Konto in `konten` über `csv_format` zuordnen.
4. Passt keine Beschreibung zur Datei: nicht raten, den Nutzer fragen.

Übliche Exportdateien einer Bank können auch alle Jahre in einer Datei enthalten (Gesamtexport).

### Verarbeitungslogik

1. CSV einlesen und nach Datum sortieren (→ Zufluss-Prinzip: Buchungsdatum = Zahlungsdatum)
2. Für jede Buchung prüfen:
   - Ist der Partner ein bekannter Lieferant/Kunde? → `lieferanten.json` abgleichen
   - Ist die Buchung bereits in der EÜR? → Duplikatsprüfung (Datum + Betrag ±0,02 EUR + Partner)
   - Bei Übereinstimmung: Betrag prüfen, bei Abweichung → Unklar-Tab melden
   - Konto `geschaeftlich`: Alle Geschäftsbuchungen relevant
   - Konto `privat_gemischt`: Nur buchen, wenn Partner in der Lieferanten-DB steht ODER manuell bestätigt

3. **Konto `geschaeftlich`: Ignorieren:**
   - Interne Umbuchungen laut `ignoriere_umbuchungen` des Kontos (z. B. Zeilen, deren Kontoname einem Unterkonto entspricht) → ignorieren

4. **Konto `privat_gemischt`: Bekannte private Buchungen ignorieren:**
   - Vollständige Liste: siehe `privat_konto` in `lieferanten.json`
   - Daumenregel: Supermärkte, Wertpapierdepots, Privatversicherungen → privat

5. **Konto `privat_gemischt`: Geschäftsrelevante Buchungen:**
   - Alle Partner, die in `lieferanten.json` stehen → Betriebsausgaben
   - Zahlungsdienst-Buchungen (z. B. Online-Bezahldienste) → prüfen, ob der Empfänger im Verwendungszweck erkennbar ist
   - Unklare Zahlungsdienst-Buchungen → Unklar-Tab

6. Neue Buchungen → in EÜR eintragen oder Unklar-Tab

### Einnahmen erkennen

Eingehende Zahlungen auf dem Geschäftskonto mit einer Referenz im Nummernformat aus `ausgangsrechnungen.nummernformat` in `config.json` (im Beispiel `RE` plus Datum `YYYYMMDD`) → als Einnahme buchen. Bekannte Kunden: siehe `einnahmen` in `lieferanten.json`.

### Nach Verarbeitung

CSV-Datei aus dem Dump in den passenden `[Jahr]/Bank/`-Ordner verschieben.
Bei Gesamtexport (alle Jahre in einer CSV): Datei im `STEUER_ROOT` belassen, nicht verschieben.

---

## Modul 3: Kreditkarten-PDFs per OCR

Wird automatisch von Modul 1 aufgerufen, wenn KK-Abrechnungen im Dump erkannt werden. Kann auch separat über PDFs in `[Jahr]/Kreditkarte/` laufen.

### Erkennung

KK-Abrechnungen erkennen an:
- Dateiname enthält einen Wert aus `kreditkarte.abrechnung_dateiname_enthaelt` in `config.json`
- Inhalt zeigt einen Wert aus `kreditkarte.abrechnung_inhalt_enthaelt`
- Tabellarisches Format mit Umsätzen

### Verarbeitungslogik

1. PDF über ein Dateisystem-Werkzeug laden (z. B. `Filesystem:read_media_file`) → Claude versucht direkt zu lesen
2. Falls nicht lesbar: ein Bild-/OCR-Werkzeug nutzen (z. B. `nano-banana-pro:describe_image`)
3. Aus jeder Abrechnung extrahieren:
   - Abrechnungszeitraum → bestimmt das Jahr
   - Einzelne Umsätze: Datum, Händler, Betrag, Währung
4. Jeden Umsatz klassifizieren:
   - Bekannter Lieferant → Betriebsausgabe
   - Lieferant mit Kategorie `KFZ` (KFZ-Leasing, siehe Grundprinzipien) → KFZ, Anteil wie in den Grundprinzipien beschrieben
   - Unbekannt → Unklar-Tab
   - Supermärkte, Restaurants, Tankstellen → privat, ignorieren
5. **Deduplizierung:** Prüfen, ob KK-Einzelposten bereits über Bank-CSV erfasst sind
   - Die monatliche KK-Gesamtabbuchung auf dem Bankkonto → NICHT als separate Ausgabe buchen
   - Nur die Einzelposten aus der KK-Abrechnung buchen, die Betriebsausgaben sind

### Nach Verarbeitung

KK-PDF aus dem Dump in den passenden `[Jahr]/Kreditkarte/`-Ordner verschieben.

---

## Modul 4: Gmail nach Rechnungen scannen

### Vorgehen

Nutze die Gmail-Suche (in der Vorlage `Gmail:gmail_search_messages`; Tool vorher laden via `tool_search`, der Name hängt vom verwendeten Connector ab).

Die Absender-Domain eines Lieferanten steht nicht in `lieferanten.json`. Sie aus früheren Rechnungs-Mails ermitteln (Suche nach dem Lieferantennamen) oder den Nutzer fragen, nicht raten.

```
Suchbegriffe pro Lieferant (<DOMAIN_LIEFERANT> durch die Absender-Domain ersetzen):
- from:<DOMAIN_LIEFERANT> subject:rechnung after:[YYYY]/01/01 before:[YYYY]/12/31
- from:<DOMAIN_LIEFERANT> invoice after:[YYYY]/01/01 before:[YYYY]/12/31
- from:<DOMAIN_LIEFERANT> subject:invoice OR rechnung after:[YYYY]/01/01 before:[YYYY]/12/31
- from:<DOMAIN_LIEFERANT> invoice OR receipt after:[YYYY]/01/01 before:[YYYY]/12/31
```

Für jede gefundene Rechnung:
1. Prüfen, ob die Rechnung bereits im Eingangsrechnungen-Ordner liegt
2. Falls nicht → melden, was fehlt
3. **WICHTIG:** PDF-Downloads NICHT automatisch machen → nur auflisten, was fehlt
4. Nutzer anweisen: "Lade diese Rechnungen herunter und lege sie in `STEUER_ROOT/Dump/` ab, dann verarbeite ich sie"

---

## Modul 5: EÜR Status-Report

Erstelle einen Überblick über den aktuellen Stand:

```
EÜR Status [Jahr]
=================
Einnahmen (Brutto):  XX.XXX,XX EUR
Ausgaben (Brutto):   XX.XXX,XX EUR
Gewinn (Brutto):     XX.XXX,XX EUR
Gewinn (Netto):      XX.XXX,XX EUR

Buchungen gesamt:    XXX
Davon unklar:        XX  ← Diese noch klären!

Fehlende Monate (Vergleich Bank zu EÜR):
- Kreditkarte: Apr, Jun, Aug
- Geschäftskonto: komplett [OK]
- Privatkonto: komplett [OK]

Bekannte Lieferanten ohne Buchung dieses Jahr:
- [Lieferant]: hatte letztes Jahr X Buchungen, dieses Jahr 0
```

---

## EÜR-Buchungslogik

### Dateistandort

```
STEUER_ROOT/[Jahr]/EÜR [Jahr].xlsx
```

Falls die Datei nicht existiert → automatisch neu anlegen (siehe `references/euer-format.md`).

**Vor dem Bearbeiten:** Lies `references/euer-format.md` für die exakte Spaltenstruktur.

### Sheets

| Sheet | Inhalt |
|-------|--------|
| **Gesamt** | Alle Buchungen des Jahres, chronologisch |
| **Q1 bis Q4** | Quartalsansichten (falls vorhanden) |
| **Unklar** | Buchungen, die noch geklärt werden müssen |

### Unklar-Tab Struktur

Der Unklar-Tab hat dieselben Spalten wie der Gesamt-Tab (A bis J), plus:

| Spalte | Header | Beschreibung |
|--------|--------|--------------|
| K | Quelle | `Bank-<id>` (id des Kontos aus `config.json`), `KK`, `Dump`, `Gmail` |
| L | Grund | `Unbekannter Lieferant`, `Empfänger nicht Rechnungsempfänger`, `Zahlungsdienst ohne Empfänger`, `Betrag unleserlich`, `Datum unleserlich`, `Privat oder geschäftlich unklar`, `Betragsdifferenz`, `Neuer Lieferant` |

### Buchungsregeln

1. **Einnahmen** (Ausgangsrechnungen):
   - Brutto = Rechnungsbetrag
   - Netto = ROUND(Brutto / 1.19, 2)
   - USt = Brutto - Netto

2. **Ausgaben (Deutschland, 19% USt):**
   - Brutto = Rechnungsbetrag
   - Netto = ROUND(Brutto / 1.19, 2)
   - USt = Brutto - Netto

3. **Ausgaben (Deutschland, 7% USt):**
   - Netto = ROUND(Brutto / 1.07, 2)
   - USt = Brutto - Netto

4. **Ausgaben (Ausland, keine USt):**
   - Brutto = Rechnungsbetrag
   - Netto = Brutto (identisch!)
   - USt = 0,00 EUR

5. **Duplikatsprüfung:** Vor jeder Buchung prüfen, ob ein Eintrag mit gleichem Datum + ähnlichem Betrag (±0,02 EUR) + gleichem Partner bereits existiert. Bei Abweichung → Unklar-Tab mit "Betragsdifferenz".

6. **Sortierung:** Neue Buchungen chronologisch einsortieren, nicht ans Ende anhängen.

7. **SUM-Formeln:** Nach dem Einfügen neuer Zeilen die SUM-Formeln am Ende aktualisieren!

### EÜR-Datei bearbeiten (Technisch)

**Lesen:** Die XLSX-Datei über ein Dateisystem-Werkzeug lesen (z. B. `Filesystem:read_media_file`, Base64) → im Container decodieren → mit openpyxl verarbeiten → Ergebnis als Base64 → über ein Schreib-Werkzeug (z. B. `Filesystem:write_file`) zurückschreiben.

**Alternativ:** Direkt über Dateisystem-Werkzeuge lesen und schreiben, wenn die Umgebung das erlaubt.

**Backup:** Vor JEDER Änderung: `EÜR [Jahr]_backup_[YYYYMMDD].xlsx` im selben Ordner.

---

## Lieferanten-Datenbank

Lies `lieferanten.json` (eigene Liste im Konfigurationsordner) für die vollständige Liste. Enthält:
- Bekannte Ausgaben-Lieferanten mit Bank-Namen-Varianten, Kategorie, USt-Status, Land
- Bekannte Einnahmen-Kunden
- Ignorier-Liste (Rechnungen, die auf eine andere Firma laufen)
- Privat-Filter (`privat_konto`) für das gemischt genutzte Privatkonto

**Wenn ein neuer Lieferant erkannt wird:**
1. Frage den Nutzer: "Neuer Lieferant: [Name]. Betriebsausgabe? Welche Kategorie?"
2. Wenn bestätigt: Lieferant in die JSON-Datei aufnehmen
3. Buchung durchführen

---

## Sicherheitsregeln

1. **Niemals Dateien endgültig löschen**, nur verschieben
2. **Vor dem Überschreiben der EÜR** immer eine Sicherungskopie anlegen
3. **Rechnungen von Absendern unter `ignorieren` ignorieren**, sie laufen auf eine andere Firma oder Person
4. **Rechnungen, die nicht auf den Rechnungsempfänger aus `config.json` ausgestellt sind** → Unklar-Tab
5. **Nie Beträge schätzen**, wenn nicht lesbar → Unklar-Tab
6. **Gmail-Attachments nicht automatisch downloaden**, nur melden, was fehlt
7. **Bei Unsicherheit lieber Unklar-Tab** als falsch buchen
8. **Unklar-Dateien im Dump belassen**, nur verarbeitete Dateien verschieben

---

## Scheduled Task (Monatlich)

Dieses Plugin unterstützt einen monatlichen Cowork Scheduled Task:

**Trigger:** Am 1. jeden Monats automatisch
**Ablauf:**
1. Prüfe, ob neue Dateien im Dump-Ordner liegen → falls ja, verarbeiten
2. Prüfe, ob lose PDFs in Jahresordnern liegen → falls ja, verarbeiten
3. Erstelle Status-Report für das aktuelle Jahr
4. Melde: "X neue Buchungen verarbeitet, Y unklar. Dump-Ordner hat noch Z unverarbeitete Dateien."

**Einrichten:** In Cowork `/schedule` verwenden und diesen Task konfigurieren.

---

## Ablauf bei "Steuern aktualisieren" / "EÜR [Jahr] updaten"

1. Frage: "Welches Jahr?" (oder alle)
2. Falls EÜR-Datei für das Jahr nicht existiert → automatisch anlegen
3. Dump-Ordner verarbeiten (Modul 1): erkennt automatisch CSVs, KK-PDFs, Rechnungen
4. Lose PDFs in Jahresordnern verarbeiten (Modul 1, Schritt 6)
5. Falls noch Bank-CSVs in `[Jahr]/Bank/` liegen, die nicht verarbeitet wurden → Modul 2
6. Falls noch KK-PDFs in `[Jahr]/Kreditkarte/` liegen, die nicht verarbeitet wurden → Modul 3
7. EÜR aktualisieren und speichern
8. Status-Report ausgeben (Modul 5)
9. Optional: Gmail scannen (Modul 4), nur auf Anfrage, da zeitintensiv

Am Ende immer den Unklar-Tab präsentieren und fragen: "Diese X Buchungen sind noch unklar. Sollen wir die jetzt durchgehen?"
