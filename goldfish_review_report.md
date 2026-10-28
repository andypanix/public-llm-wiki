# Goldfish Annotated Review Report

**Target:** `/root/development/wiki-site/docs/designs/wiki-site.md` (FeatureRange 1..607; alcuni interventi toccano 608..969 per non lasciare il piano tecnico incoerente con la specifica)

**Fonte dei finding:** `critic_report.md` (Stage 7, 46 consolidati) più i temi già emersi in `zoom_out_report.md` (Stage 5) e `comprehension_report.md` (Stage 6).

**Status: resolved** — 42 voci `PROPOSE FIX` applicate al documento, 4 `CONTEST` non applicate. Vedi "Applicato" in fondo per gli interventi extra e per i residui rimasti aperti.

## Sintesi delle decisioni

| # | Sev | Tema | Verdetto |
| --- | --- | --- | --- |
| B1 | High | Pipeline di fetch non eseguibile | PROPOSE FIX |
| B2 | High | Rollback annullato dal ciclo successivo | PROPOSE FIX |
| B3 | High | Contraddizioni sullo stato del sito e mount dei file di stato | PROPOSE FIX |
| B4 | High | Manca la sezione Rischi e mitigazioni | PROPOSE FIX |
| B5 | High | Contratto del modello incompleto (`context`, `sources`) e riga 57 falsa | PROPOSE FIX |
| B6 | High | O11 senza bind, firewall e verifica | PROPOSE FIX |
| B7 | High | `failed.txt` permanente, revert che congela il sito | PROPOSE FIX |
| B8 | High | Manca l'alternativa Obsidian mobile e i criteri di scelta | PROPOSE FIX |
| M1 | Medium | Segreti sull'host senza inventario né rotazione | PROPOSE FIX |
| M2 | Medium | Verifica di O1 non progettata, grandezze mescolate | PROPOSE FIX |
| M3 | Medium | 858 link assoluti: numero non riproducibile | PROPOSE FIX (con misura nuova) |
| M4 | Medium | 195 file: la ripartizione non chiude | PROPOSE FIX (con misura nuova) |
| M5 | Medium | `export` rotto su `main` blocca tutti gli SHA | PROPOSE FIX |
| M6 | Medium | Test dell'export tautologico | PROPOSE FIX |
| M7 | Medium | "Cinque domande" con sei voci, D5 duplicata | PROPOSE FIX |
| M8 | Medium | Completezza dichiarata mentre due decisioni restano aperte | PROPOSE FIX |
| M9 | Medium | Obiettivi non misurabili (3 su 13 hanno `Verifica:`) | PROPOSE FIX |
| M10 | Medium | O3 (senza JS) senza verifica | PROPOSE FIX |
| M11 | Medium | CUJ 3 promette la cronologia del browser | PROPOSE FIX |
| M12 | Medium | "Quattro pezzi, tre non nostri" contro cinque blocchi | PROPOSE FIX |
| M13 | Medium | "Generatore" indica due oggetti | PROPOSE FIX |
| M14 | Medium | Cleanup e spazio libero non previsti | PROPOSE FIX |
| M15 | Medium | Immagine pinnata senza cadenza | PROPOSE FIX |
| M16 | Medium | "I due hook" contro tre moduli | PROPOSE FIX |
| M17 | Medium | Alternativa I: affermazione falsa sul linter | PROPOSE FIX |
| M18 | Medium | Misure di ricerca incompatibili (20 contro 80) | PROPOSE FIX (con misura nuova) |
| M19 | Medium | Budget dell'indice misurato su un artefatto diverso | PROPOSE FIX (misura eseguita: il budget tiene) |
| M20 | Medium | Collaudo sull'IP che aggira D5; rimando a O2 inesistente | PROPOSE FIX |
| N1-N5 | Low | Identificatori, righe orizzontali, 1.492/1.507, colonna "Come è misurato", 973 voci | PROPOSE FIX (N2 CONTEST) |
| N6 | Low | Registri diversi negli H3 | CONTEST |
| N7-N13 | Low | Partizione 6+6, convenzioni URL, vincolo radice, CUJ↔obiettivi, non-obiettivi, Risorse, rimandi | PROPOSE FIX |
| N14-N17 | Low | Metafore, lessico, affermazioni non misurate, ridondanze | PROPOSE FIX (parziale: N14 e N17 solo dove la ripetizione è provata) |
| N18 | Low | Dettagli di contratto aperti (SHA breve, `schema_version`, peso pagina, FdV senza fonte) | PROPOSE FIX |

Misure eseguite dall'Elephant per questa revisione (prototipo `~/.hermes/cache/scratch/proto-mkdocs`, albero `3a1495a`):
archivio = **215 file**; indice di ricerca del prototipo = **973 voci, 730.004 B, 216.192 B con gzip -9**; con testo e titolo ripiegati = **767.242 B, 219.053 B con gzip -9**; link con percorso assoluto nel contenuto = **2.350 occorrenze, 187 URL distinti (125 pagine, 62 raw)**; nessun URL `/raw/` nell'indice.

---

## High

### B1 — La pipeline di fetch non è eseguibile [`egm-design`] — Severity: High
* **Affected Scope:** 256-279, diagramma riga 257 e svolgimento righe 274-279
* **Discrepancy Description:** `git ls-remote` restituisce ref e object id senza scaricare oggetti; `git archive <sha>` subito dopo non ha nulla da impacchettare. `git archive --remote=` non è disponibile su GitHub.
* **Rationale:** l'intero ciclo (validazione, swap, O1, O8) poggia su quel passo.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** il difetto è verificabile dalla semantica di git e non ha alternative interpretative: due comandi in sequenza non possono funzionare come descritto.
* **Proposed Edit:**
  * Location: riga 257 (diagramma)
  * Change: `│  git ls-remote + git archive di refs/heads/main` → `│  git fetch --depth 1 in un mirror bare locale, poi git archive dello SHA`
  * Location: righe 274-275
  * Change: `Fa `git ls-remote` per lo SHA di `origin/main`, esce senza fare nulla` → `Aggiorna il mirror bare in `/srv/wiki-site/mirror` (`git fetch --depth 1 origin main`), legge dal mirror lo SHA di `origin/main`, esce senza fare nulla`
  * Location: dopo la riga 279, nuovo capoverso
  * Change: aggiungere: «Il mirror serve perché `git ls-remote` non scarica oggetti: `git archive` lavora solo su un repository locale, e per estrarre un archivio da remoto servirebbe il servizio `upload-archive`, che GitHub non espone. Con `--depth 1` il rollback è limitato agli SHA ancora presenti fra le release.»

