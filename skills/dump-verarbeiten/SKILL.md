---
name: dump-verarbeiten
description: >
  Verarbeite ALLE Dateien im zentralen Dump-Ordner (<STEUER_ROOT>/Dump/).
  Erkennt automatisch PDFs, Bank-CSVs, Kreditkartenabrechnungen und sortiert sie ein.
  IMMER verwenden bei: "Dump verarbeiten", "Belege sortieren", "Rechnungen einsortieren",
  "Dump-Ordner aufräumen", "neue Belege verarbeiten".
---

# Dump verarbeiten

Verarbeite ALLE Dateien im zentralen Dump-Ordner (`<STEUER_ROOT>/Dump/`).

Der Dump-Ordner nimmt alles entgegen: PDFs, Bank-CSVs, Kreditkartenabrechnungen. Claude erkennt automatisch den Dateityp:

- **CSV-Dateien** → Bank-Daten (Format laut `csv_formate` in `config.json`), Transaktionen verarbeiten und buchen
- **KK-Abrechnungen (PDF)** → Per OCR auslesen, Betriebsausgaben extrahieren
- **Rechnungs-PDFs** → Lesen, klassifizieren, in Lieferanten-Ordner einsortieren, buchen
- **Kontoauszüge (PDF)** → In Bank-Ordner verschieben
- **Steuerbescheide** → In Steuerbescheide-Ordner verschieben

## Ablauf

1. Aktiviere den `steuer-euer` Skill
2. Lese den Dump-Ordner aus, klassifiziere jede Datei nach Typ
3. Verarbeite CSVs → extrahiere Buchungen, buche in EÜR
4. Verarbeite KK-PDFs → OCR, extrahiere Betriebsausgaben
5. Verarbeite Rechnungs-PDFs → klassifiziere, verschiebe, buche
6. Verschiebe verarbeitete Dateien in die richtigen Ordner
7. Zeige Zusammenfassung: X verarbeitet, Y gebucht, Z unklar

Falls der Dump-Ordner leer ist, prüfe auch lose PDFs/CSVs in den Jahresordnern (alle vorhandenen Jahre).

$ARGUMENTS enthält optional das Jahr, z.B. "2024". Dann nur Dateien für dieses Jahr verarbeiten.
