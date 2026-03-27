# claude-euer-plugin

Claude Code plugin for German sole proprietors who keep their books as a cash-basis profit calculation (EÜR, Einnahmenüberschussrechnung). It sorts receipts from one drop folder, reads bank CSV exports and scanned credit card statements, lists invoices that are missing from Gmail and updates the yearly Excel file. The documentation is in German because the workflow follows German tax practice.

Dieses Plugin ist keine Steuerberatung. Lesen Sie vor der Nutzung den Abschnitt "Grenzen".

## Was das Plugin macht

Das Plugin `steuer-euer` beschreibt einen Ablauf, mit dem Claude die Belege eines Selbstständigen in eine jährliche EÜR-Excel-Datei bucht.

Alle neuen Dateien kommen zuerst in einen einzigen Dump-Ordner. Claude erkennt, ob eine Datei eine Rechnung, eine Bank-CSV, eine Kreditkartenabrechnung, ein Kontoauszug oder ein Steuerbescheid ist, sortiert sie in die Jahresordner ein und bucht sie nach dem Zufluss-/Abflussprinzip. Maßgeblich ist das Zahlungsdatum, nicht das Rechnungsdatum. Kategorien übernimmt Claude aus den bestehenden EÜR-Dateien der Vorjahre. Was sich nicht sicher zuordnen lässt, landet im Tab "Unklar", statt geschätzt zu werden.

Das Repository enthält keine echten Daten. Lieferantenliste und Konten sind erfundene Beispiele, die beiden CSV-Formate sind an reale Bank-Exportformate angelehnt. Ersetzen Sie alles durch eigene Werte.

## Die sechs Skills

Claude Code stellt die Komponenten eines Plugins unter dem Plugin-Namen bereit, hier also zum Beispiel `steuer-euer:dump-verarbeiten`.

| Name | Auslöser (Beispiele) | Was er tut |
|------|----------------------|------------|
| `steuer-euer` | "Steuern", "EÜR", "Belege verarbeiten", "Bankdaten einlesen" | Grundskill mit allen Regeln: Zufluss-/Abflussprinzip, Kategorien, Duplikatsprüfung, Aufbau der EÜR-Datei und die fünf Module (Dump, Bankdaten, Kreditkarte, Gmail, Status-Report). Die anderen Skills aktivieren ihn. |
| `steuern-aktualisieren` | "Steuern aktualisieren", "EÜR updaten", "Buchhaltung machen" | Gesamtlauf: fragt das Jahr ab, verarbeitet Dump, Bankdaten und Kreditkarten, aktualisiert die EÜR, erstellt den Status-Report und stellt den Unklar-Tab zur Klärung vor. |
| `euer-status` | "EÜR Status", "Was fehlt noch?", "Buchungsstand" | Zeigt je Jahr Einnahmen, Ausgaben und Gewinn (brutto und netto), die Zahl der Buchungen und der unklaren Fälle, fehlende Monate bei Bank und Kreditkarte sowie Lieferanten ohne Buchung im laufenden Jahr. |
| `dump-verarbeiten` | "Dump verarbeiten", "Belege sortieren", "neue Belege verarbeiten" | Liest alle Dateien im Dump-Ordner, erkennt den Typ (Bank-CSV, Kreditkartenabrechnung, Rechnung, Kontoauszug, Steuerbescheid), bucht und verschiebt sie in die Jahresordner. Unklare Dateien bleiben im Dump. |
| `kreditkarte-auslesen` | "Kreditkarte auslesen", "KK verarbeiten", "Kartenabrechnung auslesen" | Liest gescannte Kreditkartenabrechnungen per OCR, trennt Betriebsausgaben von privaten Umsätzen und prüft auf Doppelbuchungen mit den Bank-Buchungen. |
| `gmail-scannen` | "Gmail scannen", "fehlende Rechnungen", "welche Rechnungen fehlen" | Durchsucht Gmail je bekanntem Lieferanten, vergleicht mit dem Eingangsrechnungen-Ordner und listet fehlende Rechnungen auf. Es werden keine Anhänge heruntergeladen. |

Bank-CSVs liest kein eigener Skill ein. Das erledigt Modul 2 des Skills `steuer-euer`, aufgerufen über `dump-verarbeiten` oder `steuern-aktualisieren`.

## Benötigte Werkzeuge

Die Skills benennen diese Werkzeuge (Fundstellen in `skills/steuer-euer/SKILL.md`):

| Zweck | Name in den Skills |
|-------|--------------------|
| PDFs und Excel-Dateien lesen und schreiben | `Filesystem:read_media_file`, `Filesystem:write_file` |
| Gescannte PDFs per OCR beschreiben | `nano-banana-pro:describe_image` |
| Gmail durchsuchen | `Gmail:gmail_search_messages` (wird per `tool_search` nachgeladen) |
| EÜR-Excel bearbeiten | Python mit `openpyxl` |

