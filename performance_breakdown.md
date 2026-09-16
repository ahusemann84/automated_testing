# SAP CRM Performance – Diagrammatische Aufschlüsselung & UI-Technologie-Fokus

## 1. Antwortzeit-Zerlegung (Diagramm): CRM WebClient UI vs. UI5

```
┌──────────────────────────────────────────────────────────────────────────┐
│                  GESAMTE ANTWORTZEIT (aus Nutzersicht)                   │
└──────────────────────────────────────────────────────────────────────────┘
        │
        ├── 1) CLIENT-/BROWSER-ZEIT
        │     ├─ CRM WebClient UI (BSP/BOL-GENIL-basiert, ABAP-gerendertes HTML)
        │     │    - HTML/DOM-Rendering großer BSP-Seiten
        │     │    - JavaScript (WCF-Client-Framework) Event-Handling
        │     │    - Vollständiges/partielles Neuladen der Seite pro Navigation
        │     └─ SAP UI5 / Fiori (CRM Fiori Apps, z. B. My CRM, Lead Mgmt)
        │          - OData-Modellbindung & JSON-Parsing
        │          - UI5-Control-Rendering (clientseitig, umfangreichere JS-Laufzeit)
        │          - SPA-Verhalten: kein vollständiges Neuladen, aber schwereres
        │            initiales Laden
        │
        ├── 2) NETZWERKZEIT
        │     ├─ WebClient UI: größere HTML-Payloads pro Roundtrip (View-Zusammenbau)
        │     └─ UI5: mehrere OData-Batch-Aufrufe, kleinere Payloads, aber mehr Requests
        │
        ├── 3) APPLICATION-SERVER-ZEIT (ABAP-Stack)
        │     ├─ Dialog-Wartezeit (Warten auf freien Workprozess)
        │     ├─ Roll-in-/Roll-out-Zeit
        │     │     - WCF: großer ABAP-Session-Kontext (BOL-Entitäten, View-Kontext,
        │     │       Navigationshistorie) pro Nutzer → stärkere Nutzung des Rollbereichs
        │     │     - UI5/OData (Gateway): typischerweise zustandsloser pro Request,
        │     │       geringerer Rollbereichs-Bedarf, aber SICF/Gateway-Overhead
        │     ├─ Lade-/Generierungszeit (BSP-Komponenten, CHTMLB-Tags, Floorplans)
        │     ├─ BOL/GENIL-Schicht-Verarbeitung
        │     │     - Objektinstanziierung, Relation-Buffering, Query-Ausführung
        │     │     - CRM-spezifisches Caching (BOL-Puffer, GENIL-Modellpuffer)
        │     └─ CPU-Zeit (Geschäftslogik: BAdIs, Aktionen, Regel-Policies, PPF)
        │
        ├── 4) DATENBANKZEIT  ◄── meist größter & zustandsabhängigster Anteil
        │     ├─ CRM-Tabellen: CRMD_ORDERADM_H/I, CRMD_PARTNER, CRMD_LINK, etc.
        │     ├─ Puffer-Trefferquote (Customizing-, Konditions-/Preistabellen)
        │     ├─ Qualität von Index/Statistik (große transaktionale CRM-Tabellen)
        │     ├─ Sperrwartezeiten (gemeinsame Objekte, Statusverwaltung, Änderungsbelege)
        │     └─ I/O-Wartezeit (speicherbedingte Lesevorgänge bei großer Belegs-Historie)
        │
        └── 5) MIDDLEWARE-/INTEGRATIONSZEIT (falls zutreffend)
              - CRM Middleware (BDoc-Verarbeitung, Queues) - asynchron, kann aber die
                Datenverfügbarkeit verzögern und die gefühlte Performance beeinflussen
              - SAP Gateway / OData-Schicht (für UI5-Apps) - Metadaten- und
                Entitäts-Serialisierungs-Overhead
```

## 2. Komponentenvergleich: WebClient UI vs. UI5

