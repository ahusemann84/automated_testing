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

Die Puffer-Fassade (`zcl_rest_buffer`) kennt keine HTTP-Details. Sie delegiert das eigentliche Abrufen der Daten bei einem Cache-Miss an eine **injizierte Provider-Instanz**, die das Interface `zif_rest_provider` implementiert. Dadurch bleibt die Pufferlogik vollständig unabhängig vom Transportmechanismus (REST, SOAP, RFC, etc.).

```abap
CLASS zcl_rest_buffer DEFINITION.
  PUBLIC SECTION.
    METHODS constructor
      IMPORTING io_provider TYPE REF TO zif_rest_provider.

    METHODS get_data
      IMPORTING iv_key         TYPE string
                iv_ttl_seconds TYPE i DEFAULT 300
      RETURNING VALUE(rv_data) TYPE string  " JSON-Payload
      RAISING   zcx_rest_error.

  PRIVATE SECTION.
    DATA mo_provider TYPE REF TO zif_rest_provider.
ENDCLASS.

CLASS zcl_rest_buffer IMPLEMENTATION.
  METHOD constructor.
    mo_provider = io_provider.
  ENDMETHOD.

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

    " 3. Cache-Miss in beiden Schichten -> sperren, Provider aufrufen, Write-Through
    CALL FUNCTION 'ENQUEUE_EZ_REST_BUFFER' EXPORTING cache_key = iv_key.
    TRY.
        " Delegation an den injizierten Provider statt direktem HTTP-Aufruf
        rv_data = mo_provider->fetch(
          iv_key  = iv_key
          iv_etag = zcl_persistent_buffer=>get_etag( iv_key ) ).

        zcl_persistent_buffer=>set(
          iv_key  = iv_key
          iv_data = rv_data
          iv_ttl  = iv_ttl_seconds
          iv_etag = mo_provider->get_last_etag( ) ).
        zcl_session_buffer=>set( iv_key = iv_key iv_data = rv_data ).
      CLEANUP.
        CALL FUNCTION 'DEQUEUE_EZ_REST_BUFFER' EXPORTING cache_key = iv_key.
    ENDTRY.
    CALL FUNCTION 'DEQUEUE_EZ_REST_BUFFER' EXPORTING cache_key = iv_key.
  ENDMETHOD.
ENDCLASS.
```

## Provider-Klasse und Interaktion mit der Puffer-Fassade

Die Provider-Klasse kapselt ausschließlich den Transport (HTTP/REST) und ist über ein schlankes Interface an die Puffer-Fassade angebunden. Dieses Zusammenspiel folgt dem **Dependency-Injection**-Prinzip: die Fassade kennt nur das Interface, nicht die konkrete Implementierung.

```abap
INTERFACE zif_rest_provider.
  METHODS fetch
    IMPORTING iv_key          TYPE string
              iv_etag         TYPE string OPTIONAL
    RETURNING VALUE(rv_data)  TYPE string
    RAISING   zcx_rest_error.

  METHODS get_last_etag
    RETURNING VALUE(rv_etag) TYPE string.
ENDINTERFACE.

CLASS zcl_rest_provider DEFINITION.
  PUBLIC SECTION.
    INTERFACES zif_rest_provider.
    METHODS constructor
      IMPORTING iv_destination TYPE string.

  PRIVATE SECTION.
    DATA mv_destination TYPE string.
    DATA mv_last_etag   TYPE string.
    DATA mo_http_client  TYPE REF TO if_http_client.
ENDCLASS.

CLASS zcl_rest_provider IMPLEMENTATION.
  METHOD constructor.
    mv_destination = iv_destination.
  ENDMETHOD.

  METHOD zif_rest_provider~fetch.
    " 1. HTTP-Client für die konfigurierte Destination aufbauen (z. B. via cl_http_client=>create_by_destination)
    " 2. Falls iv_etag übergeben wurde, den Header 'If-None-Match' setzen -> Conditional GET
    " 3. Request absenden und Antwort auswerten:
    "    - 200 OK  -> Payload zurückgeben, neuen ETag aus Response-Header in mv_last_etag ablegen
    "    - 304 Not Modified -> zcx_rest_error mit Kennzeichen "not_modified" auslösen,
    "      damit die Fassade den bestehenden Puffereintrag lediglich verlängert (TTL-Refresh)
    "    - 4xx/5xx -> zcx_rest_error auslösen, Fassade kann Negative-Caching anwenden
  ENDMETHOD.

  METHOD zif_rest_provider~get_last_etag.
    rv_etag = mv_last_etag.
  ENDMETHOD.
ENDCLASS.
```

