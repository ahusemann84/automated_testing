    # Konzept zur Pufferung von REST-Daten (ABAP) — 2-Schichten-Modell

## Übersicht

Dieses Dokument beschreibt ein überarbeitetes Pufferungskonzept für Daten, die von einem Remote-System über eine REST-API in einer ABAP-Anwendung abgerufen werden. Es erweitert den bestehenden ABAP-Session-Puffer um eine persistente, nutzer-/serverübergreifende Schicht — *ohne* auf Shared Memory (`CL_SHM_AREA`) zurückzugreifen.

## Architektur

```
┌─────────────────────────────────────────────┐
│ Schicht 1: Session-Puffer (bestehend)        │  ← pro Nutzer, im Speicher, am schnellsten
│   - Singleton-Klasse, interne Tabelle/Hash   │
├─────────────────────────────────────────────┤
│ Schicht 2: Persistenter Puffer (neu)         │  ← nutzer-/serverübergreifend, überlebt Neustart
│   - Custom Z-Tabelle mit TTL / ETag          │
├─────────────────────────────────────────────┤
│ Schicht 0: Remote-REST-API                   │  ← nur bei vollständigem Cache-Miss angesprochen
└─────────────────────────────────────────────┘
```

## Begründung für den Verzicht auf Shared Memory

- Entfällt die Notwendigkeit, den Lebenszyklus von `CL_SHM_AREA` (Versionierung, Area-Attach/Detach) zu verwalten.
- Eine bewegliche Komponente weniger, die über mehrere Applikationsserver hinweg konsistent gehalten werden muss (Shared Objects sind ohnehin pro Applikationsserver lokal und bieten kein echtes serverübergreifendes Teilen — die Z-Tabelle deckt diesen Anwendungsfall bereits besser ab).
- Der Zugriff auf eine gepufferte/persistente Tabelle ist für die meisten REST-Caching-Anforderungen schnell genug und bietet Persistenz über Serverneustarts hinweg kostenlos mit.

## Zentrale Zugriffsklasse

```abap
CLASS zcl_rest_buffer DEFINITION.
  PUBLIC SECTION.
    CLASS-METHODS get_data
      IMPORTING iv_key         TYPE string
                iv_ttl_seconds TYPE i DEFAULT 300
      RETURNING VALUE(rv_data) TYPE string  " JSON-Payload
      RAISING   zcx_rest_error.
ENDCLASS.

CLASS zcl_rest_buffer IMPLEMENTATION.
  METHOD get_data.
    " 1. Session-Puffer (Schicht 1) prüfen - bei Treffer & Gültigkeit sofort zurückgeben
    IF zcl_session_buffer=>is_valid( iv_key ).
      rv_data = zcl_session_buffer=>get( iv_key ).
      RETURN.
    ENDIF.

    " 2. Persistente Z-Tabelle (Schicht 2) prüfen
    TRY.
        rv_data = zcl_persistent_buffer=>get( iv_key ).
        zcl_session_buffer=>set( iv_key = iv_key iv_data = rv_data ).
        RETURN.
      CATCH zcx_buffer_miss.
    ENDTRY.

    " 3. Cache-Miss in beiden Schichten -> sperren, REST-API aufrufen, Write-Through
    CALL FUNCTION 'ENQUEUE_EZ_REST_BUFFER' EXPORTING cache_key = iv_key.
    TRY.
        rv_data = zcl_rest_client=>fetch( iv_key ).
        zcl_persistent_buffer=>set( iv_key = iv_key iv_data = rv_data iv_ttl = iv_ttl_seconds ).
        zcl_session_buffer=>set( iv_key = iv_key iv_data = rv_data ).
      CLEANUP.
        CALL FUNCTION 'DEQUEUE_EZ_REST_BUFFER' EXPORTING cache_key = iv_key.
    ENDTRY.
    CALL FUNCTION 'DEQUEUE_EZ_REST_BUFFER' EXPORTING cache_key = iv_key.
  ENDMETHOD.
ENDCLASS.
```

## Persistente Puffertabelle: ZREST_BUFFER

