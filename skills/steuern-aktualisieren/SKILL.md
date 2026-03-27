---
name: steuern-aktualisieren
description: >
  Führt den kompletten Steuer-Update-Workflow aus: Dump verarbeiten, Bankdaten einlesen,
  Kreditkarten-OCR, EÜR aktualisieren und Status-Report erstellen.
  IMMER verwenden bei: "Steuern aktualisieren", "EÜR updaten", "Buchhaltung machen",
  "alles auf den neuesten Stand bringen", "komplettes Steuer-Update".
---

# Steuern aktualisieren

Führe den kompletten Steuer-Update-Workflow aus:

1. Frage den Nutzer, welches Jahr (oder alle vorhandenen Jahre)
2. Aktiviere den `steuer-euer` Skill und folge dessen Anweisungen (inklusive Abschnitt "Konfiguration")
3. Verarbeite den Dump-Ordner (Modul 1)
4. Lese Bankdaten ein (Modul 2)
5. Lese Kreditkarten-PDFs per OCR aus (Modul 3)
6. Aktualisiere die EÜR-Dateien
7. Erstelle einen Status-Report (Modul 5)
8. Präsentiere den Unklar-Tab und frage, ob diese Buchungen geklärt werden sollen

$ARGUMENTS enthält optional das Jahr, z.B. "2024" oder "alle".
