# REST-Pufferkonzept (ABAP, Deutsch) ohne Shared Objects

## Zweck und Geltungsbereich

Dieses Dokument beschreibt ein **konzeptionelles** 2-Schichten-Modell für REST-GET-Caching in ABAP:

1. bestehender Session-Cache (pro Session)
2. persistente, **ungepufferte** Z-Tabelle (serverübergreifend über DB)

Es wird **kein** application-managed Shared Memory / Shared Objects (`CL_SHM_AREA`) eingesetzt. Alle ABAP-Beispiele sind bewusst **Skeletons** (nicht direkt aktivierbar). Fehlende Typen, Interfaces, Klassen und Exceptions sind als vertragliche Bausteine zu implementieren.

---

## Architektur und Verantwortungen

### Schichten

```text
┌──────────────────────────────────────────────────────────────┐
│ Schicht 1: Session-Cache (bestehend)                         │
│ - schnell, sessionlokal, darf begrenzt stale sein            │
├──────────────────────────────────────────────────────────────┤
│ Schicht 2: Persistenter Cache (neu/überarbeitet)             │
│ - ungepufferte Z-Tabelle, Cache-Metadaten + Payload          │
├──────────────────────────────────────────────────────────────┤
│ Schicht 0: Remote REST API                                   │
│ - nur bei Miss/Revalidierung                                 │
└──────────────────────────────────────────────────────────────┘
```

### Kollaboratoren (Rollen)

| Baustein | Verantwortung | Darf nicht |
|---|---|---|
| `zif_rest_provider` + Implementierung | HTTP-GET gegen Destination, Rückgabe strukturierter Antwort inkl. Header/Status/Zeitstempel | Kein Cachezugriff, kein `COMMIT WORK`/`ROLLBACK WORK`, kein mutable „last response“-State |
| `zcl_rest_cache_facade` | Key-Building, Security-Scope-Prüfung, Session/Persistent-Lookup, Orchestrierung Refresh | Kein direkter Transportcode |
| `zif_rest_refresh_unit` | Locking, Re-Read nach Lock, Provider-Aufruf, Cache-SQL, Commit/Rollback des **Cache-only**-Abschnitts | Keine Business-Änderungen |
| `zif_rest_session_store` | Session-Lesen/Schreiben/Löschen | Kein Persistenzzugriff |
| `zif_rest_persistent_store` | SQL für persistenten Cache (ungepufferte Tabelle) | Kein HTTP |
| `zif_rest_key_builder` | deterministische Schlüsselbildung inkl. Scope/Varianten | Keine Credentials/Tokens im Key |
| `zif_rest_cache_policy` | Deterministische Cachebarkeit/Freshness/Retention/`Vary`/`no-store` | Keine Transaktionssteuerung, kein I/O |

---

## Provider-Vertrag (zustandslos)

> **Konzept-Skeleton**: nicht aktivierungsfertig.

```abap
INTERFACE zif_rest_provider PUBLIC.
  TYPES: BEGIN OF ty_header,
          name  TYPE string,
          value TYPE string,
         END OF ty_header,
         ty_headers TYPE STANDARD TABLE OF ty_header WITH EMPTY KEY.

  TYPES: BEGIN OF ty_request_descriptor,
           destination_id         TYPE string,
           sap_client             TYPE mandt,
           resource_path          TYPE string,
           normalized_query       TYPE string,
           accept_language        TYPE sylangu,
           trusted_security_scope TYPE string,
           representation_headers TYPE ty_headers, "z. B. Accept, Accept-Language, Custom-Vary-Header
         END OF ty_request_descriptor.

  TYPES: BEGIN OF ty_validator,
          etag          TYPE string,
          last_modified TYPE string,
         END OF ty_validator.

  TYPES: BEGIN OF ty_response,
           http_status           TYPE i,
           payload_x             TYPE xstring,
           headers               TYPE ty_headers, "Duplikate bleiben erhalten
           sent_at_utc           TYPE timestampl,
           received_at_utc       TYPE timestampl,
         END OF ty_response.

  METHODS fetch_get
    IMPORTING
      is_request     TYPE ty_request_descriptor
      is_validator   TYPE ty_validator OPTIONAL
    RETURNING
      VALUE(rs_resp) TYPE ty_response
    RAISING
      zcx_rest_transport_failure
      zcx_rest_protocol_failure.
ENDINTERFACE.
```

