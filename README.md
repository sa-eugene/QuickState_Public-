# Quickstate - Funktionsweise und Systemressourcen

## Übersicht

**Quickstate** ist eine Desktop-Anwendung zur Echtzeitüberwachung und Performance-Analyse von Systemressourcen. Die App kombiniert Electron (für den Desktop-Zugriff) mit React (für die Benutzeroberfläche) und bietet detaillierte Visualisierungen sowie Datenexport-Funktionalität.

---

## Architektur der Anwendung

### 1. Ebenen-Struktur

Die App besteht aus three Ebenen:

```
┌─────────────────────────────────────┐
│      React Frontend (UI-Layer)      │  ← Benutzeroberfläche mit Charts
│  (App.tsx, Components, Charts)      │     und Komponenten
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│    IPC Bridge (Preload-Script)      │  ← Sicherer Kommunikationskanal
│  (preload.js - ContextBridge)       │     zwischen Prozessen
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│   Electron Main Process (Backend)   │  ← Zugriff auf Betriebssystem
│  (main.js - Systemüberwachung)      │     und Systemdaten
└─────────────────────────────────────┘
```

### 2. Prozessmodell

- **Hauptprozess (Electron Main)**: Läuft im Hintergrund, sammelt Systemdaten, verwaltet Window-Fenster
- **Renderer-Prozess (React)**: Zeigt die UI an, reagiert auf Benutzerinteraktionen
- **IPC-Kommunikation**: Sichere bidirektionale Kommunikation über `contextBridge` und `ipcRenderer`

---

## Datenfluss: Wie die App funktioniert

### 1. Start-Prozess

1. **Fenster wird erstellt**: Electron öffnet das Anwendungsfenster
2. **UI wird geladen**: 
   - Im Dev-Modus: Lädt vom Vite-Development-Server (Port 3001)
   - Im Produktionsmodus: Lädt aus `renderer/index.html`
3. **React-App startet**: `App.tsx` wird initialisiert
4. **Electron API wird registriert**: `preload.js` verbindet Frontend mit Backend

### 2. Echtzeitüberwachung beim Starten

```
Electron Main:
├─ Initialisiert Systemüberwachung (systeminformation)
├─ Startet regelmäßige Datenerfassung
└─ Sendet Updates über IPC an renderer

React Frontend:
├─ Empfängt 'system-data-update' Events
├─ Aktualisiert lokalen State
└─ Rendert Charts und Metriken neu
```

### 3. Continuous Monitoring (kontinuierliche Überwachung)

**Update-Intervalle:**
- **FAST**: 3 Sekunden (Standard)
- **MEDIUM**: 20 Sekunden
- **SLOW**: 30 Sekunden
- **VERY_SLOW**: 120 Sekunden (2 Minuten)

Die Intervalle können von der Benutzeroberfläche aus angepasst werden.

---

## Systemressourcen: Was wird überwacht?

### 1. **CPU-Auslastung**
- Aktuelle CPU-Nutzung in %
- Anzahl der CPU-Kerne
- CPU-Temperatur
- CPU-Modellbezeichnung
- CPU-Taktfrequenz
- **Hintergrund-Tool**: `systeminformation` Modul liest CPU-Daten vom Betriebssystem

### 2. **RAM / Speicher**
- Verwendeter RAM in GB
- Gesamt verfügbarer RAM
- Verfügbarer (freier) RAM
- **Datenquelle**: Betriebssystem-API ruft via Node.js ab

### 3. **Festplatte / Speicherplatz**
- Festplatten-Auslastung pro Partition
- Verfügbarer Speicherplatz pro Laufwerk (C:, D:, etc.)
- Partitionsinformationen (Name, Dateisystem)
- **Methode**: File-System-APIs und Betriebssystem-Abfragen

### 4. **Netzwerk**
- Download-/Upload-Geschwindigkeit (für aktuelle Sitzung)
- Gesamt-Daten (seit App-Start)
- Netzwerkschnittstellen-Status
- Schnittstellentyp (Ethernet, WiFi, etc.)
- **Tool**: `systeminformation` überwacht Netzwerk-Adapter

### 5. **Prozesse (Running Programs)**
- Liste aller laufenden Prozesse
- CPU-Nutzung pro Prozess
- RAM-Nutzung pro Prozess
- Prozess-ID (PID)
- Prozess-Status
- **Datenquelle**: 
  - Windows: `tasklist` / `Get-Process` (PowerShell)
  - Linux/Mac: `ps` Kommando

