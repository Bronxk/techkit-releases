# TechKit – Downloads

Öffentliche Auslieferung von **TechKit**, dem portablen Techniker-Werkzeug für Windows (innos).
Dieses Repository enthält nur fertige Programme und das Startskript, keinen Quellcode.

## Starten

In Windows PowerShell:

```powershell
irm https://bronxk.github.io/techkit-releases/install.ps1 | iex
```

Das Skript lädt die passende `techkit.exe` (x64 oder ARM64) nach `%LOCALAPPDATA%\TechKit\bin`,
prüft Größe und SHA-256 und startet sie. Danach aktualisiert sich TechKit selbst.

Manuell: `techkit.exe` aus den [Releases](https://github.com/Bronxk/techkit-releases/releases) laden.
Die Prüfsummen stehen in `checksums.txt` an jedem Release.

## Signaturen

- Update-Listen (`stable/latest.json`, `beta/latest.json`) sind mit ed25519 signiert; TechKit prüft jedes Update selbst.
- Die EXE ist **noch nicht** per Authenticode signiert. Windows SmartScreen kann deshalb warnen.

## Nutzung

Nutzung nur nach Absprache.