Die Namen stammen aus der Umgebung, in der der Ablauf entstanden ist. Die Skills nennen sie nur als Beispiele ("z. B."). Ein gleichwertiges Werkzeug in Ihrer Umgebung sollte genügen, das ist aber nicht getestet. Das Plugin bringt keine eigenen Skripte oder MCP-Server mit.

## Einrichtung

1. **Plugin laden.** Das Verzeichnis folgt dem Standard-Layout aus der Plugin-Dokumentation: Manifest in `.claude-plugin/plugin.json`, Skills unter `skills/<name>/SKILL.md`. Laut Dokumentation lädt `claude --plugin-dir <pfad>` ein Plugin aus einem Verzeichnis für die aktuelle Sitzung:

   ```
   git clone https://github.com/withus/claude-euer-plugin.git
   claude --plugin-dir ./claude-euer-plugin
   ```

   Zur dauerhaften Installation siehe die offizielle Dokumentation: https://code.claude.com/docs/en/plugins/loading.md und https://code.claude.com/docs/en/plugins-reference. Das Manifest lässt sich mit `claude plugin validate ./claude-euer-plugin` prüfen.

2. **Konfiguration anlegen.** Kopieren Sie `config.example.json` als `config.json` in den Datenordner des Plugins und ersetzen Sie die Platzhalter in spitzen Klammern. Laut Plugin-Dokumentation zeigt `${CLAUDE_PLUGIN_DATA}` auf `~/.claude/plugins/data/<id>/` und bleibt bei Plugin-Updates erhalten. Beim Deinstallieren des Plugins löscht Claude Code diesen Ordner laut Dokumentation standardmäßig; sichern Sie `config.json` und Ihre `lieferanten.json` vorher. Fehlt `config.json`, legt der Skill `steuer-euer` sie beim ersten Lauf aus der Vorlage an und fragt die Werte ab.

   | Feld | Bedeutung |
   |------|-----------|
   | `steuer_root` | Basisordner aller Steuerunterlagen (`<STEUER_ROOT>`). Darunter liegen `Dump/` und je Jahr ein Ordner `[Jahr]/`. |
   | `unternehmer.rechnungsempfaenger` | Name, auf den Eingangsrechnungen ausgestellt sein müssen. Rechnungen an andere Empfänger landen im Tab "Unklar". |
   | `ausgangsrechnungen.nummernformat` | Nummernformat der eigenen Rechnungen (Beispiel `RE-YYYYMMDDNN`). |
   | `konten` | Je Konto `id`, `art` (`geschaeftlich` oder `privat_gemischt`), `csv_format` und optional `kontonummern`, `csv_dateiname_enthaelt`, `kontoauszug_dateiname_enthaelt`, `ignoriere_umbuchungen`. |
   | `csv_formate` | Parser-Beschreibung je CSV-Format (siehe Schritt 4). |
   | `kreditkarte` | Erkennung von Kreditkartenabrechnungen über Dateiname und Inhalt. |

3. **Eigene Lieferantenliste anlegen.** `skills/steuer-euer/references/lieferanten.json` ist die Vorlage mit fünf erfundenen Lieferanten. Legen Sie Ihre Liste als `lieferanten.json` im Datenordner an. Beim ersten Lauf erfasst der Skill vorhandene Kategorien aus Ihren EÜR-Dateien und ergänzt neue Lieferanten nach Rückfrage.

   | Abschnitt | Felder |
   |-----------|--------|
   | `ausgaben` | `name`, `bank_names` (Schreibweisen auf Kontoauszug und Kreditkarte), `kategorie`, `ust` (`DE-19%`, `DE-7%` oder `Ausland-0%`), `land`, `notizen` |
   | `einnahmen` | `name`, `bank_names`, `notizen` |
   | `ignorieren` | `name`, `bank_names`, `grund` (Rechnungen, die auf eine andere Firma laufen) |
   | `privat_konto` | Suchbegriffe für private Buchungen auf dem gemischt genutzten Privatkonto |

4. **CSV-Formate beschreiben.** Jede Bank exportiert anders. In `csv_formate` beschreiben Sie je Format Trennzeichen, Datumsformat, Dezimaltrenner, die Spaltenköpfe zur Erkennung und die Zuordnung der Felder `buchungsdatum`, `partner`, `verwendungszweck` und `betrag`. Mitgeliefert sind zwei erfundene Beispiele (`format_a` mit Komma und englischen Köpfen, `format_b` mit Semikolon und deutschen Köpfen). Ein weiteres Format fügen Sie hinzu, indem Sie die Kopfzeile der CSV beschreiben und das Format in `konten` über `csv_format` zuordnen. Passt kein Format zu einer Datei, rät der Skill nicht, sondern lässt die Datei im Dump liegen.