### B2 — Il rollback viene annullato dal ciclo successivo [`egm-design`] — Severity: High
* **Affected Scope:** 138-141 (O10), 274-276, 387-394, 571-573
* **Discrepancy Description:** dopo un rollback a uno SHA precedente, il ciclo successivo trova `origin_main_sha` diverso da `published_sha` e ripubblica il commit appena scartato.
* **Rationale:** O10 promette un ritorno a uno SHA precedente che dura 15 minuti.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** manca uno stato che esprima la volontà dell'operatore; è una lacuna del contratto di `status.json`, non un dettaglio implementativo.
* **Proposed Edit:**
  * Location: O10, dopo la riga 141
  * Change: aggiungere: «Il rollback scrive `pinned_sha` in `status.json`; finché il pin è valorizzato e diverso da `origin_main_sha`, `publish.sh` non pubblica. Un nuovo merge e il comando `publish.sh --unpin` lo azzerano. Verifica: tre cicli consecutivi dopo un rollback non cambiano `published_sha`.»
  * Location: contratto `status.json`, riga 392
  * Change: `"releases": ["3a1495a", "154c360"]` → `"pinned_sha": null, "releases": ["3a1495a", "154c360"]`
  * Location: elenco di `publish.sh`, riga 782
  * Change: `uscita immediata se lo SHA è pubblicato o presente in `failed.txt`` → `uscita immediata se lo SHA è pubblicato, pin attivo o presente in `failed.txt``

### B3 — Contraddizioni sullo stato del sito [`egm-design`, `egm-structural`, `egm-style`] — Severity: High
* **Affected Scope:** 131-137 (O8, O9), 207-213 (CUJ 4), 856-859 (mount)
* **Discrepancy Description:** la home è deterministica ma CUJ 4 le attribuisce dati per-ciclo; O9 dichiara quattro informazioni e CUJ 4 ne chiede cinque; `status.html` e `status.json` sono montati come singoli file mentre `status.py` li scrive con `rename`.
* **Rationale:** il mount di un file lega l'inode: dopo il primo `rename` nginx serve la versione precedente. È il guasto silenzioso che O9 vuole impedire.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** nessuno dei tre punti si risolve nel testo senza una decisione esplicita su quale pagina mostra cosa.
* **Proposed Edit:**
  * Location: O8, riga 132-133
  * Change: `Entro 60 minuti da un merge la versione servita è quella del nuovo commit, e la home mostra SHA pubblicato e ora del build.` → `Entro 60 minuti da un merge la versione servita è quella del nuovo commit. La home mostra lo SHA del proprio build — costante per SHA, quindi deterministico — e collega `/status.html`; ogni dato che cambia a ogni ciclo vive solo lì.`
  * Location: O9, riga 134-137
  * Change: `Esistono quattro informazioni sempre leggibili: SHA costruito, ora dell'ultimo fetch, ora dell'ultimo tentativo e suo esito.` → `Esistono cinque informazioni sempre leggibili in `/status.html`: SHA costruito, SHA su `main` (e quindi la loro differenza), ora dell'ultimo fetch, ora dell'ultimo tentativo e suo esito.`
  * Location: CUJ 4, righe 210-213
  * Change: `si guarda il fondo della home o si apre la pagina di stato.` → `si tocca il link in fondo alla home e si apre `/status.html`.`
  * Location: CUJ 4, riga 211-213
  * Change: `si legge lo SHA pubblicato, l'ora dell'ultimo fetch riuscito, l'ora e l'esito dell'ultimo tentativo, e la differenza fra i due SHA. Il contenuto servito è l'ultimo valido, e non finge di essere fresco.` → `si leggono i due SHA e la loro differenza, l'ora dell'ultimo fetch riuscito, l'ora e l'esito dell'ultimo tentativo. Il contenuto servito è l'ultimo valido, e il ritardo è dichiarato in pagina.`
  * Location: `docker/compose.yml`, riga 856-858
  * Change: `mount in sola lettura di `/srv/wiki-site/current` su `/usr/share/nginx/html` e di `status.html` e `status.json` nella stessa radice` → `mount in sola lettura di `/srv/wiki-site/current` su `/usr/share/nginx/html` e della directory `/srv/wiki-site/status/` su `/usr/share/nginx/html/status/` (una directory, non singoli file: `status.py` scrive con temporaneo più `rename`, e un mount di file resterebbe legato all'inode precedente)`
  * Location: URL, riga 356
  * Change: `/status.html   stato del publisher, scritto fuori dal build` → `/status/status.html   stato del publisher, scritto fuori dal build e servito da una directory montata`

### B4 — Manca la sezione Rischi e mitigazioni [`egm-structural`, `egm-design`] — Severity: High
* **Affected Scope:** fra `## Requisiti non funzionali` (558) e la riga 557
* **Discrepancy Description:** lo standard feature-spec impone una sezione che dichiari modalità di guasto, sicurezza, limiti e mitigazioni. I tre rischi di 65-75 restano senza contropartita e le mitigazioni sono sparse.
* **Rationale:** senza un registro unico non si può verificare che ogni rischio abbia un rimedio.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** la sostanza esiste nel documento; manca il posto dove leggerla insieme.
* **Proposed Edit:**
  * Location: prima della riga 558, nuova sezione
  * Change: inserire:

```text
## Rischi e mitigazioni

| Rischio | Modalità di guasto | Rilevazione | Mitigazione | Verifica |
| --- | --- | --- | --- | --- |
| Credenziale del repository sull'host dove gira l'agente | la chiave scade o viene revocata e il fetch smette di funzionare | `last_attempt_ok: false` in `status.json`, Telegram | deploy key in sola lettura, utente dedicato, `env` 0600, rotazione documentata | revoca simulata: il sito resta su e lo stato lo dice |
| Build fallito per causa transitoria | lo SHA entra nella lista dei falliti e il sito si ferma | contatore `attempts` per SHA | N tentativi con backoff, poi stop dichiarato | tre errori di rete simulati, poi pubblicazione entro 60 minuti |
| Porta del sito raggiungibile oltre LAN e Tailscale | contenuto leggibile da Internet | `ss -ltnp` e `curl` da fuori | bind esplicito più regola `DOCKER-USER` | il `curl` dall'esterno fallisce |
| Disco pieno durante lo swap | release parziale, `status.json` non scrivibile | soglia di spazio libero prima del build | swap solo dopo build e validazione riusciti | build con disco quasi pieno: esce con errore dichiarato |
| Orologio dell'host con fuso o offset errato | confronti "oltre 60 minuti" sbagliati | `timedatectl` | i confronti usano epoch, non stringhe | test con orologio spostato di un'ora |
```

