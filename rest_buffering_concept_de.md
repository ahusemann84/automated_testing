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
| `zif_rest_policy` | Cachebarkeit/Freshness/Retention/`Vary`/`no-store` usw. | Keine Transaktionssteuerung |

---

## Provider-Vertrag (zustandslos)

> **Konzept-Skeleton**: nicht aktivierungsfertig.

```abap
INTERFACE zif_rest_provider PUBLIC.
  TYPES: BEGIN OF ty_request_descriptor,
           destination_id         TYPE string,
           sap_client             TYPE mandt,
           resource_path          TYPE string,
           normalized_query       TYPE string,
           accept_language        TYPE sylangu,
           trusted_security_scope TYPE string,
         END OF ty_request_descriptor.

  TYPES: BEGIN OF ty_validator,
           etag          TYPE string,
           last_modified TYPE string,
         END OF ty_validator.

  TYPES: BEGIN OF ty_header,
           name  TYPE string,
           value TYPE string,
         END OF ty_header,
         ty_headers TYPE STANDARD TABLE OF ty_header WITH EMPTY KEY.

  TYPES: BEGIN OF ty_response,
           http_status           TYPE i,
           payload_x             TYPE xstring,
           headers               TYPE ty_headers, "Duplikate bleiben erhalten
           sent_at_utc           TYPE utclong,
           received_at_utc       TYPE utclong,
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
- Request-Descriptor enthält echte Request-Dimensionen; ein Cache-Hash allein ist unzulässig.

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

## Datenvertrag für Cache-Einträge und Refresh-Ergebnis

> **Konzept-Skeleton**: unterstützende Typen/Enums/Exceptions sind zu definieren.

```abap
TYPES: BEGIN OF ty_cache_entry,
         cache_key                 TYPE string,
         found                     TYPE abap_bool,
         payload_x                 TYPE xstring,
         etag                      TYPE string,
         last_modified             TYPE string,
         http_status               TYPE i,
         cache_control_raw         TYPE string,
         vary_raw                  TYPE string,
         validated_at_utc          TYPE utclong,
         fresh_until_utc           TYPE utclong,
         retain_until_utc          TYPE utclong,
         representation_version    TYPE string,
       END OF ty_cache_entry.

TYPES: BEGIN OF ty_refresh_result,
         entry               TYPE ty_cache_entry,
         session_allowed     TYPE abap_bool,
         persistent_allowed  TYPE abap_bool,
       END OF ty_refresh_result.
```

Semantik:

- `200`: Payload + Validatoren + Metadaten werden vollständig ersetzt; fehlende frühere Validatoren werden gelöscht.
- `304`: normaler Ergebnisfall, Payload bleibt erhalten, Metadaten/Freshness werden neu berechnet.
- `304` ohne vorhandene Payload: genau **ein** Retry ohne Validatoren; zweite unbrauchbare `304` => Protokollfehler.
- `retain_until_utc` ist von Freshness getrennt (Revalidation mit abgelaufener Repräsentation möglich).
- ETags werden nicht gekürzt.

---

## Refresh-Algorithmus und Transaktionsgrenzen

### Wichtige Leitplanken

- Refresh läuft als **Cache-only Application Phase vor Business-Änderungen**.
- Facade und Provider committen/rollbacken **nie** Business-Transaktionen.
- Die Refresh-Unit besitzt Lock + Cache-SQL + `COMMIT WORK`/`ROLLBACK WORK` für den Cache-Teil.
- Methoden-/RFC-Aufruf ist **keine** automatische Transaktionsisolation.
- Implizite Commit-Grenzen sind zu berücksichtigen.
- Cache-Miss in beliebigen laufenden Business-LUWs ist im Initialmodell **nicht unterstützt**.

### Ablauf (vereinfacht)

```text
Client
  -> Facade: get(Descriptor, CallerContext)
  -> Facade: Scope validieren
  -> SessionStore: read
  -> PersistentStore: read
  -> RefreshUnit: refresh_if_needed
      -> LockAdapter: acquire_exclusive(key, timeout)
      -> PersistentStore: reread_unbuffered(key)
      -> [falls jetzt wiederverwendbar] return
      -> Provider: fetch_get(Descriptor, optional validator)
      -> [200] neues Entry bauen
      -> [304+payload] Payload behalten, Metadaten mergen
      -> [304 ohne payload] genau 1x ohne Validator retry
      -> [andere Status] zcx_rest_http_status_failure
      -> PersistentStore: stage upsert/delete gemäß Policy
      -> COMMIT WORK (nur Cache-Teil)
      -> LockAdapter: release
  -> Facade: Session publizieren falls erlaubt
  -> return
