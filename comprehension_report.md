# Stage 6 — Comprehension Test

* **Target:** `/root/development/wiki-site/docs/designs/wiki-site.md`
* **Range:** FeatureRange 1..607 e DetailedRange 608..969 (969 righe)
* **Modalità:** REVIEW, preset interactive
* **Reviewer:** `egm-comprehension` (Naive Reader), contesto fresco, sola lettura
* **Standard:** `resources/document_standards/re-implementation-standard.md`
* **GoldfishContext:** `/root/development/llm-wiki/README.md`, `SCHEMA.md`, `tools/wiki.py`

Evidenza di lettura dichiarata dal reviewer:

```text
Read: .../agents/comprehension.md, .../document_standards/re-implementation-standard.md,
      /root/development/wiki-site/docs/designs/wiki-site.md:1-969
```

Criterio applicato: un ingegnere competente, con in mano solo il documento (più i
tre file di contesto letti come sono), deve poter scrivere codice funzionalmente
identico al primo tentativo senza indovinare default, gestione errori, schemi di
output o transizioni di stato.

## Esito

25 finding: **6 Blocker, 11 Major, 8 Nit**. Il documento racconta bene *cosa* vuole
e *perché*; non è ricostruibile *come*. I buchi non sono di prosa: sono contratti
che il piano nomina e non scrive, e in sei casi sono bloccanti perché un
re-implementatore deve scegliere al posto dell'autore su punti che cambiano
l'architettura.

| # | Sev | Titolo | Ambito |
| --- | --- | --- | --- |
| 1 | Blocker | Chi scrive `sha` e `lint` nel modello? `export` gira su un archivio senza `.git` | 306-347, 311-332, 619-629, 783 |
| 2 | Blocker | `model.json` sta in `.builds/<sha>/` ma l'hook lo cerca accanto a `mkdocs.yml`; l'invocazione docker non esiste | 260-262, 726-727, 783-784 |
| 3 | Blocker | L'autenticazione al repository privato non è descritta in nessun punto | 274-279, 703, 924-925, 952 |
| 4 | Blocker | "Riga di contesto" dei backlink: nessun campo nel contratto, e l'hook non può rileggere il sorgente | 128-130, 327, 372-374 |
| 5 | Blocker | Il formato dell'indice di ricerca non è scritto, quindi nemmeno l'asserzione sulle 973 voci | 749-757, 234-237, 357, 814 |
| 6 | Blocker | `state.json` non è mai definito, insieme alle tre sonde di `notify.py` | 831-841, 404-405 |
| 7 | Major | Il contratto CLI di `publish.sh` non è scritto: exit code, `--rollback`, `--force`, `failed.txt`, e cosa fa `last_attempt_at` nei giri a vuoto | 777-791, 134-141, 703, 923 |
| 8 | Major | La forma canonica di `markers[].target` per i 69 marcatori cross-wiki non è definita | 324-325, 366-370 |
| 9 | Major | Il linter conta come backlink solo i wikilink: riusando quella mappa, "Citata da" perde le 180 citazioni cross-wiki | 621-623, 327, 372-374 |
| 10 | Major | O2 vuole il titolo nel breadcrumb, la verifica dice che `navigation.path` non lo dà: manca la decisione | 238, 108-112 |
| 11 | Major | Il ripiegamento degli accenti non specifica la funzione, il campo, né l'aggancio JS | 375-378, 769-775, 236-237 |
| 12 | Major | Tutti i pesi sono "compressi" ma `nginx.conf` non attiva la compressione; e il punto di mount del file non è detto | 864-872, 568-570 |
| 13 | Major | `8088:80` pubblica su tutte le interfacce: O11 non è garantito da nessuna riga | 142-143, 858, 946 |
| 14 | Major | Le due pagine generate non hanno scheletro, ordinamento né collation | 289-304, 793-806, 352-353 |
| 15 | Major | I campi obbligatori del frontmatter: tre per il documento, sette per il linter; e `sources` è citato ma assente dallo schema | 621-628, 318-331 |
| 16 | Major | Quattro opzioni di `mkdocs.yml` nominate senza effetto osservabile; manca "come appare una pagina raw" | 709-721, 355-362 |
| 17 | Major | I controlli di `validate.py` sono esiti, non algoritmi: tre possono passare sempre | 808-819 |
| 18 | Nit | "195 file" contro 125+62+7=194; "858 link assoluti" contro 804 wikilink | 220, 360-362, 88 |
| 19 | Nit | Un solo timer con heartbeat e giro ordinario: `notify.py` non ha modo di distinguerli | 884-890, 836 |
| 20 | Nit | `.builds/<sha>/` nel diagramma, `builds/` in installazione; forma del link `current` non detta | 260-265, 923-924 |
| 21 | Nit | L'evidenza centrale vive in una directory di scratch fuori dai due repository | 582, 584 |
| 22 | Nit | Attori e macchine senza definizione: "l'agente", "LXC 111", Tailscale/MagicDNS, nessun glossario | 13-14, 71, 169, 424, 525 |
| 23 | Nit | "I due hook" in un piano che ne crea tre | 364, 673-679, 716 |
| 24 | Nit | "Le cinque domande del brief" ne elenca sei (D1..D6) | 164-176, 407-429 |
| 25 | Nit | O3 senza JS: quale navigazione sopravvive, e `navigation.instant` mai dichiarato assente | 113-115, 360 |