5. **Ordnerstruktur anlegen.** Der Skill legt einen fehlenden Dump-Ordner und fehlende Lieferanten-Ordner selbst an. Zielbild:

   ```
   <STEUER_ROOT>/
     Dump/
     [Jahr]/
       EÜR [Jahr].xlsx
       Bank/
       Kreditkarte/
       Eingangsrechnungen/<Lieferant>/
       Ausgangsrechnungen/
       Steuerbescheide/
   ```

## Ablauf

Der Gesamtlauf `steuern-aktualisieren` arbeitet in dieser Reihenfolge:

1. Jahr abfragen (oder alle vorhandenen Jahre).
2. Fehlt die EÜR-Datei des Jahres, wird sie mit den Tabellenblättern "Gesamt" und "Unklar" angelegt.
3. Dump-Ordner verarbeiten: CSVs, Kreditkarten-PDFs und Rechnungen werden erkannt, gebucht und einsortiert.
4. Lose PDFs in den Jahresordnern genauso verarbeiten.
5. Noch nicht verarbeitete Bank-CSVs in `[Jahr]/Bank/` einlesen.
6. Noch nicht verarbeitete Kreditkarten-PDFs in `[Jahr]/Kreditkarte/` auslesen.
7. EÜR aktualisieren und speichern. Vor jeder Änderung entsteht eine Sicherungskopie `EÜR [Jahr]_backup_[YYYYMMDD].xlsx` im selben Ordner.
8. Status-Report ausgeben.
9. Den Tab "Unklar" vorstellen und fragen, ob die offenen Buchungen jetzt geklärt werden sollen.

Die Gmail-Suche gehört nicht zum Standardlauf. Sie läuft nur auf Anfrage über `gmail-scannen`.

Grundregeln des Ablaufs, wie sie in den Skills stehen:

- Gebucht wird nach Zahlungsdatum (Zufluss-/Abflussprinzip).
- Bestehende Buchungen werden nicht doppelt angelegt. Bei abweichendem Betrag (Toleranz 0,02 EUR) wandert der Fall in den Tab "Unklar".
- Interne Umbuchungen zwischen eigenen Konten werden ignoriert.
- Dateien werden nie gelöscht, nur verschoben.
- Beträge werden nie geschätzt. Ist ein Betrag oder Datum nicht lesbar, kommt die Buchung in den Tab "Unklar".
- Gmail-Anhänge werden nicht automatisch heruntergeladen.

## Grenzen

- **Keine Steuerberatung.** Das Plugin ist ein Werkzeug zum Sortieren und Buchen, kein Ersatz für eine steuerliche Beratung. Sie bleiben für Ihre Buchführung und Steuererklärung selbst verantwortlich.
- **Steuerwerte müssen geprüft werden.** Die Buchungsregeln enthalten feste Werte: Umsatzsteuersätze von 19 % und 7 %, Netto-Berechnung per Division durch 1,19 beziehungsweise 1,07, Ausland ohne Umsatzsteuer und ohne Vorsteuerabzug, Reverse-Charge ohne gesonderte Verfolgung, 19 % Umsatzsteuer auf alle Ausgangsrechnungen, 100 % Betriebsausgabe beim Beispiel KFZ-Leasing (nur bei ausschließlich betrieblicher Nutzung) und eine Toleranz von 0,02 EUR bei der Duplikatsprüfung. Im Text steht kein Steuerwert mit Jahreszahl. Ob die Werte für Ihr Jahr und Ihre Situation stimmen, etwa bei Kleinunternehmerregelung, Privatanteilen oder Auslandsleistungen, müssen Sie selbst oder mit Ihrer Steuerberatung klären. Ändern Sie die Regeln in den `SKILL.md`-Dateien, wenn sie nicht passen.
- **Beispieldaten.** Lieferanten und Konten in diesem Repository sind erfunden, die CSV-Formate sind an reale Exportformate angelehnt. Die Beispielformate wurden nicht gegen aktuelle Bank-Exporte getestet.
- **OCR und Modellfehler.** Gescannte Belege können falsch gelesen werden. Prüfen Sie Beträge und Daten, besonders im Tab "Unklar" und nach dem ersten Lauf.
- **Datenschutz.** Beim Lesen von Belegen, Kontoumsätzen und Mails verarbeitet Claude diese Inhalte. Prüfen Sie vorab, ob das mit Ihren Datenschutzanforderungen vereinbar ist.
- **Monatlicher Scheduled Task.** Der Abschnitt dazu in `skills/steuer-euer/SKILL.md` stammt aus der Ursprungsumgebung und ist außerhalb davon nicht getestet.
- **Umgebung.** Die Werkzeugnamen und der Ablauf mit Excel-Dateien in einem lokalen Ordner hängen von Ihrer Umgebung ab.

## Haftungsausschluss

Die Software wird ohne Gewähr bereitgestellt (siehe `LICENSE`). Autor und Mitwirkende haften nicht für fehlerhafte Buchungen, falsche Steuerwerte oder daraus entstehende steuerliche Folgen.

## Lizenz

MIT, siehe `LICENSE`.