| Feld | Typ | Zweck |
|---|---|---|
| `CACHE_KEY` | CHAR(100), Schlüssel | Hash aus Endpunkt + Parametern |
| `PAYLOAD` | STRING/RAWSTRING | Gepufferte JSON-Antwort |
| `ETAG` | CHAR(100) | Für Conditional GET (If-None-Match) |
| `CREATED_AT` | TIMESTAMPL | Zeitpunkt des Einfügens |
| `VALID_UNTIL` | TIMESTAMPL | Ablauf der TTL |

Aktivieren Sie *generische Pufferung/Einzelsatzpufferung* für diese Tabelle (Transaktion SE13), damit wiederholte Lesezugriffe auf denselben Schlüssel den Tabellenpuffer des Applikationsservers statt der Datenbank treffen. Dies gewinnt den Großteil des Performancevorteils zurück, den Shared Memory geboten hätte — ohne dessen Lebenszyklus-Komplexität.

## Zentrale Designentscheidungen

| Aspekt | Empfehlung |
|---|---|
| **TTL / Invalidierung** | `valid_until` pro Eintrag speichern; Vergleich mit `utclong_current( )`. Konfigurierbar je Datenkategorie (Stammdaten = Stunden, Preise = Minuten). |
| **Serverübergreifende Konsistenz** | Wird durch die Z-Tabelle (Single Source of Truth in der DB) + SAP-Tabellenpuffer-Invalidierungs-Broadcast auf natürliche Weise sichergestellt — keine eigene Synchronisationslogik nötig. |
| **Auslöser für Cache-Invalidierung** | ETag/Last-Modified nutzen, sofern die Remote-API dies unterstützt; in `ZREST_BUFFER-ETAG` speichern, Conditional Requests senden, TTL bei 304 ohne erneutes Parsen des Payloads auffrischen. |
| **Schutz vor Cache-Stampede** | `ENQUEUE`/`DEQUEUE` auf `cache_key`, damit gleichzeitige Cache-Misses nicht mehrfach denselben REST-Aufruf auslösen. |
| **Serialisierung** | JSON-String über `/ui2/cl_json`, damit die Pufferschicht formatunabhängig bleibt. |
| **Negative-Caching** | Kurzlebige "Miss/Error"-Marker puffern, um ein fehlerhaftes/langsames Remote-System vor wiederholten Anfragen zu schützen. |
| **Bereinigung** | Regelmäßiger Hintergrundjob (SM36), der abgelaufene Zeilen aus `ZREST_BUFFER` löscht; hält die Tabelle schlank für effiziente Pufferung. |

## Ablaufdiagramm

```
Anfrage nach Daten
      │
      ▼
Session-Puffer Treffer & gültig? ──Ja──► zurückgeben
      │ Nein
      ▼
Persistente Tabelle Treffer & gültig? ──Ja──► Session-Puffer befüllen ──► zurückgeben
      │ Nein
      ▼
ENQUEUE-Sperre auf cache_key
      │
      ▼
REST-API aufrufen (mit ETag, falls verfügbar)
      │
      ▼
Write-Through: Session-Puffer + Persistente Tabelle
      │
      ▼
DEQUEUE, Daten zurückgeben
```

## Implementierungshinweise

- Das Design hinter einer Schnittstelle `ZIF_REST_BUFFER` kapseln, damit die Implementierung der persistenten Schicht (Z-Tabelle vs. HANA-gepufferte Tabelle vs. zukünftige Alternative) ausgetauscht werden kann, ohne den aufrufenden Code anzupassen.
- Die Pufferungseinstellung der Tabelle ("Vollständig gepuffert" vs. "Generisch bereichsgepuffert") sollte anhand der Schlüsselkardinalität gewählt werden — generische Pufferung auf dem `CACHE_KEY`-Präfix funktioniert gut, wenn die Schlüssel strukturiert sind (z. B. `ENDPUNKT_PARAMHASH`).
- Den eigentlichen HTTP-Aufruf (`cl_http_client` / `if_rest_client`) hinter einer Provider-Klasse kapseln, die in die Puffer-Fassade injiziert wird — so bleibt die Pufferungslogik vollständig von der Transportschicht entkoppelt.