### Interaktionsablauf zwischen Fassade und Provider

1. **Instanziierung / Injection**: Der Aufrufer (z. B. eine Business-Klasse) erzeugt `zcl_rest_buffer` und übergibt eine konkrete Provider-Instanz im Konstruktor (`NEW zcl_rest_buffer( io_provider = NEW zcl_rest_provider( 'Z_MY_DESTINATION' ) )`). Dadurch lässt sich der Provider pro Anwendungsfall austauschen (z. B. unterschiedliche Destinationen, Mock-Provider für Tests).
2. **Nur bei Cache-Miss aktiv**: Der Provider wird ausschließlich dann angesprochen, wenn sowohl Session- als auch persistenter Puffer keinen gültigen Treffer liefern — die Fassade übernimmt die gesamte Entscheidungslogik, der Provider bleibt zustandslos bezüglich Caching.
3. **ETag-Weitergabe**: Die Fassade liest einen ggf. vorhandenen ETag aus dem persistenten Puffer und reicht ihn als `iv_etag` an `fetch( )` weiter. Der Provider nutzt ihn für einen Conditional GET (`If-None-Match`-Header).
4. **Antwortverarbeitung**: 
   - Bei `200 OK` liefert der Provider den vollständigen Payload zurück; die Fassade schreibt ihn inklusive des neuen ETags (`get_last_etag( )`) in beide Pufferschichten (Write-Through).
   - Bei `304 Not Modified` signalisiert der Provider dies über eine spezielle Exception; die Fassade verlängert lediglich die TTL des bestehenden Puffereintrags, ohne den Payload neu zu schreiben.
   - Bei Fehlern (`4xx`/`5xx`) wirft der Provider `zcx_rest_error`; die Fassade kann daraufhin optional einen kurzlebigen Negative-Cache-Eintrag anlegen, um wiederholte Fehlanfragen an ein instabiles Remote-System zu vermeiden.
5. **Austauschbarkeit für Tests**: Da die Fassade nur gegen `zif_rest_provider` programmiert, lässt sich in Unit-Tests problemlos ein Test-Double (`zcl_rest_provider_mock`) injizieren, das feste Antworten liefert — ohne echten HTTP-Aufruf.

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
Provider->fetch() aufrufen (mit ETag, falls verfügbar)
      │
      ├── 200 OK           -> Write-Through: Session-Puffer + Persistente Tabelle (inkl. neuem ETag)
      ├── 304 Not Modified -> nur TTL des bestehenden Puffereintrags verlängern
      └── 4xx/5xx          -> optional Negative-Cache-Eintrag anlegen
      │
      ▼
DEQUEUE, Daten zurückgeben
```

## Implementierungshinweise

- Das Design hinter einer Schnittstelle `ZIF_REST_BUFFER` kapseln, damit die Implementierung der persistenten Schicht (Z-Tabelle vs. HANA-gepufferte Tabelle vs. zukünftige Alternative) ausgetauscht werden kann, ohne den aufrufenden Code anzupassen.
- Die Pufferungseinstellung der Tabelle ("Vollständig gepuffert" vs. "Generisch bereichsgepuffert") sollte anhand der Schlüsselkardinalität gewählt werden — generische Pufferung auf dem `CACHE_KEY`-Präfix funktioniert gut, wenn die Schlüssel strukturiert sind (z. B. `ENDPUNKT_PARAMHASH`).
- Den eigentlichen HTTP-Aufruf (`cl_http_client` / `if_rest_client`) hinter einer Provider-Klasse kapseln, die über das Interface `zif_rest_provider` in die Puffer-Fassade injiziert wird — so bleibt die Pufferungslogik vollständig von der Transportschicht entkoppelt und der Provider pro Anwendungsfall (Destination, Mock für Tests) austauschbar.