### B5 — Il contratto del modello non copre O5 e O7 [`egm-design`, `egm-structural`] — Severity: High
* **Affected Scope:** 57, 120-130, 311-332, 372-374
* **Discrepancy Description:** `backlinks` non ha la riga di contesto richiesta da O7 e il generatore non può ricalcolarla; `sources` non è nello schema benché O5 la richieda; la riga 57 descrive il linter come se calcolasse tutti i link entranti, mentre conta solo i wikilink.
* **Rationale:** un requisito che il contratto non rappresenta non è implementabile senza violare la regola "nessun ricalcolo nel generatore".
* **AI Suggestion:** PROPOSE FIX
* **Justification:** verificato sul codice: `tools/wiki.py` incrementa `inbound` solo nel ciclo dei wikilink (233-240) e valida `sources` separatamente (213-232).
* **Proposed Edit:**
  * Location: riga 57
  * Change: `mantenere a mano, su 125 pagine, una contabilità di link che oggi il linter già calcola e butta via` → `mantenere a mano, su 125 pagine, una contabilità di link che oggi il linter calcola solo in parte — conta i link entranti dei wikilink, non quelli relativi — e usa solo per individuare le pagine orfane`
  * Location: contratto, riga 327
  * Change: `"backlinks": [{"slug": "design-come-nuovo-codice", "wiki": "software-engineering"}]` → `"backlinks": [{"slug": "design-come-nuovo-codice", "wiki": "software-engineering", "context": "frase che contiene il link, al massimo 120 caratteri"}], "sources": ["raw/articles/rensig-elephants-goldfish-golden-age.md"]`
  * Location: O7, dopo la riga 130
  * Change: aggiungere: `Verifica: una pagina `homelab/` citata solo da `ai-llm/` compare nei suoi backlink, cioè il conteggio non dipende dalla wiki di appartenenza.`

### B6 — O11 senza meccanismo né verifica [`egm-design`] — Severity: High
* **Affected Scope:** 142-143, 267-269, 856-859, 944-946
* **Discrepancy Description:** `8088:80` lega tutte le interfacce; Docker inserisce le proprie regole prima di quelle dell'host; il tailnet con ACL di default è aperto a ogni device; manca la verifica.
* **Rationale:** è l'unico obiettivo di sicurezza del documento e non ha né una riga di configurazione né un criterio.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** il binding esplicito è una riga di configurazione e non cambia il design.
* **Proposed Edit:**
  * Location: O11, righe 142-143
  * Change: `**O11 — Raggiungibile solo da LAN e Tailscale,** senza autenticazione e senza esposizione su Internet.` → `**O11 — Raggiungibile solo da LAN e Tailscale,** senza autenticazione e senza esposizione su Internet. Il binding è dichiarato (`BIND_ADDR:8088:80`, con `BIND_ADDR` pari all'indirizzo LAN dell'host) e il traffico in ingresso sulla porta è bloccato in `DOCKER-USER`, che precede le regole di Docker. Verifica: dall'esterno il `curl` verso l'indirizzo IPv6 dell'host fallisce, e `ss -ltnp` mostra il bind dichiarato.`

### B7 — Un fallimento transitorio blocca lo SHA fino al merge successivo [`egm-design`] — Severity: High
* **Affected Scope:** 140-141 (O10), 274-276, 383-405, 416-419 (D3)
* **Discrepancy Description:** la lista dei falliti è permanente e senza invalidazione; se `main` torna a uno SHA già fallito il sito non si aggiorna più, e lo stato dice "niente da fare".
* **Rationale:** contraddice O8 e il principio dichiarato alle righe 68-71, e il silenzio è proprio il guasto che il documento vuole evitare.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** la correzione non cambia l'architettura: sostituisce un insieme con un contatore e aggiunge una distinzione nei messaggi.
* **Proposed Edit:**
  * Location: O10, riga 140
  * Change: `Lo SHA che fallisce non viene ritentato in loop ogni ora.` → `Uno SHA il cui build fallisce viene ritentato al massimo tre volte, con intervallo crescente; dopo la terza resta nella lista dei falliti e non viene ritentato finché lo SHA di `origin/main` non cambia.`
  * Location: O8, riga 131-132
  * Change: `Nessun comando manuale. Entro 60 minuti da un merge` → `Nessun comando manuale, salvo i fallimenti registrati in `status.json`. Entro 60 minuti da un merge`
  * Location: elenco `publish.sh`, riga 782
  * Change: `uscita immediata se lo SHA è pubblicato o presente in `failed.txt`` → `uscita immediata se lo SHA è pubblicato o se ha esaurito i tentativi in `failed.txt` (una riga per SHA: `sha attempts last_error first_failed_at`)`
  * Location: `notify.py`, riga 835-838
  * Change: aggiungere dopo l'elenco delle transizioni: `Il messaggio distingue "fallito, ritenterò" da "fallito, rinuncio". Un heartbeat assente è l'unico segnale se l'host muore: publisher e watcher tacciono insieme, quindi la sua assenza va notata da una persona o da un secondo host.`

### B8 — Manca l'alternativa a costo zero [`egm-design`] — Severity: High
* **Affected Scope:** 431-554, 152-162
* **Discrepancy Description:** l'app Obsidian mobile con il vault sincronizzato non è valutata, benché sia l'unica alternativa che possa cancellare il progetto; la scelta fra le dodici non è fatta rispetto a criteri dichiarati.
* **Rationale:** la regola delle alternative dello standard non è soddisfatta se manca la più economica.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** si chiude con un paragrafo, e la risposta (nessun URL condivisibile, nessuno stato verificabile, sincronizzazione a pagamento) è coerente con i vincoli già scritti.
* **Proposed Edit:**
  * Location: riga 433-434, prima dell'elenco
  * Change: aggiungere: `Il confronto usa tre criteri, in quest'ordine: fedeltà a `origin/main`, resa mobile del percorso di lettura, costo di manutenzione. Le alternative sotto sono valutate su quelli.`
  * Location: dopo l'ultima alternativa (dopo riga 554), nuova voce
  * Change: inserire:

```text
### Alternativa M: app Obsidian mobile sul vault sincronizzato

* **Pro**: nessun build, nessun servizio; wikilink, backlink, ricerca, frontmatter
  nascosto e resa mobile sono già risolti; il vault è lo stesso che si scrive.
* **Contro**: la sincronizzazione ufficiale è a pagamento; il vault va aperto su
  ogni device; non esiste un URL condivisibile né una versione servita
  verificabile per SHA; il consumo dipende dall'app, non da un artefatto che si
  può ispezionare.
* **Motivo del rifiuto**: il metro del documento (righe 41-46) è un percorso di
  lettura con URL memorizzabili e uno stato del build leggibile. Obsidian mobile
  è il lettore migliore e resta lo strumento di scrittura, ma non dà né l'uno né
  l'altro, e non rende verificabile la distanza da `origin/main`.
```