### Regeln für den Provider

- Führt nur HTTP-GET aus; Credentials kommen aus der Destination.
- Kein Zugriff auf Session-/Persistent-Cache.
- Keine transaktionalen Statements (`COMMIT WORK`, `ROLLBACK WORK`) und keine Update-Task-Registrierung.
- Kein `get_last_etag`, kein `mv_last_etag`.
- HTTP-Status wie `304`, `404`, `429`, `503` werden **normal** im Ergebnis geliefert.
- Exception nur bei Transport-/Protokollfehlern ohne verwertbare Antwort.
- Request-Descriptor enthält echte Request-Dimensionen inkl. darstellungsrelevanter Header; ein Cache-Hash allein ist unzulässig.

---

## Fassade und Schlüsselisolation

Die Fassade erhält ihre Abhängigkeiten per Injection:

- Session-Store
- Persistent-Store
- Key-Builder
- Policy
- Refresh-Unit
- optional Clock/Lock-Adapter

Cache-Key trennt mindestens:

- Destination
- SAP-Client
- Resource + normalisierte Query
- Repräsentation/Sprache
- Autorisierungs-/Security-Scope

**Nie** Teil des Keys: Credentials, Access-Tokens, Secret-Material.

Session-Lesepfad:

1. Session-Read
2. Persistent-Read
3. Refresh

Ein `found`-Flag unterscheidet „kein Eintrag“ von „gültiger leerer Payload“. Promotion von persistent nach Session übernimmt **absolute Expiry** und Metadaten unverändert (kein TTL-Neustart). Vor Session-Neuschreiben nach Refresh wird ein alter Session-Eintrag des Callers entfernt; Schreiben nur wenn `session_allowed = abap_true`.

---

## Shared Types `zif_rest_cache_types` (Folgeerweiterung)

> **Konzept-Skeleton**: nicht aktivierungsfertig; benötigte DDIC-/Klassenobjekte sind projektspezifisch nachzuziehen.

```abap
INTERFACE zif_rest_cache_types PUBLIC.
  TYPES ty_key TYPE c LENGTH 64. "stabiler Hash-Key

  TYPES ty_layer TYPE c LENGTH 1.
  CONSTANTS:
    co_session    TYPE ty_layer VALUE 'S',
    co_persistent TYPE ty_layer VALUE 'P'.

  TYPES: BEGIN OF ty_entry,
           found                   TYPE abap_bool,
           cache_key               TYPE ty_key,
           schema_version          TYPE i,
           payload_x               TYPE xstring,
           headers                 TYPE zif_rest_provider=>ty_headers,
           validators              TYPE zif_rest_provider=>ty_validator,
           destination_id          TYPE string,
           sap_client              TYPE mandt,
           trusted_security_scope  TYPE string,
           variant_fingerprint     TYPE string,
           stored_at               TYPE timestampl,
           validated_at            TYPE timestampl,
           fresh_until_session     TYPE timestampl,
           fresh_until_persistent  TYPE timestampl,
           retain_until            TYPE timestampl,
           session_allowed         TYPE abap_bool,
           persistent_allowed      TYPE abap_bool,
           revalidate_always       TYPE abap_bool,
         END OF ty_entry.

  TYPES ty_entries TYPE HASHED TABLE OF ty_entry WITH UNIQUE KEY cache_key.

  TYPES: BEGIN OF ty_decision,
           reusable TYPE abap_bool,
           reason   TYPE string, "z. B. MISS/FRESH/EXPIRED/CONTEXT_MISMATCH
         END OF ty_decision.

  TYPES: BEGIN OF ty_refresh,
           entry               TYPE ty_entry,
           session_allowed     TYPE abap_bool,
           persistent_allowed  TYPE abap_bool,
         END OF ty_refresh.
ENDINTERFACE.
```

Semantik und Invarianten:

- `found = abap_false` bedeutet „kein Eintrag“; das ist **nicht** gleichbedeutend mit „leerer, aber gültiger Payload“.
- Promotion persistent -> Session übernimmt absolute Deadlines unverändert (kein TTL-Neustart, keine Deadline-Verlängerung).
- `retain_until` dient Revalidation/Cleanup, **nicht** der Auslieferung stale Daten.
- Reuse prüft immer Schlüssel + Kontext + Schema vor Freshness.
- Session-/Persistent-Freshness können differieren (z. B. `max-age` vs. `s-maxage`) und berücksichtigen Origin-Age (`Date`/`Age`), nicht „TTL ab jetzt“.
- Falls `ty_refresh` zusätzlich Layer-Flags führt, müssen sie konsistent zu `entry-session_allowed`/`entry-persistent_allowed` sein (keine divergierenden Wahrheiten).
- Schlüssel isolieren mindestens Destination/Client/Autorisierung/Repräsentation; keine Credentials/Tokens speichern.
- `Vary` nur nutzen, wenn Key-/Lookup-Schema die Header sicher abbildet; sonst Caching konservativ ablehnen.

## Side-effect-freier Policy-Vertrag `zif_rest_cache_policy`

```abap
INTERFACE zif_rest_cache_policy PUBLIC.
  METHODS can_reuse
    IMPORTING
      is_entry             TYPE zif_rest_cache_types=>ty_entry
      is_request           TYPE zif_rest_provider=>ty_request_descriptor
      iv_key               TYPE zif_rest_cache_types=>ty_key
      iv_layer             TYPE zif_rest_cache_types=>ty_layer
      iv_now               TYPE timestampl
    RETURNING
      VALUE(rs_decision)   TYPE zif_rest_cache_types=>ty_decision.

  METHODS from_response
    IMPORTING
      is_request           TYPE zif_rest_provider=>ty_request_descriptor
      iv_key               TYPE zif_rest_cache_types=>ty_key
      is_http              TYPE zif_rest_provider=>ty_response
      iv_now               TYPE timestampl
    RETURNING
      VALUE(rs_refresh)    TYPE zif_rest_cache_types=>ty_refresh
    RAISING
      zcx_rest_protocol_failure.

  METHODS revalidate
    IMPORTING
      is_request           TYPE zif_rest_provider=>ty_request_descriptor
      iv_key               TYPE zif_rest_cache_types=>ty_key
      is_old               TYPE zif_rest_cache_types=>ty_entry
      is_http              TYPE zif_rest_provider=>ty_response
      iv_now               TYPE timestampl
    RETURNING
      VALUE(rs_refresh)    TYPE zif_rest_cache_types=>ty_refresh
    RAISING
      zcx_rest_protocol_failure.
ENDINTERFACE.
```

Regeln:

- Policy ist deterministisch: `iv_now` wird übergeben; keine versteckte Uhr und kein I/O.
- `from_response` verarbeitet erfolgreiche GET-Repräsentationen und ersetzt alte Metadaten/Validatoren vollständig.
- `revalidate` verlangt passende retained Repräsentation + `304`; Payload bleibt, Metadaten/Deadlines werden neu berechnet.
- Typische `reason`-Werte: `MISS`, `CONTEXT_MISMATCH`, `SCHEMA_MISMATCH`, `LAYER_FORBIDDEN`, `REVALIDATION_REQUIRED`, `EXPIRED`, `FRESH`.
- HTTP-Restriktionen (`no-store`, `no-cache`, `private`, Regeln zu authentifizierten Shared-Responses) gelten immer; Security-Scope-Keying hebt sie nicht auf.
- Diese Doku liefert **keine** pseudo-vollständige RFC-9111-Implementierung, sondern nur den Vertrag für eine spätere Produktionsklasse.

---

## Refresh-Algorithmus und Transaktionsgrenzen

### Wichtige Leitplanken

- Refresh läuft als **Cache-only Application Phase vor Business-Änderungen**.
- Facade und Provider committen/rollbacken **nie** Business-Transaktionen.
- Die Refresh-Unit nutzt einen injizierten Transaction-Adapter (`zif_rest_cache_tx`) nur für den Cache-Teil.
- Methoden-/RFC-Aufruf ist **keine** automatische Transaktionsisolation.
- Implizite Commit-Grenzen sind zu berücksichtigen.
- Cache-Miss in beliebigen laufenden Business-LUWs ist im Initialmodell **nicht unterstützt**.

### Ablauf (vereinfacht)