## Finding

### 1 — Blocker: chi produce `sha` e `lint` nel modello

Il contratto (311-332) contiene `"sha": "<sha di origin/main costruito>"` e
`"lint": {"errors": 0, "warnings": 0}`, ma `export` gira sull'archivio estratto
(783), dove `.git` non esiste: lo SHA lo conosce solo `publish.sh` via
`git ls-remote`. Il documento non dice chi scrive quel campo né con quale
meccanismo (argomento `--sha`, variabile d'ambiente, riscrittura del JSON dopo
l'export), e non dice se `export` esegue lui il lint o riceve i conteggi. Il
determinismo byte-per-byte (627-629) dipende da questa scelta.

*Cosa chiude il buco:* la firma completa di `export` (argomenti, incluso lo SHA in
input), l'ordine esatto delle operazioni nel publish, e la regola sul lint rosso
(modello non prodotto, oppure prodotto con `lint.errors > 0`).

### 2 — Blocker: dove sta `model.json`, e come si lancia il container

Il diagramma colloca `model.json` in `.builds/<sha>/`, fratello di `staging/`
(260-262); `hooks/model.py` lo carica "dalla directory del file di configurazione"
(726-727), cioè da dove sta `mkdocs.yml`, che secondo la riga 262 è dentro
`staging/`. O lo staging lo copia o l'hook non lo trova, e il documento non dice
quale. Manca del tutto l'invocazione del build: nessun `docker run` con immagine,
`-v`, `-w` e comando; il tag `wiki-site-build:1` compare solo nel piano di verifica
(932), e non è detto dove finisca `site/` rispetto ai mount.

*Cosa chiude il buco:* l'albero esatto di `.builds/<sha>/` con `model.json`,
`mkdocs.yml`, `docs/` e `site/`, più la riga completa del `docker run`.

### 3 — Blocker: nessuna descrizione dell'accesso al repository privato

Il sistema si regge su `git ls-remote` e `git archive` verso un repository privato.
Il documento nomina solo "la chiave esistente" (703) e "token Telegram già in uso da
Hermes" (925): non dice tipo di credenziale (deploy key SSH o token HTTPS), percorso,
nome del remote, come viene passata a git (`GIT_SSH_COMMAND`,
`url.<base>.insteadOf`, credential helper), gestione di `known_hosts`, né quali
variabili esistono davvero — `WIKI_REPO` appare solo nell'iniezione di guasti (952).

*Cosa chiude il buco:* l'elenco nominato delle chiavi in `/etc/wiki-site/env`
(`WIKI_REPO`, `WIKI_REF`, `SITE_PORT`, `TELEGRAM_TOKEN`, `TELEGRAM_CHAT_ID`) con il
comando git che le consuma, e dove vive la chiave privata con quali permessi.

### 4 — Blocker: la riga di contesto dei backlink non ha casa

O7 chiede titolo, wiki e "una riga di contesto"; la riga 373 ripete che i dati
arrivano dal modello e che il generatore non ricalcola nulla; lo schema (327) porta
`{slug, wiki}`. Il contesto non esiste in nessun punto del contratto, e il lettore
non sa se va aggiunto al modello (con quale regola di estrazione e quale
troncamento) o se l'hook deve rileggere il sorgente, cosa che il documento vieta.

*Cosa chiude il buco:* `backlinks[].context` con regola e troncamento, oppure la
rinuncia esplicita con modifica di O7.

Nota: stesso finding di Stage 5 F3, visto dal lato del contratto invece che da
quello dello scope. È il candidato più chiaro a una decisione di rimozione.

### 5 — Blocker: il formato dell'indice di ricerca

"Riscrive il JSON compatto", "973 voci", "toglie le raw", "aggiunge il titolo
ripiegato": nessuno di questi enunciati dice lo schema di una voce (`location`,
`title`, `text`? altri campi?), se l'indice lunr è precompilato o costruito lato
client, il nome del campo con il titolo ripiegato, e soprattutto **come si calcola
il numero atteso di voci** con cui l'hook deve fallire. 973 non è derivabile dal
modello, che conta 125 pagine: o è una costante (fragile, e il documento non lo
ammette) o è una regola (una voce per heading) che non è scritta.