### 6. **System-Information**
- Betriebssystem (Windows/Linux/macOS)
- Systemarchitektur (x64, ARM, etc.)
- Hostname (Computername)
- Systemlaufzeit (Uptime)
- **Quelle**: Node.js `os` Modul

---

## Hintergrund-Programme und Abhängigkeiten

### 1. **Node.js Module**

```
systeminformation (v5.27.11)
├─ Sammelt erweiterte Systeminfos
├─ CPU, Memory, Disk, Network
└─ Prozessliste und Temperaturen
```

### 2. **Windows-spezifische Tools**

Die App nutzt diese PowerShell-Kommandos im Hintergrund:

```powershell
# Prozessliste auslesen:
Get-Process | Select-Object ProcessName, Id, CPU, WorkingSet

# Netzwerk-Statistiken:
Get-NetAdapter

# Festplatte-Info:
Get-Volume | Select-Object DriveLetter, Size, SizeRemaining
```

**PowerShell-Pfade (automatisch erkannt):**
- `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe`
- `C:\Windows\Sysnative\WindowsPowerShell\v1.0\powershell.exe` (32-Bit-Kompatibilität)
- `pwsh.exe` (PowerShell Core, falls installiert)

### 3. **Externe Bibliotheken**

```json
Runtime Dependencies:
├─ React 18.3.1 (UI-Framework)
├─ Recharts 2.15.2 (Charts & Grafiken)
├─ Radix UI (UI-Komponenten)
├─ Tailwind CSS (Styling)
├─ Next-Themes (Dark/Light Mode)
├─ pidusage 4.0.1 (Prozess-Ressourcennutzung)
└─ Electron 38.4.0 (Desktop-Framework)
```

---

## Datenverarbeitung im Detail

### Schritt-für-Schritt: CPU-Daten sammeln und anzeigen

```
1. main.js → systeminformation.currentLoad()
   └─ Ruft CPU-Auslastung vom OS ab

2. main.js → ipcMain.handle('get-system-data')
   └─ Verarbeitet Anfrage vom Renderer

3. main.js → broadcastSystemData(mainWindow)
   └─ Sendet Daten an Frontend via IPC

4. preload.js → contextBridge.exposeInMainWorld()
   └─ Macht Daten für React zugänglich

5. App.tsx → electronAPI.onSystemDataUpdate(callback)
   └─ React empfängt Update

6. React State Update
   └─ setSystemData(newData)

7. CPUChart.tsx → Rendert den Chart neu
   └─ Benutzer sieht aktualisierte Grafik
```

### Daten-Caching

Die App cacht Daten, um die Systemlast zu reduzieren:

```javascript
dataCache = {
  disks: { data: [...], timestamp: 1234567890 },
  network: { data: {...}, timestamp: 1234567890 },
  processes: { data: [...], timestamp: 1234567890 },
  cpu: { data: {...}, timestamp: 1234567890 },
  // Nur zugleich re-fetch, wenn Cache älter als threshold
}
```

---

## Export- und Aufzeichnungs-Features

### 1. **Live-Aufzeichnung (Recording)**
- Speichert Systemmetriken in Archive
- Konfigurierbare Aufzeichnungsintervalle
- Automatische Cleanup nach 60 Tagen
- JSON-Format für Datenverarbeitung

### 2. **Export-Formate**
- **PDF**: Visualisiert Statistiken grafisch
- **JSON**: Rohe Daten zur weiteren Analyse
- **CSV**: Tabellendaten für Spreadsheets

### 3. **Archivverwaltung**
- Dateien werden mit Namen `quickstate-archive-{timestamp}.json` gespeichert
- Automatisches Cleanup nach konfigurierbaren Retentions-Tagen
- Benutzer kann Export-Verzeichnis wählen

---

## Performance-Optimierungen

### 1. **Throttling der Updates**
```javascript
UPDATE_INTERVALS = {
  FAST: 3000,      // 3 Sekunden
  MEDIUM: 20000,   // 20 Sekunden
  SLOW: 30000,     // 30 Sekunden
  VERY_SLOW: 120000 // 2 Minuten
}
```
Benutzer kann auswählen, wie oft die Daten aktualisiert werden sollen.

### 2. **Logging und Diagnose**
- `appLogService.ts`: Zentrale Logging-Verwaltung
- Logs werden in `userData/logs/quickstate.log` gespeichert
- Automatisches Log-Rotation/-Cleanup