```text
Client
  -> Facade: get(Descriptor, CallerContext)
  -> Facade: Scope validieren
  -> SessionStore: read + Policy.can_reuse(..., layer='S', now)
  -> PersistentStore: read + Policy.can_reuse(..., layer='P', now)
  -> RefreshUnit: refresh_if_needed
      -> LockAdapter: acquire_exclusive(key, timeout)
      -> PersistentStore: reread_unbuffered(key)
      -> Policy.can_reuse(..., layer='P', now)
      -> [falls jetzt wiederverwendbar] return (ohne Provider, ohne Commit)
      -> Provider: fetch_get(Descriptor, optional validator)
      -> [200] Policy.from_response(...)
      -> [304+payload] Policy.revalidate(...)
      -> [304 ohne payload] genau 1x ohne Validator retry
      -> [andere Status] zcx_rest_http_status_failure
      -> PersistentStore: stage upsert/delete gemäß Policy
      -> CacheTx: commit_cache( ) / bei Fehler rollback_cache( )
      -> LockAdapter: release
  -> Facade: Session publizieren falls erlaubt
  -> return
```

### Lock-/Fehlerregeln

- Exklusive Sperre pro Key mit begrenzter Wartezeit und explizitem Fehler (`zcx_rest_lock_timeout` o.ä.).
- Lock wird bei **allen** Ausgängen freigegeben (Erfolg, Transport-/HTTP-/Protokoll-/Store-Fehler).
- Lock-Ownership muss Commit/Rollback überleben; Scope explizit festlegen.
- Release-Fehler dürfen Originalfehler nicht maskieren; Release-Fehler werden separat geloggt.
- `rollback_cache` darf per Vertrag keinen ursprünglichen Fehler maskieren.

### Transaktions-Phasen-Tabelle

| Phase | Erlaubt | Nicht erlaubt |
|---|---|---|
| Vor Refresh (Cache-only) | Reads, Locking vorbereiten | Business-Daten ändern |
| HTTP-Phase | Provider-Aufruf, Status/Metadaten auswerten | Cache-SQL starten |
| Cache-SQL-Phase | direkte synchrone SQL-Operationen (Upsert/Delete) | Update-Task-basiertes Schreiben |
| Abschluss | `zif_rest_cache_tx->commit_cache( )`/`rollback_cache( )`, dann Unlock | Business-COMMIT/ROLLBACK beeinflussen |

---

## Clock-Seam und Fake (deterministische Zeit)

```abap
INTERFACE zif_rest_clock PUBLIC.
  METHODS now
    RETURNING VALUE(rv_now) TYPE timestampl.
ENDINTERFACE.

CLASS lcl_clock_fake DEFINITION FINAL.
  PUBLIC SECTION.
    INTERFACES zif_rest_clock.
    DATA current_time TYPE timestampl.
ENDCLASS.

CLASS lcl_clock_fake IMPLEMENTATION.
  METHOD zif_rest_clock~now.
    rv_now = current_time.
  ENDMETHOD.
ENDCLASS.
```

- ABAP-Unit-Zeitpunkte sind fix (kein `WAIT UP TO`, keine Sleeps).
- Produktionscode nutzt release-passende Timestamp-APIs für Arithmetik; keine „Sekunden als Zahl auf Timestamp addieren“-Abkürzung.

## Transaction-Adapter und Spy

```abap
INTERFACE zif_rest_cache_tx PUBLIC.
  METHODS commit_cache RAISING zcx_rest_cache_write.
  METHODS rollback_cache.
ENDINTERFACE.

CLASS lcl_tx_spy DEFINITION FINAL.
  PUBLIC SECTION.
    INTERFACES zif_rest_cache_tx.
    DATA commits   TYPE i READ-ONLY.
    DATA rollbacks TYPE i READ-ONLY.
    DATA fail_commit TYPE abap_bool.
ENDCLASS.

CLASS lcl_tx_spy IMPLEMENTATION.
  METHOD zif_rest_cache_tx~commit_cache.
    commits = commits + 1.
    IF fail_commit = abap_true.
      RAISE EXCEPTION TYPE zcx_rest_cache_write.
    ENDIF.
  ENDMETHOD.

  METHOD zif_rest_cache_tx~rollback_cache.
    rollbacks = rollbacks + 1.
  ENDMETHOD.
ENDCLASS.
```

- Diese Seam ersetzt nur hart codierte Transaktionsstatements im Cache-Refresh-Teil.
- Sie ist **keine** LUW-Isolation und lockert die Cache-only-Phase nicht.
- Spy rollt In-Memory-Store-Inhalte nicht automatisch zurück.

