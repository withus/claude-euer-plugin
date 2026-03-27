---
name: euer-status
description: >
  Erstellt einen Überblick über den aktuellen Stand der Einnahmenüberschussrechnung.
  Zeigt Einnahmen, Ausgaben, Gewinn, unklare Buchungen und Vollständigkeitsprüfung.
  IMMER verwenden bei: "EÜR Status", "Was fehlt noch?", "Steuerstand", "Übersicht EÜR",
  "wie sieht die EÜR aus", "Buchungsstand".
---

# EÜR Status

Erstelle einen Überblick über den aktuellen Stand der Einnahmenüberschussrechnung:

1. Aktiviere den `steuer-euer` Skill und lies die EÜR-Datei(en)
2. Zeige pro Jahr: Einnahmen, Ausgaben, Gewinn (Brutto + Netto)
3. Zeige Anzahl Buchungen und davon unklare
4. Prüfe Vollständigkeit: Fehlen Monate bei Bank, Kreditkarte oder Lieferanten?
5. Liste Lieferanten auf, die letztes Jahr Buchungen hatten, aber dieses Jahr nicht

$ARGUMENTS enthält optional das Jahr, z.B. "2025". Ohne Argument: alle Jahre zeigen.