| Aspekt | CRM WebClient UI (BSP/BOL) | SAP UI5 / Fiori (OData-basiert) |
|---|---|---|
| Rendering-Modell | Serverseitige HTML-Generierung pro View | Clientseitiges Rendering (SPA) |
| Session-Zustand | Schwerer ABAP-Session-Zustand (BOL-Kontext, Navigations-Stack) serverseitig gehalten | Leichterer Server-Zustand; mehr Logik clientseitig |
| Roll-in-/Roll-out-Auswirkung | Hoch — großer Kontext-Rollbereich, teurer pro Request | Geringer — zustandslose OData-Aufrufe reduzieren Rollkosten pro Request |
| Nutzen von Buffering | Stark: BOL/GENIL-Puffer + Tabellenpuffer reduzieren wiederholte DB-Zugriffe für Stamm-/Customizing-Daten | Gleiches DB-seitiges Buffering relevant (über OData → BOL/BAPI → DB) |
| Netzwerkmuster | Weniger, größere Roundtrips | Mehr, kleinere Roundtrips (Batch-OData-Requests mildern dies ab) |
| Empfindlichkeit gg. DB-Zustand | Hoch (BOL-Query-Performance hängt von gut getunten CRM-DB-Tabellen/-Indizes ab) | Ebenso hoch darunter — OData-Services kapseln meist BOL/BAPI-Aufrufe auf dieselben CRM-Tabellen |
| Typische Analyse-Tools | ST05 (SQL-Trace), SAT (ABAP-Trace), GENIL/BOL-Trace, ST03N | Gateway-Trace (/IWFND/TRACES), ST03N, SAT, OData-$expand-Analyse |

## 3. Datenbankzustand — CRM-spezifische Aspekte

- **Transaktionale Tabellen** (`CRMD_ORDERADM_H`, `CRMD_ORDERADM_I`, `CRMD_LINK`, `CRMD_PARTNER`) wachsen kontinuierlich mit Geschäftsvorgängen. Ohne regelmäßige Archivierung verschlechtern sich Ausführungspläne — ein anfangs "gesunder" DB-Zustand wird über die Jahre zum Flaschenhals.
- **Customizing- und Konditionstabellen** (Preisfindung, Statusprofile, Partnerermittlung) sind ideale Kandidaten für **Tabellenpufferung** — CRM ist bei der BOL-Objekterstellung stark auf gepufferte Customizing-Lookups angewiesen; ein kalter oder invalidierter Puffer führt zu einem Anstieg der DB-Zugriffe pro Transaktionsaufruf.
- **Aktualität der Statistiken**: Das flexible Datenmodell von CRM (EEWB-erweiterte Felder, generische Tabellen) kann den kostenbasierten Optimizer verwirren, wenn Statistiken nach Massendatenladungen (z. B. nach Middleware-Initialload oder Massendatenmigration) nicht aktualisiert werden.
- **Sperren**: Das Statusmanagement und Änderungsbeleg-Framework von CRM kann zu **Sperrenkonflikten** bei gemeinsam genutzten Business-Objekten führen (z. B. Serviceaufträge, die von mehreren Bearbeitern bearbeitet werden), was die DB-Wartezeit erhöht und in der WebClient UI effektiv den gesamten Rollbereich/die Session blockiert, bis die Sperre aufgelöst ist.

## 4. Serverseitiges Buffering im CRM-Kontext — Auswirkung auf die Laufzeit

| Puffer-Ebene | Was wird gepuffert | Auswirkung auf die Laufzeit |
|---|---|---|
| **BOL/GENIL-Puffer** | Business-Objektinstanzen, Relationen, Queries innerhalb einer Nutzersession | Vermeidet wiederholte BAPI-/DB-Aufrufe für dasselbe Objekt innerhalb einer Transaktion (z. B. erneutes Öffnen desselben Serviceauftrags-Tabs) — großer Laufzeitgewinn in der WebClient UI |
| **Tabellenpuffer (Customizing)** | Statusprofile, Partnerfunktionen, Preiskonditionen, Organisationsmodell | Wandelt DB-Roundtrips in lokale Speicherzugriffe um — entscheidend, da diese Tabellen bei Auftrags-/Lead-Verarbeitung ständig gelesen werden |
| **ABAP-Session-Kontext (Rollbereich/Extended Memory)** | WCF-Navigationshistorie, View-Kontext, BOL-Transaktionskontext | Große CRM-Sessions (viele offene Tabs, komplexe Zuordnungsblöcke) erhöhen die Roll-in-/out-Kosten; bei Erschöpfung des Extended Memory fallen Workprozesse auf privaten/Heap-Speicher zurück → Laufzeitverschlechterung und mögliche Workprozess-Blockade |
| **OData-Modell-Cache (UI5-Seite)** | Metadaten-Dokumente, gebündelte Entitätsdaten im Browser | Reduziert wiederholte Metadaten-Aufrufe, aber ein veralteter Cache kann zusätzliche Roundtrips (Cache-Invalidierung) oder subtil falsche Laufzeitdaten verursachen, wenn er nicht sauber verwaltet wird |

**Zentrale Erkenntnisse speziell für CRM:**