Transaktionaler Store-Fake (konzeptionell): gemeinsames Backing-Objekt, `read` auf committed Snapshot, `write_or_delete` staged Journal, `commit_cache` appliziert atomar + leert Journal, `rollback_cache` verwirft Journal. Bei simuliertem Pre-Commit-Fehler bleibt committed Snapshot unverändert; unsicherer Commit ist damit nicht „rückholbar“.

## Scripted Provider-Fake für ABAP Unit

```abap
CLASS lcl_provider_fake DEFINITION FINAL.
  PUBLIC SECTION.
    INTERFACES zif_rest_provider.

    TYPES: BEGIN OF ty_step,
             response       TYPE zif_rest_provider=>ty_response,
             transport_fail TYPE abap_bool,
           END OF ty_step,
           ty_steps TYPE STANDARD TABLE OF ty_step WITH EMPTY KEY.

    TYPES: BEGIN OF ty_call,
             request    TYPE zif_rest_provider=>ty_request_descriptor,
             validator  TYPE zif_rest_provider=>ty_validator,
           END OF ty_call,
           ty_calls TYPE STANDARD TABLE OF ty_call WITH EMPTY KEY.

    METHODS constructor IMPORTING it_steps TYPE ty_steps.
    METHODS get_calls RETURNING VALUE(rt_calls) TYPE ty_calls.
  PRIVATE SECTION.
    DATA mt_steps TYPE ty_steps.
    DATA mt_calls TYPE ty_calls.
ENDCLASS.
```

Verhaltensvertrag für `fetch_get`:

- Request + Validatoren als Call protokollieren (`get_calls` read-only verwenden).
- Nächsten Step konsumieren; bei zusätzlichem unerwartetem Call ABAP-Unit-Assertion auslösen.
- `transport_fail = abap_true`: `zcx_rest_transport_failure` raisen.
- Sonst den gescripteten HTTP-Response zurückgeben.

Minimales Script-Beispiel:

1. `200`, Payload vorhanden, `ETag: "v1"`, `Cache-Control: max-age=60`
2. danach `304`, `ETag: "v1"`, `Cache-Control: max-age=120`

Hinweis: Dieses Minimal-Script ist **kein** vollständiges Freshness-Fixture; deterministische Tests für Freshness brauchen konsistente `Date`/`Age`/`sent_at`/`received_at`. Konstruktor-/Exception-Signaturen sind Annahmen für die Doku und nicht als Produktionsabhängigkeit zu verstehen.

---

## HTTP-Caching-Policy (Initialmodell)

- `no-store`: weder Session noch persistent speichern; konservativ vorhandenen passenden Eintrag entfernen.
- `private`: keine cross-user Persistenz-Publikation.
- `no-cache`: vor Wiederverwendung revalidieren.
- Authentifizierte Shared-Responses nur gemäß RFC-9111-Regeln wiederverwenden; Security-Scope-Keying allein reicht nicht.
- `Vary`: relevante Request-Header-Werte in Key aufnehmen oder konservativ Caching ablehnen.
- Keine pauschale Negative-Caching-Strategie im Initialmodell.
- Auth-/Server-Fehler nicht als normale Daten cachen.
- Initial fokussiert auf erfolgreiche GET-Caching-Pfade; Retry/Backoff separat und begrenzt spezifizieren.

---

## Konsistenz- und Staleness-Aussagen

- Persistenter Store startet **ungepuffert** (keine blanket DDIC-Buffering-Empfehlung).
- Es gibt **keine** Aussage über sofortige serverweite Invalidation.
- Session-Kopien dürfen innerhalb expliziter Grenzen stale bleiben.
- Falls sofortige Invalidation nötig ist: separates Versions-/Invalidierungsverfahren als Folgearbeit definieren.
- Keine generische Pufferung per Hash-Substring als Standardannahme.

---

## Akzeptanz-Test-Skeletons (ABAP Unit, Policy)

> Für die **zukünftige reale** Produktionsklasse der Policy (nicht für einen Stub in dieser Doku).

```abap
CLASS ltc_cache_policy DEFINITION FINAL FOR TESTING
  DURATION SHORT
  RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    CONSTANTS c_now TYPE timestampl VALUE '20260925120000.0000000'.
    "fresh_until = 12:01, retain_until = 12:10
    METHODS empty_payload_is_valid      FOR TESTING.
    METHODS expiry_boundary_is_stale    FOR TESTING.
    METHODS scope_mismatch_is_miss      FOR TESTING.
ENDCLASS.
```