## Medium

### M1 — Segreti sull'host senza inventario né rotazione [`egm-design`] — Severity: Medium
* **Affected Scope:** 72-75, 703, 924-925
* **Discrepancy Description:** servono due credenziali sull'host; il documento non dice dove vivono, con quali permessi, con quale utente gira `publish.sh`, né come si ruotano.
* **Rationale:** il rischio 3 è mitigato per i container, non per l'host dove girano l'agente e i suoi privilegi.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** è una lacuna di specifica, non una scelta di design da contestare.
* **Proposed Edit:**
  * Location: Tappa 3, dopo la riga 925
  * Change: aggiungere: `Le chiavi in `/etc/wiki-site/env` sono due: `WIKI_REPO` (URL del repository) più la deploy key in sola lettura in `/etc/wiki-site/deploy-key` (`0600`, proprietario `wiki-site`), e `TELEGRAM_TOKEN` con `TELEGRAM_CHAT_ID`. `publish.sh` gira come utente `wiki-site`, senza privilegi; la chiave si ruota dalla pagina delle deploy key di GitHub, e la revoca è uno scenario di collaudo: il sito resta sull'ultima release e `status.json` riporta `last_attempt_ok: false` con l'errore di autenticazione.`

### M2 — Verifica di O1 non progettata [`egm-design`, `egm-structural`, `egm-style`] — Severity: Medium
* **Affected Scope:** 104-107, 226-229, 244-249, 808-819
* **Discrepancy Description:** "una validazione nostra, dopo il build" non è descritta; il criterio mescola 1.507 marcatori e 984 link.
* **Rationale:** un controllo non specificato non è un controllo.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** i tre numeri corretti sono già nel documento: 984 link di pagina, 1.492 apici, zero `[[` residue.
* **Proposed Edit:**
  * Location: O1, righe 105-107
  * Change: `Verifica: build che fallisce su un link irrisolto, più un conteggio dei link generati contro i 1.507 marcatori e i 984 link attesi.` → `Verifica, tre asserzioni separate: 984 link interni generati (804 wikilink più 180 relativi), 1.492 apici emessi (i 15 `general-knowledge` non sono link), zero `[[` residue nell'HTML. Le esegue `validate.py` sull'albero costruito, confrontando l'HTML con il modello.`

### M3 — "858 link assoluti" non è riproducibile [`egm-design`, `egm-structural`, `egm-style`] — Severity: Medium
* **Affected Scope:** 88, 360-362
* **Discrepancy Description:** 858 contro 804 wikilink e 984 link attesi, senza derivazione.
* **Rationale:** il vincolo "servito sulla radice" poggia su questo numero.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** misurato sul prototipo: nel contenuto ci sono 2.350 link con percorso assoluto, 187 URL distinti (125 pagine, 62 raw); 858 non corrisponde a nessuna delle grandezze contate.
* **Proposed Edit:**
  * Location: riga 360-362
  * Change: `Gli 858 link assoluti che l'`obsidian-bridge` genera dai wikilink funzionano solo perché il sito è servito sulla radice del nome: va scritto, perché è un vincolo, non un dettaglio.` → `I link che l'`obsidian-bridge` genera dai wikilink hanno percorso assoluto: nel prototipo il contenuto contiene 2.350 link con percorso assoluto, 187 URL distinti (125 pagine, 62 raw). Funzionano solo se il sito è servito su `/`: va scritto, perché è un vincolo, non un dettaglio.`

### M4 — Il conteggio dei file non chiude [`egm-design`, `egm-structural`] — Severity: Medium
* **Affected Scope:** 220-221, 584, 940-941
* **Discrepancy Description:** "195 file di cui 125 pagine, 62 raw, 7 indici" somma a 194; la ripartizione esclude i `SCHEMA.md` che il linter usa per riconoscere le wiki.
* **Rationale:** è la prima riga della tabella che deve provare che il piano è misurato.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** contato con `git ls-tree -r 3a1495a`: 215 file, ripartizione completa disponibile.
* **Proposed Edit:**
  * Location: riga 220-221
  * Change: `(3a1495a, 195 file di cui 125 pagine, 62 raw, 7 indici)` → `(3a1495a: 215 file — 125 pagine curate, 62 raw, 7 `index.md`, 7 `SCHEMA.md`, 7 `log.md`, 7 file di servizio fra `.github/`, `tools/wiki.py`, `README.md` e altro)`
  * Location: riga 584 e 940-941
  * Change: `195 file costruiti` → `187 file pubblicati (125 pagine più 62 raw)`

### M5 — Un `export` rotto su `main` blocca tutti gli SHA [`egm-design`] — Severity: Medium
* **Affected Scope:** 312-315, 336-347, 726-732
* **Discrepancy Description:** il tool vive nel repository che l'agente scrive; un commit che lo rompe fa fallire ogni build successivo; non è detto cosa succede con uno `schema_version` sconosciuto.
* **Rationale:** il fail-closed copre il lint rosso, non l'eccezione del tool.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** due frasi chiudono il buco senza toccare l'architettura.
* **Proposed Edit:**
  * Location: dopo riga 347
  * Change: aggiungere: `Un `export` che fallisce è una classe di guasto distinta da un lint rosso: `status.json` riporta `last_attempt_error: "modello non producibile: <errore>"` e Telegram manda un messaggio dedicato. Una modifica a `wiki.py` è una modifica al contratto di build: va accompagnata dal bump di `schema_version`.`
  * Location: `hooks/model.py`, riga 728-730
  * Change: `Se il file manca o ha `schema_version` diverso da 1, solleva un errore` → `Se il file manca, non è JSON o ha `schema_version` diverso da 1, solleva un errore (non interpreta una versione sconosciuta)`

### M6 — Test dell'export tautologico [`egm-design`] — Severity: Medium
* **Affected Scope:** 337-341
* **Discrepancy Description:** confronta due chiamate alla stessa funzione.
* **Rationale:** un test che non può fallire non protegge il contratto.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** sostituire il riferimento con un'enumerazione indipendente costa una funzione.
* **Proposed Edit:**
  * Location: riga 337-341
  * Change: `un test che confronti l'insieme delle pagine esportate con quello che il linter considera valido` → `un test che confronti l'insieme delle pagine esportate con un'enumerazione indipendente — una scansione del filesystem su `entities/`, `concepts/`, `comparisons/`, `queries/` — e con l'insieme delle pagine a lint pulito, più un caso con una pagina con frontmatter malformato`

### M7 — "Cinque domande" con sei voci [`egm-structural`, `egm-style`] — Severity: Medium
* **Affected Scope:** 407, 173, 531
* **Discrepancy Description:** il titolo dice cinque, l'elenco contiene D1..D6; la numerazione di `## Domande aperte` parte da 5.
* **Rationale:** un conteggio sbagliato nel titolo fa dubitare dei numeri misurati.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** la correzione è meccanica e non tocca le decisioni.
* **Proposed Edit:**
  * Location: riga 407
  * Change: `### Le cinque domande del brief` → `### Le sei domande del brief`
  * Location: riga 531
  * Change: `è l'unica delle cinque domande` → `è l'unica delle sei domande`
  * Location: riga 166
  * Change: `Una sola domanda resta senza risposta, e richiede il telefono:` → `Delle sei domande del brief, una resta aperta e richiede il telefono:`

### M8 — Completezza dichiarata, decisioni ancora aperte [`egm-style`, `egm-design`] — Severity: Medium
* **Affected Scope:** 166-176, 342-344, 369-371, 124-127
* **Discrepancy Description:** "va scritto: la soglia minima di SHA" e "va deciso cosa diventa" restano aperti mentre la sezione dichiara chiuso tutto tranne D5.
* **Rationale:** chi implementa non sa se quelle voci sono requisiti o domande.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** le due decisioni sono già prese altrove nel documento (la prima pubblicazione parte dal primo SHA con il modello; il piano tecnico descrive il marcatore grigio).
* **Proposed Edit:**
  * Location: riga 342-344
  * Change: `Va scritto: la soglia minima di SHA, e il rollback limitato agli SHA che hanno il modello. In pratica la prima pubblicazione parte da lì.` → `Lo SHA minimo costruibile è il primo che contiene `export --json`; il rollback è limitato a quegli SHA, e la prima pubblicazione coincide con quello.`
  * Location: O6, riga 126-127
  * Change: `i 15 `^[general-knowledge]` restano testo, non diventano link rotti.` → `i 15 `^[general-knowledge]` diventano un marcatore grigio non toccabile con `title` esplicativo: restano leggibili e non sono link.`
  * Location: riga 369-371
  * Change: `va deciso cosa diventa (un marcatore grigio non toccabile, non la sparizione silenziosa che fa oggi il prototipo)` → `diventa un marcatore grigio non toccabile, non la sparizione che applica oggi il prototipo`

### M9 — Obiettivi non misurabili [`egm-structural`, `egm-design`, `egm-style`] — Severity: Medium
* **Affected Scope:** 101-102 e gli obiettivi 108-150
* **Discrepancy Description:** la premessa dice "se non si può misurare, non è in questa lista"; `Verifica:` compare tre volte (O1, O2, O4) su tredici obiettivi, e O2 affida la propria verifica al ripristino dello scorrimento, che è del browser.
* **Rationale:** la premessa è falsa rispetto al proprio contenuto.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** bastano cinque righe di `Verifica:` e una precisazione su O2.
* **Proposed Edit:**
  * Location: riga 101-102
  * Change: `Ogni obiettivo qui sotto è un criterio di successo verificabile: se non si può misurare, non è in questa lista.` → `Ogni obiettivo qui sotto è un criterio di successo misurabile, e ognuno dice come si verifica. Dove la verifica è manuale, è dichiarata tale.`
  * Location: O2, riga 110-112
  * Change: `Verifica: dal telefono, aprire una pagina, toccare un wikilink, usare il gesto indietro e tornare alla pagina di partenza senza perdere la posizione di scorrimento.` → `Verifica: `curl` sull'HTML di ogni pagina, breadcrumb presente con i quattro livelli e link `<a href>` risolti. Il ripristino della posizione di scorrimento dipende dal browser (bfcache) e si prova a mano, senza farne un criterio del sito.`
  * Location: dopo O3 (riga 115): `Verifica: pagina, indice, breadcrumb, wikilink e backlink raggiungibili con JavaScript disattivato; solo la ricerca lo richiede.`
  * Location: dopo O5 (riga 123): `Verifica: nell'HTML non compare alcuna tabella di frontmatter; il blocco dei metadati compare in fondo alla pagina con data, confidenza, tag e fonti.`
  * Location: dopo O6 (riga 127): `Verifica: il numero di `<sup>` generati è uguale ai marcatori linkabili del modello; il box di ogni apice misura almeno 44×44 px a 375 px nei devtools.`
  * Location: dopo O7 (riga 130): `Verifica: una pagina `homelab/` citata solo da `ai-llm/` compare nei suoi backlink; una pagina senza link entranti non mostra la sezione.`
  * Location: dopo O10 (riga 141): `Verifica: quattro iniezioni di guasto del piano di verifica, compreso il rollback seguito da tre cicli che non cambiano `published_sha`.`
  * Location: dopo O12 (riga 147): `Verifica: `dpkg -l` e `pip list --user` sull'host non mostrano pacchetti aggiunti per il sito.`
  * Location: dopo O13 (riga 150): `Verifica: `git status --porcelain` vuoto sul repository della wiki dopo un ciclo completo di pubblicazione.`

### M10 — O3 senza verifica e senza limite [`egm-design`] — Severity: Medium
* **Affected Scope:** 113-115, 128-130
* **Discrepancy Description:** nessuna riga della tabella misura la lettura senza JS; "mostra tutti (N)" è il punto in cui la scelta `<details>` contro JS decide O3.
* **Rationale:** un obiettivo centrale senza prova.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** la verifica è già stata aggiunta in M9; qui si fissa il vincolo che la rende possibile.
* **Proposed Edit:**
  * Location: riga 128-130
  * Change: `tetto a 8 con "mostra tutti (N)"` → `tetto a 8 e, oltre le 8, un `<details>` nativo che apre le restanti (nessun elemento che richieda JavaScript, perché O3 lo vieta)`

### M11 — CUJ 3 promette la cronologia del browser [`egm-design`, `egm-style`] — Severity: Medium
* **Affected Scope:** 199-205
* **Discrepancy Description:** la ricerca di Material è lato client e non cambia l'URL mentre si digita; senza `search.share` il gesto indietro non riporta ai risultati.
* **Rationale:** è una promessa verificabile e probabilmente falsa.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** togliere la promessa costa meno che aggiungere una feature del tema non verificata.
* **Proposed Edit:**
  * Location: riga 203-205
  * Change: `i risultati compaiono mentre si digita entro un secondo, con titolo, wiki di appartenenza e frammento; toccandone uno si arriva alla pagina e la ricerca è un passaggio della cronologia del browser, non uno stato finto.` → `i risultati compaiono mentre si digita entro un secondo, con titolo, wiki di appartenenza e frammento; toccandone uno si arriva alla pagina. Il gesto indietro riporta alla pagina di partenza.`

### M12 — "Quattro pezzi, tre dei quali non sono codice nostro" [`egm-structural`, `egm-style`] — Severity: Medium
* **Affected Scope:** 253-287
* **Discrepancy Description:** il diagramma numera cinque blocchi, la prosa ne elenca quattro rinumerati 1-4, e l'attribuzione "tre non nostri" è falsa.
* **Rationale:** un conteggio sbagliato in apertura di sezione obbliga a ricostruire la mappa.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** la struttura reale è descrivibile in una frase.
* **Proposed Edit:**
  * Location: riga 253
  * Change: `Quattro pezzi, tre dei quali non sono codice nostro.` → `Cinque blocchi nel diagramma, quattro componenti: `publish.sh` e il watcher sono codice nostro, il build è un'immagine Docker derivata da una ufficiale, nginx è un'immagine di terze parti in sola lettura.`
  * Location: riga 274-287, numerazione
  * Change: la voce `3. **Il servizio**` diventa `3. **Il servizio**` (invariata) e la `4. **Il controllo**` resta `4.`; aggiungere al punto 1, in coda: `La validazione e lo swap del blocco `[3]` del diagramma stanno dentro `publish.sh`, non in un processo separato.`

### M13 — "Generatore" indica due oggetti [`egm-style`] — Severity: Medium
* **Affected Scope:** 291, 309, 345-347, 374, 375, 380, 438
* **Discrepancy Description:** nei primi punti "generatore" è la pipeline MkDocs, nel titolo dell'Alternativa A è il programma scritto da zero; il ripiegamento degli accenti è attribuito al publisher e agli hook in due punti diversi.
* **Rationale:** la frase chiave ("la regola vive nel generatore") diventa ambigua.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** è una sostituzione lessicale, non una riscrittura.
* **Proposed Edit:**
  * Location: righe 291, 309, 347, 374
  * Change: `Il perimetro è quello del linter, e non viene reinventato dal generatore` → `... dal sito`; `Il contratto che serve al generatore` → `Il contratto che serve alla build`; `la stessa regola vivrebbe in `wiki.py`, nella prosa di `SCHEMA.md` e nel generatore` → `... e nello staging`; `nessun ricalcolo nel generatore` → `nessun ricalcolo nel sito`
  * Location: riga 375
  * Change: `il publisher ripiega gli accenti nel testo dell'indice e vi aggiunge il titolo ripiegato` → `l'hook `search_index.py` ripiega gli accenti nel testo dell'indice e vi aggiunge il titolo ripiegato`

### M14 — Cleanup e spazio libero [`egm-design`] — Severity: Medium
* **Affected Scope:** 427-429, 777-791, 923-925
* **Discrepancy Description:** il `flock` c'è; mancano la rimozione di `src/` e `staging/` dopo lo swap e una soglia di spazio libero.
* **Rationale:** senza cleanup il conto delle cinque release non è quello dichiarato.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** il finding grezzo era impreciso sul lock; il resto regge.
* **Proposed Edit:**
  * Location: elenco `publish.sh`, riga 785-786
  * Change: `swap atomico di `current` con `rename`` → `swap atomico di `current` con `rename`, poi rimozione di `.builds/<sha>/src` e `.builds/<sha>/staging` (resta solo `site/`), e controllo dello spazio libero prima del build: sotto 2 GB il ciclo si chiude con `last_attempt_error` invece di fallire a metà`

### M15 — Immagine pinnata senza cadenza [`egm-design`] — Severity: Medium
* **Affected Scope:** 144-147 (O12), 574-576, 591
* **Discrepancy Description:** nessuno aggiorna l'immagine fino a maggio 2027 e non è detto dove si costruisce.
* **Rationale:** il container che serve la LAN resta senza patch di sicurezza per 19 mesi.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** una riga di cadenza e una di luogo chiudono il buco.
* **Proposed Edit:**
  * Location: `## Requisiti non funzionali`, voce Manutenzione, riga 574-576
  * Change: aggiungere in coda: `L'immagine del build si ricostruisce ogni trimestre — le versioni si alzano insieme e la validazione si riesegue — da una macchina con rete, non dall'host che serve il sito; il `Dockerfile` sta nel repository `wiki-site`.`

### M16 — "I due hook" contro tre moduli [`egm-structural`] — Severity: Medium
* **Affected Scope:** 364, 673-681, 716
* **Discrepancy Description:** il titolo dice due, il corpo elenca tre trasformazioni, il piano crea tre moduli.
* **Rationale:** tre conteggi diversi per lo stesso insieme.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** rinomina puntuale.
* **Proposed Edit:**
  * Location: riga 364
  * Change: `### I due hook e le otto righe di JavaScript` → `### I due hook di trasformazione, il modulo del modello e le otto righe di JavaScript`

### M17 — Alternativa I: affermazione falsa sul linter [`egm-style`] — Severity: Medium
* **Affected Scope:** 516-519
* **Discrepancy Description:** "il linter verifica solo che contenga `[[slug|`, non che sia completo" è falso: `tools/wiki.py` verifica che ogni pagina sia elencata (242-243), che i wikilink dell'indice risolvano (252-253) e che l'intestazione riporti i conteggi (255-259).
* **Rationale:** è un argomento del documento smentito dal codice che il documento stesso cita.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** verificato sul codice; il motivo del rifiuto regge anche senza questa frase.
* **Proposed Edit:**
  * Location: riga 517-519
  * Change: `il linter verifica solo che contenga `[[slug|`, non che sia completo. Un indice derivato dal filesystem è ricostruibile, uno che parsa un file a mano eredita ogni sua divergenza.` → `il linter verifica che ogni pagina sia elencata, che i wikilink dell'indice risolvano e che i conteggi dell'intestazione tornino; non verifica la struttura per sezioni né il loro ordine. Un indice derivato dal filesystem è ricostruibile, uno che legge un file a mano eredita ogni sua divergenza.`

### M18 — Misure di ricerca incompatibili [`egm-style`] — Severity: Medium
* **Affected Scope:** 236-237
* **Discrepancy Description:** "`perché` trova 20 documenti" e "`perche` e `perché` trovano entrambi 80 documenti": stessa interrogazione, due risultati, senza dire cosa cambi.
* **Rationale:** due misure che si contraddicono rendono inutilizzabile la tabella come prova.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** rimisurato sull'indice del prototipo con un metodo dichiarato: 973 voci; "perche" compare in 120 voci e "perché" in 3 sul testo non ripiegato; dopo il ripiegamento entrambe in 136.
* **Proposed Edit:**
  * Location: righe 236-237
  * Change:
    `| Ricerca con accenti | **Rotta**: "perche" trova 0 documenti, "perché" ne trova 20. Material non ripiega gli accenti |`
    → `| Ricerca con accenti | **Rotta**: sull'indice non ripiegato (973 voci) "perche" compare in 120 voci e "perché" in 3, quindi le due grafie non si trovano a vicenda. Material non ripiega gli accenti |`
    `| Ricerca con l'indice ripiegato e 8 righe di JS | "perche" e "perché" trovano entrambi 80 documenti |`
    → `| Ricerca con l'indice ripiegato e 8 righe di JS | entrambe le grafie compaiono nelle stesse 136 voci |`

### M19 — Budget dell'indice misurato su un artefatto diverso [`egm-design`] — Severity: Medium
* **Affected Scope:** 234-235, 749-757
* **Discrepancy Description:** i 217 kB sono rilevati senza il passo di folding, mentre O4 promette ≤300 kB sull'indice spedito.
* **Rationale:** la grandezza su cui poggia il budget non è quella che viene servita.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** misura eseguita: l'indice riscritto con testo e titolo ripiegati pesa 767.242 B, cioè 219.053 B con `gzip -9`, e resta sotto i 300 kB. Il margine tiene; va scritto con il numero giusto.
* **Proposed Edit:**
  * Location: riga 235
  * Change: `Indice di ricerca con le sole curate | 973 voci, 740 kB, 217 kB compressi: dentro il budget di 300 kB` → `Indice di ricerca con le sole curate | 973 voci, 730 kB, 216 kB compressi; 767 kB e 219 kB compressi dopo il ripiegamento di testo e titoli, cioè l'artefatto che viene servito: dentro il budget di 300 kB`

### M20 — Il collaudo aggira D5 e il rimando a O2 è sbagliato [`egm-structural`, `egm-style`] — Severity: Medium
* **Affected Scope:** 168-171, 944-946
* **Discrepancy Description:** D5 riguarda il nome MagicDNS e il collaudo usa l'IP; la riga 170 rimanda a un requisito di URL unico che in O2 non esiste.
* **Rationale:** un collaudo verde su IP non dice nulla sull'unica domanda aperta.
* **AI Suggestion:** PROPOSE FIX
* **Justification:** due sostituzioni puntuali.
* **Proposed Edit:**
  * Location: riga 170-171
  * Change: `Se non risolve, si accettano due URL e l'obiettivo O2 sull'URL unico va riscritto di conseguenza.` → `Se non risolve, si accettano due URL e va aggiunto a CUJ 1 il vincolo "un solo indirizzo memorizzabile per rete".`
  * Location: piano di verifica, riga 944-945
  * Change: `curl -sf http://192.168.50.192:8088/` → `curl -sf http://hermes.tail883eb1.ts.net:8088/ (l'IP resta per la diagnosi)`

## Low

### N1 — Identificatori con quattro punteggiature [`egm-structural`] — Low — PROPOSE FIX
`O1 — `, `D5 — `, `CUJ 1: `, `Alternativa A: ` convivono. Change: usare `O1 —`, `D5 —`, `CUJ 1 —`, `Alternativa A —` (trattino lungo) in tutto il documento, righe 104, 168, 180, 438.

### N2 — Righe orizzontali in 4 punti su 11 H2 [`egm-structural`] — Low — **CONTEST**
Non sono casuali: la riga 7 chiude il frontespizio, la 556 apre le appendici, la 578 separa le due appendici, la 606 introduce il piano di implementazione. La regola esiste ed è ricostruibile (separa il corpo del documento dalle sue parti accessorie), quindi non si applica.

### N3 — 1.492 contro 1.507 con la stessa etichetta [`egm-structural`] — Low — PROPOSE FIX
Riga 36-38: `I 1.492 marcatori di citazione (...)` → `I 1.507 marcatori di citazione — 1.492 linkabili e 15 `general-knowledge` — (...)`, così il totale e la sua scomposizione hanno nomi distinti.

### N4 — Colonna "Come è misurato" con valori invece di metodi [`egm-structural`, `egm-style`] — Low — PROPOSE FIX
Righe 83 e 90-93: spostare la distribuzione nella colonna Valore e il metodo nella terza. Riga 93 diventa: Valore `0 pagine senza link entranti; massimo 36 (salvatore-sanfilippo), poi 23 (enk)`, Metodo `conteggio dei link entranti per pagina`.

### N5 — "973 voci" senza unità [`egm-structural`, `egm-style`] — Low — PROPOSE FIX
Righe 116-120 e 235: `973 voci` → `973 sezioni indicizzate da 125 pagine` in entrambe le occorrenze.

### N6 — Registri diversi negli H3 [`egm-structural`] — Low — **CONTEST**
La mescolanza è deliberata e coerente con la voce del documento: i titoli in prima persona ("Cosa ho verificato prima di scrivere questo piano") annunciano sezioni di metodo, quelli interrogativi annunciano argomenti, i sintagmi nominali annunciano parti tecniche. Uniformare perderebbe l'informazione su quale tipo di sezione si sta leggendo.

### N7 — Partizione 6+6 delle alternative [`egm-structural`, `egm-style`] — Low — PROPOSE FIX
Riga 433-435: `Le prime sei sono le scelte di strumento discusse nel brief; le ultime sei sono varianti architetturali scartate mentre il piano prendeva forma.` → `Le alternative A-F rispondono alla domanda sullo strumento; G-M sono varianti discusse mentre il piano prendeva forma (la A riguarda l'architettura, la L i plugin).`

### N8 — Tre convenzioni di URL nel blocco di struttura [`egm-structural`] — Low — PROPOSE FIX
Righe 351-358: aggiungere una nota di una riga dopo il blocco: `Le rotte finiscono con `/` perché `use_directory_urls` è attivo; `/status/status.html` è fuori da quella regola perché non nasce dal build.`

### N9 — Il vincolo "servito su /" vive solo in prosa [`egm-structural`, `egm-style`] — Low — PROPOSE FIX
Dopo la riga 362, aggiungere il vincolo anche fra i requisiti non funzionali, voce Prestazioni o nuova voce Deployment: `Il sito è servito su `/` del nome: nessun sottopercorso, perché i link generati dal bridge sono assoluti.`

### N10 — Nessuna mappatura CUJ ↔ obiettivi [`egm-structural`] — Low — PROPOSE FIX
Dopo il quarto CUJ (riga 213), aggiungere: `Mappa: CUJ 1 → O2, O5, O7; CUJ 2 → O6; CUJ 3 → O4; CUJ 4 → O8, O9, O10.`

### N11 — Non-obiettivi non numerati, "il brief" senza percorso [`egm-structural`, `egm-style`] — Low — PROPOSE FIX
Riga 173: `Le altre cinque domande del brief` → `Le altre cinque domande del brief (`concepts_brief.md`)`; numerare i non-obiettivi come `NO1..NO8` e sostituire a riga 598-599 il rimando al brief con `il primo non-obiettivo`.

### N12 — "Risorse aggiuntive" chiama il brief "numeri verificati" [`egm-structural`] — Low — PROPOSE FIX
Riga 582: `il brief da cui parte questo documento: numeri verificati, confronto strumenti, vincoli dell'utente` → `il brief da cui parte questo documento: confronto strumenti e vincoli dell'utente. I suoi numeri non sono verificati: dove contano, questo documento li ha rimisurati.`

### N13 — Rimandi interni imprecisi [`egm-structural`, `egm-style`] — Low — PROPOSE FIX
Vedi M20 per il rimando a O2; qui resta la riga 342-344, già corretta da M8.

### N14 — Metafore che sostituiscono il meccanismo [`egm-style`] — Low — PROPOSE FIX (parziale)
Si applicano solo le ripetizioni provate: riga 137 `non può lasciare in pagina l'aria di essere aggiornato` → `non può lasciare in pagina dati che sembrano aggiornati`; riga 213 `e non finge di essere fresco` → `e il ritardo è dichiarato in pagina` (già in B3); riga 281-282 `Se sfonda, muore il container, non l'agente: l'OOM killer non ha modo di scegliere Hermes` → `Se supera il tetto di 512 MB, il kernel termina il container del build e non il processo dell'agente`. **CONTEST** su riga 68-71: `Un sito che sembra aggiornato è peggio di un sito fermo` è la tesi del documento e giustifica il fail-closed; resta.

### N15 — Lessico incoerente [`egm-style`] — Low — PROPOSE FIX
`raw` per i file (mai `sorgenti raw` o `fonti`), `elenco delle citazioni` per la lista a fondo pagina, e `fonte` solo per il singolo file citato; `ciclo` per un giro del timer, `tentativo` per l'esito registrato in `status.json`. Occorrenze: 37, 44, 84, 119, 122, 126, 130, 154, 191, 195, 220, 230, 234, 296, 355, 367, 373, 288, 136, 139, 404.

### N16 — Affermazioni non misurate nelle alternative [`egm-style`] — Low — PROPOSE FIX
Righe 452, 470, 487: `resa mobile ottima` / `buona` / `la migliore del gruppo` → elencare i fatti (`wikilink, backlink, ricerca e grafo inclusi`; `nessun wikilink nativo; ricerca assente`) e scrivere `resa mobile non misurata` dove manca la prova. Righe 445-447, 462, 499, 538-540: `nessuno riapre` → `la parte che il prototipo non deve scrivere`; `zero lavoro` → `nessun codice da scrivere`; `ogni sessione` → `ogni ricerca di una parola accentata`.

### N17 — Ridondanze e avverbi [`egm-style`] — Low — PROPOSE FIX (parziale)
Si applicano: riga 300, 362, 574 (tre occorrenze di "X, non un dettaglio": tenerne una sola, alla 574); riga 96-97 (nota di revisione sulle 149 pagine: eliminare); riga 244-249 (ultima frase che ripete la precedente: unire). **CONTEST** su "esattamente/affatto/davvero": sono tre occorrenze in un documento di 969 righe e non aggiungono rumore misurabile.

### N18 — Dettagli di contratto aperti [`egm-design`] — Low — PROPOSE FIX
`status.json` e `.builds/` usano lo SHA completo, breve solo in visualizzazione (riga 389-393 e 260); comportamento su `schema_version` sconosciuta: vedi M5; riga 233 `14,1 kB compressi` → `14,1 kB compressi con gzip -9, HTML completo della pagina`; riga 242 e 574-576: aggiungere `(annuncio del 2026, non verificato in questa sede)` accanto alla data di fine vita.

---

## Nota di metodo

Sono `CONTEST` quattro voci su 46 (N2, N6, e due punti dentro N14 e N17): in tutti e quattro i casi il testo difende una scelta che il reviewer ha letto come difetto, e la difesa si appoggia a una riga del documento o a una misura, non a un'opinione.

Tre misure nuove sono state eseguite per sostituire numeri non riproducibili (M3, M18, M19) e una quarta per correggere la ripartizione dei file (M4). Le altre cifre del documento non sono state rimisurate e restano come sono.

## Applicato

Documento: 969 → 1111 righe. Struttura verificata dopo l'applicazione: 10 blocchi di
codice (pari), 70 titoli senza salti di livello, 3 tabelle con numero di colonne
coerente, nessuna sequenza di righe vuote multipla.

**Quattro interventi extra, oltre alle 42 voci approvate**, tutti su fatti che
avevo già misurato durante la review:

1. `| Link entranti per pagina |` nella tabella del corpus riportava "255 sorgenti
   distinte citate"; misurate 255 coppie (pagina, bersaglio) su 80 bersagli distinti.
2. La forma dei link relativi in tabella era `](../<wiki>/...)`; nel corpus sono 180
   link, tutti con due `../` (`](../../<wiki>/<tipo>/<slug>.md)`).
3. La pagina di stato diventa `/status/` (file `status/index.html`) in tutto il
   documento, comprese le voci degli item 12, 15, 18 e il piano di verifica: era la
   conseguenza diretta del mount di directory richiesto da B3.
4. `217 kB per l'indice di ricerca` nei requisiti non funzionali, allineato alla
   misura dell'artefatto ripiegato (219 kB).

**Residui aperti, noti e non applicati** (fuori dal perimetro approvato):

* R1 — Stage 5 F12: il documento continua a dichiarare "127 righe di hook e 8 di JS";
  nel prototipo sono 141 e 9. Occorrenze alle righe 429-430, 483 e 693-694.
* R2 — Quattro Blockers dello Stage 6 non erano stati consolidati nel report dello
  Stage 7, quindi non sono stati toccati: chi scrive `sha` e `lint` nel modello
  (`export` gira su un archivio senza `.git`); la posizione di `model.json` rispetto
  a `mkdocs.yml` che `hooks/model.py` usa come riferimento; il formato dell'indice di
  ricerca e il calcolo delle 973 voci attese; lo schema di `state.json` e le tre
  sonde di `notify.py`.
* R3 — L'item 24 cita `SITEMAP.md`, che il repository non contiene.
