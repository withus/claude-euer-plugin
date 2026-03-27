---
name: gmail-scannen
description: >
  Durchsucht Gmail nach fehlenden Rechnungen und vergleicht mit dem Eingangsrechnungen-Ordner.
  Listet auf, was fehlt, lädt nichts automatisch herunter.
  IMMER verwenden bei: "Gmail scannen", "fehlende Rechnungen", "Gmail nach Rechnungen durchsuchen",
  "welche Rechnungen fehlen", "Gmail Rechnungen prüfen".
---

# Gmail Rechnungen scannen

Durchsuche Gmail nach fehlenden Rechnungen:

1. Aktiviere den `steuer-euer` Skill
2. Frage nach dem Jahr (oder nutze $ARGUMENTS)
3. Scanne Gmail für jeden bekannten Lieferanten aus der Lieferantenliste (`lieferanten.json`, siehe Abschnitt "Konfiguration" im Skill `steuer-euer`)
4. Vergleiche gefundene Rechnungen mit den Dateien im Eingangsrechnungen-Ordner
5. Liste alle fehlenden Rechnungen auf
6. Weise den Nutzer an, fehlende PDFs in `<STEUER_ROOT>/Dump/` abzulegen

Keine automatischen Downloads, nur auflisten, was fehlt.

$ARGUMENTS enthält optional das Jahr, z.B. "2024".