Fixture-Vorgaben: Destination `Z_TEST`, Client `100`, Scope `scope-a`, Key `test-key`, `schema_version = 1`, beide Layer erlaubt.

- `empty_payload_is_valid`: `found = abap_true`, leeres `payload_x` ist gültig; erwartetes `reusable = abap_true`.
- `expiry_boundary_is_stale`: wenn Deadline exakt `iv_now` entspricht, dann `reusable = abap_false`.
- `scope_mismatch_is_miss`: gespeicherter Scope `scope-a`, Anfrage `scope-b` => `reusable = abap_false`.

Zeitübergabe in Tests konsistent halten: entweder explizit `iv_now` setzen oder überall die Fake-Clock injizieren.

## Facade-/Refresh-Matrix und Event-Recorder

Pflichtfälle:

- Session-Hit => **kein** Persistent-Read, **kein** Provider, **kein** Commit.
- Persistent-Hit => Promotion in Session ohne Deadline-Verlängerung.
- Re-Read nach Lock erkennt konkurrierenden Refresh => kein Provider.
- `200` ohne ETag entfernt alte Validatoren.
- `304` behält Body und aktualisiert Metadaten.
- Fehlende Repräsentation => ohne Validator anfragen.
- Unbrauchbares `304` => genau ein unconditional Retry, dann Protokollfehler.
- `no-store` => keine Befüllung beider Layer; konservatives Entfernen passender alter Einträge.
- HTTP-/Transport-Fehler => keine staged Mutation, Lock frei.
- Store-Fehler => Rollback angefordert, committed Snapshot unverändert.
- Lock-Akquise-Fehler => kein Provider/Write/Release nicht besessener Locks.
- Scope-/Repräsentations-Mismatch => kein Reuse.

Empfohlene Eventsequenzen (shared Event-Recorder):

- Erfolgspfad: `acquire -> reread -> fetch -> stage -> commit -> release`
- Write-Fehlerpfad: `acquire -> reread -> fetch -> stage-failure -> rollback -> release`

Keine redundanten Commits bei „zweiter Read nach Lock bereits Hit“.

Testaufteilung:

- Policy-Unit-Tests: reale Policy-Klasse.
- Facade-Unit-Tests: scripted Policy-Stub ist zulässig.
- Kombinierte Tests: reale Policy + Fake-Transport/Fake-Stores.

Simulierte Interleavings sind kein Beweis für echte Lock-Ownership oder serverübergreifende Sichtbarkeit; dafür bleiben SAP-Integrations-/Systemtests erforderlich.

### Observability (kurz)

- Metriken: Layer-Hit-Rate, Refresh-Dauer, Lock-Wartezeit, Fehlerraten nach Kategorie.
- Betriebsgrenzen: Retention-Cleanup, maximale Cachegröße, maximale Payloadgröße.

---

## Offene plattformspezifische Punkte

1. Konkrete ABAP-Release-Verfügbarkeit der verwendeten Zeitstempel-APIs/Typen (`TIMESTAMPL`, Zeitdifferenzfunktionen) prüfen.
2. Lock-Scope/Owner so festlegen, dass expliziter Commit/Rollback im Cache-Teil den Lock nicht ungewollt freigibt.
3. Falls später „valid response without cache write“ als Fallback gewünscht ist: unklare Commit-/Invalidierungssemantik explizit modellieren.

---

## Referenzen (primäre Quellen)

> Hinweis: Externe Seiten waren in dieser Laufumgebung nicht live abrufbar; URLs sind als maßgebliche Primärquellen angegeben.

1. RFC 9111 – *HTTP Caching*: https://www.rfc-editor.org/rfc/rfc9111.html
2. SAP ABAP Keyword Documentation – `COMMIT WORK`: https://help.sap.com/doc/abapdocu_latest_index_htm/latest/en-US/ABAPCOMMIT.html
3. SAP ABAP Keyword Documentation – `ROLLBACK WORK`: https://help.sap.com/doc/abapdocu_latest_index_htm/latest/en-US/ABAPROLLBACK.html
4. SAP ABAP Keyword Documentation – SAP Locks / ENQUEUE-Konzept: https://help.sap.com/doc/abapdocu_latest_index_htm/latest/en-US/ABENSAP_LOCK.html