*Cosa chiude il buco:* schema della voce, nome del campo ripiegato, formula
dell'asserzione.

### 6 — Blocker: `state.json` e le sonde del watcher

`state.json` è il secondo file di stato del sistema e non è mai definito: schema,
percorso, chi lo scrive, come si rappresenta "ultima transizione notificata". Senza
di esso la regola "invia solo le transizioni" non è implementabile. Mancano anche i
tre rilevamenti: quale URL si interroga per "HTTP diverso da 200" (e con quale
timeout), come si rileva "container fermo" (`docker inspect`? nome del container?),
e con quali privilegi `notify.py` invoca docker.

*Cosa chiude il buco:* schema di `state.json` con esempio, tabella delle
transizioni (condizione → chiave → messaggio → quando si azzera), URL e timeout
della sonda, comando per lo stato del container.

### 7 — Major: il contratto CLI di `publish.sh`

"Codici di uscita distinti" senza numeri; `--rollback <sha>` compare solo nel piano
di verifica (957) e non fra le modifiche dell'item 12; il README promette "come si
forza una pubblicazione" (703) senza che esista un `--force`; `failed.txt` è
nominato (782, 923) senza formato, senza autore, senza regola di rimozione. C'è
anche un conflitto con O9: se il publisher "esce senza fare nulla" quando lo SHA è
già pubblicato, quel giro aggiorna `last_attempt_at` oppure no?

*Cosa chiude il buco:* tabella opzioni → comportamento, tabella exit code con
significato, formato di `failed.txt` con la regola di rimozione, decisione
esplicita sui giri a vuoto.

### 8 — Major: la forma canonica di `markers[].target`

L'esempio mostra solo `raw/articles/...`. Per `^[../homelab/raw/articles/x.md]`
non si sa se il modello conserva il relativo (con `../`), lo normalizza alla radice
del repository, o produce `homelab/raw/articles/x.md`. Da questa scelta dipendono
l'URL `/<wiki>/raw/<tipo>/<nome>/` e la deduplica dei marcatori verso lo stesso
file. Non è detto nemmeno come si rappresenta `general-knowledge`: `linkable: false`
con quale `target` (`null`? stringa?).

