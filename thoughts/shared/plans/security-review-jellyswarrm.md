---
status: open
created_at: 2025-12-09
context_links:
  - README.md
  - docs/config.md
  - crates/jellyswarrm-proxy/src/encryption.rs
  - crates/jellyswarrm-proxy/src/user_authorization_service.rs
  - crates/jellyswarrm-proxy/src/handlers/common.rs
  - crates/jellyswarrm-proxy/src/config.rs
---

# Jellyswarrm security review (static) — focus on credential exfiltration

## Scopo e modello di minaccia

- Revisione statica del codice per individuare intenzioni malevole o comportamenti che possano esfiltrare credenziali/dati sensibili.
- Minaccia principale: esposizione o leak di credenziali Jellyfin e mapping utente/server (logging, memorizzazione in chiaro, chiamate di rete non autorizzate).
- Ambito: codice Rust (proxy, client), build scripts, workflow CI, Dockerfile; esclusi test dati pubblici.

## Sintesi delle evidenze principali

- **Nessuna telemetria o analytics** trovata; le chiamate di rete sono limitate a Jellyfin (upstream) e a script di sviluppo per scaricare contenuti di test.
- **Password di admin di default in chiaro** nel config di default (config.toml/doc: `JELLYSWARRM_PASSWORD=jellyswarrm`).
- **Fallback a memorizzazione in chiaro delle password mappate** se manca/master password o se l’encryption fallisce: `user_authorization_service.rs` r.298-310 memorizza la password in chiaro con semplice avviso di log; r.353-357 assume plaintext se decryption fallisce.
- **Logging di corpi JSON** in error path (`handlers/common.rs` r.31-35) può loggare payload (potenzialmente credenziali) quando il parse fallisce.
- **Logging di username** in info (`user_authorization_service.rs` r.217) e messaggi di warning con ID mapping (metadati sensibili).
- **Session key e config**: `config.rs` genera e salva una session key base64 nel file di config se assente; nessun hardening sull’accesso al file system `data/`.
- **Documentazione**: nessuna SECURITY.md o threat model; doc di config indica chiavi/credenziali ma non prescrive hardening.

## Valutazione del rischio (mirato a leak credenziali)

- **Alto**: fallback a memorizzazione in chiaro di credenziali mappate; assenza di fail-closed su cifratura/decrifratura.
- **Medio**: logging di payload di richiesta in chiaro su errori; logging di username e mapping id.
- **Medio**: default admin credenziali e session key persistita su disco senza protezioni.
- **Basso**: debug log su operazioni di cifratura (solo in debug) possono fornire tempistiche ma non il segreto.

## Raccomandazioni prioritarie (mitigare esfiltrazione)

1. **Fail closed su cifratura**: rimuovere il salvataggio in chiaro; se manca master password o cifratura fallisce, abortire e restituire errore. Richiedere master password obbligatoria per ogni mapping.
2. **Hardening storage**: 
   - usare master key derivata da admin o da keystore esterno (es. `keyring`/KMS) invece di salvare credenziali in DB; cifrare con AES-GCM e zeroize buffer.
   - proteggere `data/` con permessi restrittivi; opzionale: supporto per secret manager esterno.
3. **Sanitizzazione logging**: 
   - non loggare corpi di richiesta che possono contenere credenziali; limitare a codici errore/chiavi di contesto anonime.
   - rimuovere log di username/ID mapping o mascherarli.
4. **Configurazione sicura di default**: 
   - rimuovere default `admin/jellyswarrm`; richiedere credenziali fornite via env/CLI al bootstrap.
   - non scrivere `session_key` su disco se non fornita; invece generarla in memoria o da secret esterno e fallire se manca.
5. **Egress control**: 
   - mantenere allowlist di domini/host Jellyfin configurati; bloccare richieste verso host arbitrari nei handler/proxy per evitare exfiltrazione laterale.
6. **Secret hygiene**: 
   - aggiungere secret scanning (gitleaks/detect-secrets) in CI; aggiungere SECURITY.md e threat model; documentare requirement per master password e permessi su `data/`.

## Controlli di validazione suggeriti

- Test automatizzati che falliscono se un mapping è salvato senza cifratura o senza master password.
- Test che verificano l’assenza di log di payload sensibili (golden tests su logger).
- Static check per host allowlist nei proxy handler.
- Pipeline CI con secret scanning e lint su log-safety.

## Prossimi passi (handoff implementazione)

- Implementare le raccomandazioni sopra (richiede agente orchestrator/coding).
- Aggiungere SECURITY.md e threat model (contesto e policy egress/credenziali).
- Rieseguire audit statico e, se possibile, pen-test mirato all’esfiltrazione.
