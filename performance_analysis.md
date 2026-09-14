# Allgemeines Vorgehen: Untersuchung der SAP-Performance für WebClient UI Framework (WCF) Anwendungen

Das WebClient UI (verwendet in CRM, Solution Manager usw.) besitzt zusätzliche Architekturschichten oberhalb des Standard-ABAP-Stacks. Daher muss die Untersuchung schichtweise von "breit" zu "eng" erfolgen.

## 1. Problem reproduzieren & eingrenzen
- Genaue Anwendung/Komponente identifizieren (BSP-Anwendung, UI-Komponente, View), sowie Benutzer, Mandant und die exakte Aktion (initiales Laden, Navigation, Suche, Speichern).
- Zeitstempel erfassen und prüfen, ob die Langsamkeit isoliert (einzelner Benutzer), systemisch (alle Benutzer) oder abhängig von einem bestimmten Geschäftsobjekt/Datenvolumen auftritt.

## 2. Übergeordnete Workload-Analyse
- **STAD / ST03N** – Aufschlüsselung der Antwortzeit (Frontend, Datenbank, CPU, Wartezeit, GUI-/Netzwerkzeit) für Transaktion oder Benutzersitzung abrufen. So lässt sich vor der Detailanalyse feststellen, ob der Engpass vermutlich im Frontend, Applikationsserver, in der Datenbank oder im Netzwerk liegt.

## 3. WebClient UI-spezifische Diagnose
- **Transaktion UIA (UI Analysis Tool)** – Das zentrale Werkzeug für WCF; zerlegt die Antwortzeit in UI-Framework-spezifische Phasen: Event-Verarbeitung, Controller-Verarbeitung (BOL/GenIL), Modellbindung, Laden von Konfiguration/Personalisierung, View-Rendering (Context-Node-Zugriff, Feld-Rendering) und Roundtrips zum Backend. Damit lässt sich feststellen, ob die Verzögerung in der UI-Schicht selbst oder in der zugrunde liegenden Geschäftslogik liegt.
- **BSP-Cache** sowie **UI-Konfiguration/Personalisierung** auf Overhead prüfen (eine übermäßige Anzahl von Assignment Blocks, Views oder Feldern kann das Rendering verlangsamen).
- **GenIL/BOL-Schicht** auf übermäßige Objektinstanziierungen, unnötige Buffer-Refreshes oder redundante Abfragen überprüfen.

## 4. ABAP-/Backend-Laufzeitanalyse
- **ST12 (kombinierter ABAP + SQL Trace)** – Bevorzugtes Werkzeug für Tiefenanalysen; erfasst sowohl ABAP-Laufzeit als auch SQL-Anweisungen in einer sitzungsbasierten Trace-Aufzeichnung, korreliert nach Benutzer/Transaktion.
- **SAT (Laufzeitanalyse)** – Detaillierte Analyse von ABAP-Methoden-/Funktionslaufzeiten, um Hotspots in Custom Code, BAdIs, User-Exits oder häufig aufgerufenen Standard-Framework-Methoden zu identifizieren.
- **ST05 (SQL-Trace)** – Falls der Datenbankzugriff auffällig hoch erscheint, teure/doppelte SQL-Anweisungen, fehlende Indizes oder übermäßige Tabellenzugriffe isolieren (Achtung auf typische N+1-Muster bei BOL/GenIL-Modellabfragen).

## 5. RFC-/Middleware-Prüfung (falls zutreffend)
- Falls der WebClient über RFC mit Backend-Systemen kommuniziert (z. B. CRM ↔ ERP), RFC-Aufrufe tracen (ST12 beinhaltet dies) um Netzwerklatenzen oder Warteschlangenverzögerungen zu erkennen.

## 6. Frontend-/Netzwerkschicht
- Browser-Entwicklertools (Netzwerk-/Performance-Tab) oder SAP Web Dispatcher-/ICM-Logs verwenden, um Netzwerklatenz, große Payloads (JS/CSS/Bilder) oder übermäßige Roundtrips zwischen Browser und Server auszuschließen.
- ICM-/Web Dispatcher-Warteschlangenstatistiken auf Überlastung prüfen.

## 7. Überprüfung von Custom Code & Konfiguration
- Kundenerweiterungen (BAdIs, EEWB-Felder, Custom Controller/Views), die in ST12/SAT-Traces als Hotspots auffallen, überprüfen.
- Anzahl aktiver Konfigurationen, Personalisierungseinstellungen und Assignment Blocks prüfen – dies sind häufige, nicht code-bezogene Ursachen für WCF-Langsamkeit.

## 8. Iterieren und Validieren
- Nach jeder Korrektur (Index, Code-Optimierung, Konfigurationsvereinfachung) STAD/UIA/ST12 erneut ausführen, um die Verbesserung zu bestätigen und Regressionen an anderer Stelle auszuschließen.

## Zusammenfassende Tabelle

| Schicht | Werkzeug | Zweck |
|---|---|---|
| Übersicht | STAD / ST03N | Aufschlüsselung der Antwortzeit nach Komponente |
| UI-Framework | UIA | WCF-spezifische Phasenanalyse (Rendering, BOL, Konfiguration) |
| ABAP + SQL | ST12 | Kombinierter Trace, korreliert Code und Datenbank |
| Nur ABAP | SAT | Detaillierte Laufzeitanalyse von Custom Code |
| Nur SQL | ST05 | Analyse von Datenbankzugriffen/Abfragen |
| Netzwerk | Browser-Entwicklertools, Web Dispatcher-Logs | Frontend-/Netzwerklatenz |

**Best Practice:** Zunächst breit ansetzen (STAD) → WCF-spezifischen Overhead isolieren (UIA) → ins Backend eintauchen (ST12/SAT/ST05) → Netzwerk/Frontend prüfen → beheben und iterativ erneut validieren.