*Cosa chiude il buco:* la regola di normalizzazione, più un esempio di voce per il
caso cross-wiki e uno per `general-knowledge`.

### 9 — Major: i backlink cross-wiki spariscono se l'export riusa il linter

Il documento dice che l'export riusa "la mappa dei link entranti". Nel codice
attuale `inbound` viene incrementato **solo** nel ciclo dei wikilink
(`tools/wiki.py:233-240`); i risultati di `check_mdlinks` alimentano `links` ma non
`inbound`. Riusando quella mappa, "Citata da" perde silenziosamente tutte le 180
citazioni cross-wiki, e O7 resta formalmente soddisfatto: nessun test fallisce, la
sezione è solo più povera. Il documento non definisce nemmeno cosa conta come
backlink (wikilink? link markdown relativi? entrambi? `index.md` escluso perché non
pubblicato).

*Cosa chiude il buco:* la definizione di link entrante, e la nota che l'export deve
estendere il calcolo del linter.

Verifica Elephant: confermata sul codice (righe 233-240 di `tools/wiki.py`).

### 10 — Major: chi mette il titolo nel breadcrumb

La verifica (238) dice che `navigation.path` mostra gli antenati e non il titolo;
O2 chiede `home › wiki › tipo › titolo`. Le due affermazioni non possono essere
entrambe vere con la sola feature. Il documento non dice come si aggiunge il titolo
(H1 nel markdown, `page.title`, override di un partial), e la scelta tocca O2 e
`test_render.py`.

*Cosa chiude il buco:* la decisione esplicita, oppure la riformulazione di O2.

Stesso nucleo di Stage 5 F4.

### 11 — Major: il ripiegamento degli accenti non è una funzione, è un'intenzione