### 3. **Error Handling**
- `ChartErrorBoundary.tsx`: Fängt Component-Fehler ab
- Verhindert UI-Crashes bei Datenprobleme
- Fallback-UI für fehlerhafte Charts

---

## UI-Komponenten und ihre Funktion

### Datenvisualisierung:
| Komponente | Visualisiert |
|-----------|-------------|
| `CPUChart.tsx` | CPU-Auslastung über Zeit (Linien-Chart) |
| `MemoryChart.tsx` | RAM-Nutzung über Zeit |
| `NetworkChart.tsx` | Download/Upload-Geschwindigkeit |
| `DiskUsage.tsx` | Festplatten-Partition-Aufteilung |
| `ProcessList.tsx` | Top-Prozesse nach CPU/Memory |
| `SystemInfo.tsx` | Statische System-Infos |

### UI-Framework:
- **Tabs**: Navigation zwischen verschiedenen Ansichten
- **Alerts**: Warnmeldungen bei Problemen
- **Badge**: Status-Anzeigen
- **Dark/Light Mode**: Theme-Umschaltung

---

## Verzeichnisstruktur und Datenfluss

```
Quickstate/
├─ src/
│  ├─ electron/
│  │  ├─ main.js ..................... Hauptprozess, Systemdaten-Erfassung
│  │  ├─ preload.js .................. Sicherer Bridge für IPC
│  │  ├─ systemMonitor.js ........... Systemüberwachungs-Logik
│  │  └─ utils.js ................... Hilfsfunktionen
│  ├─ components/
│  │  ├─ CPUChart.tsx ............... CPU-Visualisierung
│  │  ├─ MemoryChart.tsx ............ RAM-Visualisierung
│  │  ├─ NetworkChart.tsx ........... Netzwerk-Visualisierung
│  │  └─ ProcessList.tsx ............ Prozesse-Liste
│  ├─ services/
│  │  ├─ electronAPI.ts ............. TypeScript-Definitionen für IPC
│  │  ├─ electronExportService.ts ... Export-Logik
│  │  └─ appLogService.ts ........... Logging
│  └─ App.tsx ....................... Hauptkomponente
├─ renderer/
│  └─ index.html .................... HTML-Template
└─ build/
   └─ index.html .................... Produktions-Build
```

---

## Sicherheitsaspekte

### 1. **IPC-Sicherheit**
- `contextBridge` isoliert die Renderer-API
- Nur bestimmte, whitelisted Funktionen sind verfügbar
- Keine direkter Zugriff auf Node.js `require()`

### 2. **Web Security**
- `webSecurity: true` ist aktiviert
- CORS-Restrictions für Dev-Server
- Sichere Origins für Kommunikation

### 3. **Berechtigungen**
- App läuft mit aktuellen User-Berechtigungen
- Kann auf alle lokalen Dateien des Benutzers zugreifen
- Systemzugriff über OS-APIs (Read-Only Monitoring)

---

## Typischer Ablauf bei der Nutzung

### Szenario: Benutzer öffnet App und sieht CPU-Auslastung

```
1. App startet → Fenster wird erstellt
2. Electron main.js initiert SystemMonitoring
3. React App renderiert mit Platzhaltern
4. main.js sammelt CPU-Daten via systeminformation
5. main.js sendet 'system-data-update' event über IPC
6. preload.js → React Callback aktiviert
7. App.tsx State aktualisiert sich
8. CPUChart re-rendert mit neuen Daten
9. Benutzer sieht aktuelle CPU-Auslastung im Chart
10. Loop wiederholt sich alle 3 Sekunden (FAST-Mode)
```

---

## Fazit

Quickstate ist eine vollständig integrierte Desktop-Monitor-Anwendung, die:

✅ **Echtzeitdaten** vom Betriebssystem erfasst  
✅ **Interaktive Charts** für visuelle Analyse bietet  
✅ **Prozessdaten** der laufenden Programme anzeigt  
✅ **Archivdaten** für historische Analyse speichert  
✅ **Export-Funktionen** für weitere Verarbeitung bereitstellt  
✅ **Responsive UI** mit Dark/Light Mode bietet  
✅ **Sichere IPC-Kommunikation** implementiert  

Die App ist optimiert für reibungslose Überwachung ohne Systemüberlastung und bietet professionelle Visualisierungs- und Export-Möglichkeiten.
