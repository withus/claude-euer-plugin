---
name: kreditkarte-auslesen
description: >
  Liest Kreditkartenabrechnungen per OCR aus und bucht Betriebsausgaben in die EÜR.
  IMMER verwenden bei: "Kreditkarte auslesen", "KK verarbeiten", "Kreditkartenabrechnung",
  "KK-Abrechnungen buchen", "Kartenabrechnung auslesen".
---

# Kreditkarte auslesen

Lese Kreditkartenabrechnungen per OCR aus und buche Betriebsausgaben:

1. Aktiviere den `steuer-euer` Skill
2. Frage nach dem Jahr (oder nutze $ARGUMENTS)
3. Lese alle PDFs aus `<STEUER_ROOT>/[Jahr]/Kreditkarte/`
4. Extrahiere Umsätze per OCR (die Abrechnungen sind gescannte Bilder)
5. Klassifiziere jeden Umsatz: Betrieb oder privat
6. Buche Betriebsausgaben in die EÜR, unklare in den Unklar-Tab
7. Prüfe auf Deduplizierung mit bestehenden Bank-Buchungen

$ARGUMENTS enthält optional das Jahr, z.B. "2025".