```

### Lock-/Fehlerregeln

- Exklusive Sperre pro Key mit begrenzter Wartezeit und explizitem Fehler (`zcx_rest_lock_timeout` o.ä.).
- Lock wird bei **allen** Ausgängen freigegeben (Erfolg, Transport-/HTTP-/Protokoll-/Store-Fehler).
- Lock-Ownership muss Commit/Rollback überleben; Scope explizit festlegen.
- Release-Fehler dürfen Originalfehler nicht maskieren; Release-Fehler werden separat geloggt.

### Transaktions-Phasen-Tabelle

| Phase | Erlaubt | Nicht erlaubt |
|---|---|---|
| Vor Refresh (Cache-only) | Reads, Locking vorbereiten | Business-Daten ändern |
| HTTP-Phase | Provider-Aufruf, Status/Metadaten auswerten | Cache-SQL starten |
| Cache-SQL-Phase | direkte synchrone SQL-Operationen (Upsert/Delete) | Update-Task-basiertes Schreiben |
| Abschluss | `COMMIT WORK`/`ROLLBACK WORK` der Cache-Änderungen, dann Unlock | Business-COMMIT/ROLLBACK beeinflussen |

---

## HTTP-Caching-Policy (Initialmodell)

- `no-store`: weder Session noch persistent speichern; konservativ vorhandenen passenden Eintrag entfernen.
- `private`: keine cross-user Persistenz-Publikation.
- `no-cache`: vor Wiederverwendung revalidieren.
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

## Test-Checkliste (fachlich)

- [ ] Session-Hit ruft Provider nicht auf
- [ ] Persistent-Hit ruft Provider nicht auf
- [ ] Re-Read nach Lock verhindert Doppel-Refresh
- [ ] `200` ersetzt Payload/Validatoren vollständig
- [ ] `304` behält Payload und aktualisiert Metadaten/Freshness
- [ ] `304` ohne Payload triggert genau einen unbedingten Retry
- [ ] `no-store` speichert in keiner Schicht
- [ ] Transport-/HTTP-/Protokoll-/Store-Fehler geben Lock frei
- [ ] Store-Fehler nach SQL-Start führt zu Rollback + Propagation
- [ ] Promotion Session<-Persistent erhält absolute Expiry
- [ ] Isolation nach Destination/Client/Security-Scope
- [ ] Refresh vor Business-Änderungen (Cache-only-Phase)
- [ ] Explizit begrenzte Staleness über Sessions dokumentiert

### Observability (kurz)

- Metriken: Layer-Hit-Rate, Refresh-Dauer, Lock-Wartezeit, Fehlerraten nach Kategorie.
- Betriebsgrenzen: Retention-Cleanup, maximale Cachegröße, maximale Payloadgröße.

---

## Offene plattformspezifische Punkte

1. Konkrete ABAP-Release-Verfügbarkeit der verwendeten Zeitstempel-APIs/Typen (`utclong` etc.) prüfen.
2. Lock-Scope/Owner so festlegen, dass expliziter Commit/Rollback im Cache-Teil den Lock nicht ungewollt freigibt.
3. Falls später „valid response without cache write“ als Fallback gewünscht ist: unklare Commit-/Invalidierungssemantik explizit modellieren.

---

## Referenzen (primäre Quellen)

> Hinweis: Externe Seiten waren in dieser Laufumgebung nicht live abrufbar; URLs sind als maßgebliche Primärquellen angegeben.

1. RFC 9111 – *HTTP Caching*: https://www.rfc-editor.org/rfc/rfc9111.html
2. SAP ABAP Keyword Documentation – `COMMIT WORK`: https://help.sap.com/doc/abapdocu_latest_index_htm/latest/en-US/ABAPCOMMIT.html
3. SAP ABAP Keyword Documentation – `ROLLBACK WORK`: https://help.sap.com/doc/abapdocu_latest_index_htm/latest/en-US/ABAPROLLBACK.html
4. SAP ABAP Keyword Documentation – SAP Locks / ENQUEUE-Konzept: https://help.sap.com/doc/abapdocu_latest_index_htm/latest/en-US/ABENSAP_LOCK.html