Metà nel publisher, metà in 8 righe di JS, e i due lati devono usare la stessa
funzione: il documento non dice quale (NFD più rimozione dei segni combinanti?
tabella per l'italiano?). Manca il punto di aggancio client: selettore del campo,
evento, e come si convive col fatto che Material ricostruisce il componente di
ricerca a ogni navigazione. "Aggiunge il titolo ripiegato" non dice in quale campo
né se il titolo entra anche nel testo indicizzato.

*Cosa chiude il buco:* la regola di folding in forma di regola (es. "NFD, si
eliminano i caratteri della categoria Mn, poi lowercase"), il nome del campo, e
l'aggancio JS dichiarato.

### 12 — Major: le cifre "compresse" senza compressione, e il mount di `nginx.conf`

Tutti i pesi sono dichiarati compressi (14 kB/pagina, 217 kB/indice, budget di
300 kB), ma l'item 19 non attiva `gzip`/`gzip_static`/`brotli`. Se il file viene
montato al posto di `/etc/nginx/nginx.conf`, sparisce anche il `gzip on` del blocco
`http` di default dell'immagine: il documento non dice **dove** viene montato
(`/etc/nginx/nginx.conf` o `/etc/nginx/conf.d/default.conf`), e la differenza
decide se `validate.py` misura byte compressi o no.

*Cosa chiude il buco:* la direttiva di compressione con tipi MIME e livello, e il
percorso di mount del file.

Correzione all'argomento del reviewer: la regola `immutable` per "gli asset con hash
nel nome" non è infondata — Material hasha i propri bundle
(`assets/stylesheets/main.57a0d79d.min.css` lo conferma nel prototipo) — ma i nostri
file (`javascripts/query-fold.js`, `extra.css`) non sono hashati, quindi la regola va
scritta per categoria, non per "gli asset".

### 13 — Major: `8088:80` non è "solo LAN e Tailscale"

O11 è un obiettivo, e l'unica configurazione data pubblica la porta su tutte le
interfacce dell'host. Non c'è bind esplicito, non c'è regola di firewall, non c'è la
frase che spiega perché l'host non è raggiungibile da fuori. La verifica dice
"verifica che la porta non risponda dall'esterno" senza dire da dove né con quale
comando.

*Cosa chiude il buco:* `192.168.50.192:8088:80` oppure la regola di firewall/ACL,
più la topologia che rende vera l'affermazione.

### 14 — Major: le pagine generate non hanno scheletro

Per la home: "sette wiki, conteggi, SHA in fondo" — quali conteggi, con quale
etichetta, in che ordine le wiki, con quale testo per lo SHA. Per gli atterraggi:
"pagine raggruppate per tipo" con "ordine per titolo" — con quale collation, e cosa
mostra ogni voce. Non è detto nemmeno che le pagine siano `index.md` (implicito,
mai scritto).

*Cosa chiude il buco:* lo scheletro markdown delle due pagine, ordinamento e
collation, e la conferma `docs/<wiki>/index.md`.

### 15 — Major: quanti e quali campi obbligatori

L'item 1 dice che l'export valida `title`, `type`, `updated`; il linter impone già
sette chiavi (`PAGE_KEYS`), valida `type` e `confidence` e nomina già il file nei
messaggi. Un'implementazione che ne valida tre accetta pagine che il linter rifiuta,
e il documento non dice quale insieme vince. Incoerenza interna: `sources` è
elencato fra i metadati da aggiungere (628) ma non compare nello schema JSON.

*Cosa chiude il buco:* la lista canonica con la nota che il lint ne impone già
sette, la frase su duplicazione o riuso, e `sources` nello schema.

Stesso nucleo di Stage 5 F2, visto dal lato del lettore.

### 16 — Major: opzioni nominate senza effetto osservabile

`use_directory_urls` (cosa cambia nell'URL), `site_url` vuoto (conseguenza su link e
sitemap), `not_in_nav` (pagina pubblicata e fuori dalla nav: chi la linka?),
`theme.font: false` (cosa resta dei font, e come si sostituiscono). Manca del tutto
"come appare una pagina raw": titolo, provenienza, breadcrumb, assenza delle sezioni
Fonti e Citata da, e se `navigation.path` funziona per una pagina fuori dalla nav.

*Cosa chiude il buco:* una riga di effetto osservabile per opzione, e un paragrafo
sulla pagina raw.

### 17 — Major: `validate.py` controlla esiti, non algoritmi

Tre controlli possono passare sempre, così come sono scritti: "zero link interni
senza bersaglio" (su quale base si risolvono gli `href`, cosa si ignora), "zero
chiavi di frontmatter visibili" (con quale euristica), "indice ≤300 kB compressi"
(con quale compressore e livello). Manca anche se "dimensione di ogni pagina sotto
200 kB" sia il file HTML o il testo.

*Cosa chiude il buco:* per ogni controllo, regola di risoluzione e unità di misura.

### 18 — Nit: aritmetiche non riconciliate

"195 file di cui 125 pagine, 62 raw, 7 indici" somma a 194: il file in più (il
`README.md` di radice?) non è nominato. "858 link assoluti generati dai wikilink"
contro "804 wikilink": 54 link non spiegati, in una sezione che fonda la propria
credibilità sui conteggi.

### 19 — Nit: un timer per due compiti

Un timer con `OnUnitActiveSec=15min` **e** `OnCalendar=*-*-* 21:00`, e un
`notify.py` senza argomento: l'heartbeat parte ogni 15 minuti, oppure il giro
ordinario non parte mai alle 21. Inoltre `Persistent=true` non ha effetto su un
timer attivato da `OnUnitActiveSec`.

*Cosa chiude il buco:* due timer, o `ExecStart` con `--heartbeat`.

### 20 — Nit: due nomi per lo stesso albero

`.builds/<sha>/` nel diagramma e nel §12, `builds/` nella lista di installazione.
Non è detto se `current` è un link relativo o assoluto, cosa che conta per il mount
in nginx e per lo swap.

### 21 — Nit: l'evidenza del documento è in una directory effimera

I numeri della tabella 224-242 e il costo in righe vengono da
`~/.hermes/cache/scratch/proto-mkdocs`, fuori dai due repository e soggetta a
potatura. Chi ricostruisce il sistema non può ispezionarla. Idem per
`concepts_brief.md`.

Nota: stesso fatto di Stage 5 F7, con una conseguenza diversa — là è un rischio per
il piano, qui è un buco per il lettore.

### 22 — Nit: attori e macchine senza definizione

"L'agente" (mai nominato, mai descritto come processo sullo stesso host) è il
soggetto della terza minaccia; "LXC 111" non è collegato al nome `hermes` né
all'IP usato in verifica; Tailscale e MagicDNS sono citati senza rinvio. Nessun
glossario nel documento.

*Cosa chiude il buco:* tre righe di "nomi e macchine" (host `hermes` = LXC 111, IP
LAN, nome MagicDNS) e un rinvio per Tailscale.

### 23 — Nit: "I due hook" in un piano che ne crea tre

La sezione si intitola "I due hook" (364), l'albero elenca `hooks/model.py`,
`render.py`, `search_index.py`, e la riga 716 dice "i tre file".
*Cosa chiude il buco:* "i due hook di trasformazione più il modulo condiviso".

### 24 — Nit: "Le cinque domande del brief" ne elenca sei

La sezione (164-176) rinvia a D1..D6 e parla di "altre cinque": il lettore non ha il
brief e non può ricostruire la numerazione.
*Cosa chiude il buco:* rinumerare 1-6, o dichiarare D5 fuori numerazione, e
sostituire i rimandi con i titoli delle domande.

### 25 — Nit: O3 senza JavaScript non è verificato né delimitato

O3 chiede che pagina, indice, home e navigazione fra wikilink funzionino a JS
spento; il tema costruisce drawer e toggle in JS e il piano non dice quale parte
sopravvive, né prevede un passo di verifica no-JS. Manca anche la dichiarazione che
`navigation.instant` non è attivo, che cambierebbe il comportamento del gesto
indietro richiesto da O2 e CUJ 1.

## Cosa ha verificato l'Elephant

| Verifica | Esito |
| --- | --- |
| Righe 306-347, 619-629, 783: nessuno dice chi scrive `sha`/`lint` nel modello | finding 1 confermato |
| Righe 260-262 vs 726-727: `model.json` fuori da `staging/`, hook che lo cerca lì | finding 2 confermato |
| Righe 783-784: nessun `docker run` completo, nessun `-v`/`-w`, tag solo alla riga 932 | finding 2 confermato |
| Nessuna descrizione di credenziale in tutto il documento (solo "la chiave esistente", 703) | finding 3 confermato |
| `tools/wiki.py:233-240`: `inbound` incrementato solo nel ciclo dei wikilink | finding 9 confermato |
| `tools/wiki.py:204-210`: `PAGE_KEYS` = 7 chiavi, `type` e `confidence` validati | finding 15 confermato |
| Prototipo: `assets/stylesheets/main.<hash>.min.css` hashato, `javascripts/query-fold.js` no | finding 12 corretto e ridotto alla compressione e al mount |
| Riga 173-176: "le altre cinque" + D1..D6 | finding 24 confermato |
| Riga 364 "I due hook" vs 673-679 (tre moduli) e 716 ("i tre file") | finding 23 confermato |
| `git status --porcelain` prima e dopo la wave | invariato, nessuna scrittura del reviewer |

## Non verificato

L'aritmetica degli 858 link e l'identità del 195° file (finding 18): vanno spiegati
dal documento, non ricostruiti da chi legge. Non ho rimisurato i pesi compressi,
non ho eseguito un build, e non ho verificato il comportamento di
`navigation.path` né di `navigation.instant` sulla versione del tema citata.