1. In der **WebClient UI** ist der BOL/GENIL-Puffer der wirkungsvollste serverseitige Puffer — er reduziert direkt Backend-/Datenbankzugriffe bei der Navigation zwischen Zuordnungsblöcken derselben Transaktion. Schlecht gepufferte oder häufig invalidierte BOL-Queries (z. B. durch benutzerdefinierte BAdIs, die das Buffering umgehen) sind eine sehr häufige Ursache für langsame Ladezeiten von Transaktionen.
2. Da CRM-WebClient-Sessions deutlich mehr serverseitigen Zustand pro Nutzer tragen als typische UI5-/Fiori-Apps, sind die **Roll-in-/Roll-out-Kosten ein größerer Faktor** bei der CRM-WebClient-Performance als bei UI5-basierten CRM-Apps, die tendenziell zustandsloser sind und den Zustand in den Browser auslagern.
3. Bei **UI5-/Fiori-basierten CRM-Erweiterungen** ist die Performance stärker OData-/Gateway-gebunden: Das darunterliegende DB- und BOL-Buffering gilt weiterhin (da Gateway-Services meist dieselbe BOL-Schicht aufrufen), sodass die Empfindlichkeit gegenüber dem DB-Zustand ebenso vorhanden ist — nur anders gemessen (Gateway-Trace statt ST03N-Dialogschritte).
4. Unabhängig von der UI-Technologie bleibt ein ungesunder **Datenbankzustand** (veraltete Statistiken, Tabellenwachstum, Sperrenkonflikte) der dominierende Faktor für die gesamte CRM-Antwortzeit — Buffering mildert nur, *wie oft* die DB angesprochen wird, nicht *wie teuer* jeder Zugriff bei einem Cache-Miss ist.

## 5. Checkliste: Tuning CRM WebClient UI vs. UI5-Apps

### Allgemein (beide Technologien)

- [ ] Datenbankstatistiken für CRM-Kerntabellen regelmäßig aktualisieren (insbesondere nach Massendatenladungen)
- [ ] Archivierungsstrategie für `CRMD_ORDERADM_H/I`, `CRMD_LINK`, Change Documents etablieren
- [ ] Tabellenpuffer-Einstellungen für Customizing-/Konditionstabellen prüfen (Transaktion SE13/RSRV)
- [ ] Sperrenkonflikte analysieren (SM12) bei parallelem Zugriff auf Serviceaufträge/Leads
- [ ] SQL-Traces (ST05) für teure/wiederholte Statements bei typischen Geschäftsvorfällen aufnehmen
- [ ] ST03N/STAD regelmäßig auswerten, um Antwortzeit-Trends zu erkennen

### CRM WebClient UI (BSP/BOL/GENIL)

- [ ] BOL/GENIL-Puffer-Nutzung prüfen — unnötige Query-Wiederholungen vermeiden
- [ ] Custom-BAdIs/Erweiterungen daraufhin prüfen, ob sie das BOL-Buffering umgehen
- [ ] Anzahl und Komplexität der Zuordnungsblöcke (Assignment Blocks) pro View reduzieren
- [ ] Konfiguration der Navigationshistorie/Session-Kontextgröße überprüfen (Rollbereich-Verbrauch)
- [ ] Unnötige Full-Page-Reloads vermeiden (Nutzung von "Do Navigate" statt vollständigem Refresh)
- [ ] Extended-Memory-Konfiguration (`em/initial_size_MB`, `ztta/roll_extension`) für Workprozesse prüfen
- [ ] SAT-Trace auf BOL-Klassen (`CL_CRM_BOL_*`) anwenden, um teure Objektinstanziierungen zu finden
- [ ] Anzahl paralleler Dialogprozesse und Wartezeiten (SM50/SM66) unter Lastspitzen überwachen

### SAP UI5 / Fiori (OData-basiert)

- [ ] OData-Services auf unnötige `$expand`/`$select`-Overhead prüfen (nur benötigte Felder laden)
- [ ] Batch-Requests statt vieler einzelner OData-Calls nutzen
- [ ] Gateway-Trace (`/IWFND/TRACES`, `/IWFND/ERROR_LOG`) für langsame oder fehlerhafte Requests analysieren
- [ ] Metadaten-Caching im Frontend aktivieren, aber Invalidierungsstrategie definieren
- [ ] Anzahl der Roundtrips beim initialen App-Start minimieren (Lazy Loading von Fragmenten/Views)
- [ ] Serverseitige Gateway-Performance messen (Zeit in `/IWBEP/*`-Klassen vs. reine BOL/BAPI-Zeit)
- [ ] Browser-seitiges Rendering/Performance-Profiling (Chrome DevTools, UI5 Diagnostics) durchführen
- [ ] Sicherstellen, dass zugrunde liegende BOL/BAPI-Aufrufe dieselben Buffering-Vorteile nutzen wie die WebClient UI

---

*Hinweis: Diese Datei ist eine deutsche Übersetzung mit Erweiterung der ursprünglichen englischsprachigen Performance-Analyse für SAP CRM.*
