# Stage 9 — Implementation Readiness Test

* **Target:** `/root/development/wiki-site/docs/designs/wiki-site.md`
* **Range:** 718..1111 (piano di implementazione), con lettura della specifica per coerenza
* **Modalità:** REVIEW, preset interactive
* **Reviewer (dispatch diretto):** `egm-implementation-readiness` e `egm-codebase-details` con `implementation-readiness-standard`; `egm-style` senza standard
* **Contesto:** `llm-wiki` (README, SCHEMA, `tools/wiki.py`, `.github/workflows/`), prototipo `proto-mkdocs`

Evidenza di lettura:

```text
Read: .../implementation_readiness.md, .../implementation-readiness-standard.md, <target>:718-1111
Read: .../codebase_details.md, .../implementation-readiness-standard.md, <target>:718-1111
Read: .../style.md, none, <target>:718-1111
```

## Esito

77 finding grezzi (14 Blocker, 50 Major, 13 Nit) → **36 consolidati: 10 Blocker, 16 Major, 10 Nit.**

Il verdetto è netto: **il piano non è eseguibile come scritto.** Non per difetti di prosa, ma
perché nove dei suoi passi richiedono una decisione che il piano non prende, e due
presuppongono codice che non esiste. Il piano descrive *cosa* deve esistere (script,
hook, unità, fixture) e quasi mai *cosa deve fare* ciascun file: nessun item elenca le
funzioni di `publish.sh`, gli argomenti di `export`, i campi che `status.py` scrive, il
formato di `failed.txt`, i codici di uscita numerici, il contenuto della fixture.

Due errori numerici che avevo introdotto nella risoluzione dello Stage 7 sono stati
trovati e corretti prima di questo report: le pagine costruite sono **195** (125 curate +
62 raw + home + 7 atterraggi, più `404.html`), non 187; e l'indice pesa **219 kB**
compressi nell'item 9 come altrove, non 217.

## Blocker

### K1 — `publish.sh` non è scrivibile come descritto
**Ambito:** item 12 (894-913) contro 288-320. **Fonte:** IR1, CD5, ST1, ST15, IR9, IR10.

Cinque lacune nello stesso item: (a) manca l'aggiornamento del mirror, quindi `git archive`
non ha oggetti; (b) mancano il path del lock e il comportamento a lock occupato; (c)
mancano immagine, mount, `-w`, `--user` e comando del `docker run`; (d) lo "swap atomico"
è nominato, non descritto — `rename` su una directory non è atomico su tutti i filesystem,
e su un symlink si fa con link temporaneo più `os.replace`; (e) l'ordine è invertito: la
rimozione di `src/` e `staging/` è scritta prima del controllo dello spazio libero, che
deve precedere il build. Mancano anche `--rollback`, `--unpin`, `--force` e i codici di
uscita numerici, che il piano di verifica usa.

**Da scrivere:** i passi numerati nell'ordine esatto, con path assoluti, argomenti e valore
di ritorno di ciascuno; la tabella `opzione → comportamento`; la tabella
`codice → causa → cosa scrive in status.json → transizione di notifica`.

### K2 — Il modello non è producibile e non è collocabile
**Ambito:** item 1 (729-744), item 7 (838-848), item 13 (918-924), contratto 354-377.
**Fonte:** IR2, CD8, CD9.

Nessun item dice chi fornisce `"sha"` (l'export gira su un archivio senza `.git`) né chi
scrive `lint.{errors,warnings}`. Il percorso di `model.json` non è definito: il diagramma
lo mette in `.builds/<sha>/`, `hooks/model.py` lo cerca "dalla directory del file di
configurazione" (cioè dove sta `mkdocs.yml`, dentro `staging/`), e nessun item dice che
lo staging lo copia. Se finisce dentro `docs_dir`, MkDocs lo pubblica in `site/` — con
dentro fino a 120 caratteri di contesto per ogni backlink — e `validate.py` non se ne
accorge.

**Da scrivere:** `export --sha <sha> --out <path>` con la precedenza fra le due opzioni,
chi passa lo SHA, lo step "copia `model.json` in `staging/`", e `exclude_docs` o
equivalente che impedisca la pubblicazione del modello.

### K3 — I tre tentativi non hanno dati né algoritmo
**Ambito:** O10 (151-160), tabella rischi (651-652), `status.json` (446-452), item 12 (899).
**Fonte:** IR3, IR8, ST2.

`attempts` è nominato nel registro dei rischi e non esiste in nessun file; `failed.txt`
non ha formato né ciclo di vita; "intervallo crescente" non è esprimibile con un timer
fisso da 15 minuti; e la riga 899 ("uscita immediata se lo SHA è presente in `failed.txt`")
contraddice il ritentativo che la stessa sezione descrive.

**Da scrivere:** dove vive il contatore (un solo file di stato), il formato
`<sha> <attempts> <last_attempt_at> <last_error>`, i tre intervalli in minuti, la regola di
azzeramento, e la coerenza con l'uscita immediata.

### K4 — `validate.py` non ha un insieme atteso, e quello scritto è sbagliato
**Ambito:** item 14 (930-941) contro item 13 (919-924) e piano di verifica (1080-1083).
**Fonte:** IR4, ST3.

"Numero di pagine uguale al modello" non ha un numero: le pagine curate sono 125, le raw
62, ma lo staging genera anche `docs/index.md` e sette `docs/<wiki>/index.md`. L'insieme
pubblicato è 195 pagine (196 file HTML con `404.html`), non 187. Inoltre il controllo
"nessun `*.md` in `docs/` fuori perimetro" fallisce sempre, perché le otto pagine di
atterraggio sono `.md` generate. Manca il metodo per "zero link interni senza bersaglio":
prefissi ammessi, risoluzione degli href, cosa si ignora.

**Da scrivere:** l'insieme esatto atteso, la sua derivazione dal modello, l'eccezione
esplicita per le otto pagine generate, e la regola di risoluzione dei link.

### K5 — Il contratto degli apici è ambiguo in due punti che la validazione misura
**Ambito:** item 8(b) (853-861), item 9 (869-871), item 14 (934-936), O6 (128-134).
**Fonte:** IR5.

Con quale elemento si rende `^[general-knowledge]`? Se è un `<sup>`, il conteggio
"numero di `<sup>` = marcatori linkabili" non torna. Le citazioni ripetute dello stesso
bersaglio ricevono lo stesso numero (il prototipo usa un dizionario) o uno nuovo? Cambia
la numerazione e la lunghezza della lista. Mancano gli id degli anchor e il ritorno.

### K6 — Il riuso dichiarato non esiste: `lint()` non espone niente
**Ambito:** item 1 (729-744) contro `tools/wiki.py:33, 140-260`. **Fonte:** CD2, CD4, CD3.

`check_wikilink` e `check_mdlinks` sono funzioni annidate dentro `lint()`, chiudono su
`slugs`, `names`, `all_slugs` e `err`, e appendono a `errors`; `inbound` è un dizionario
locale usato solo per il warning "orphan page". `ROOT` è una costante di modulo e
`wikis()`, `pages()`, `raws()`, `lint()` la usano senza parametro: le verifiche
dell'item 2 su un vault di fixture ("pagina con frontmatter malformato", "campo
obbligatorio mancante") non sono eseguibili senza un `--root`.

**Da scrivere:** il refactor di `lint()` (estrazione di `check_wikilink`/`check_mdlinks`/
`inbound` in funzioni restituibili), il parametro `root=` o `--root`, e l'item che lo
contiene. Oggi il piano dice "riusa quello che il linter già calcola" e non è vero.

### K7 — La verifica di O7 non è implementabile col riuso dichiarato
**Ambito:** item 1 (730-733), O7 (135-140), righe 58-60. **Fonte:** CD3.

I backlink da link markdown relativi non sono nella mappa del linter (`check_mdlinks`
alimenta `links`, non `inbound`), quindi l'export che la riusa pubblica un grafo
incompleto proprio nel caso che O7 dichiara come criterio. Il prototipo ha già la
regex giusta (`hooks/wiki.py:37-47`, `RELLINK` con `PurePosixPath`): è riuso mancato.

**Da scrivere:** la quarta funzione nuova dell'export (risoluzione di
`../../<wiki>/<tipo>/<slug>.md` in slug), i valori ammessi per `links[].kind`, e se il
`context` vale anche per questi backlink.

### K8 — L'export non è deterministico, e il piano promette che lo sia
**Ambito:** item 1 (738-740), `tools/wiki.py:28-31, 79-113`. **Fonte:** CD7.

`parse_frontmatter` usa PyYAML se disponibile e un parser di ripiego altrimenti: con
PyYAML `updated: 2026-10-14` diventa un `datetime.date`, senza resta stringa, e
`json.dump` di una data solleva `TypeError`. Stesso albero, due modelli. Il piano dice
"due export dello stesso albero devono essere identici byte per byte" senza fissare i
tipi.

**Da scrivere:** tutte le date come stringhe ISO nel modello, `None` → `[]` per
`aliases`/`tags`/`sources`, e la decisione se l'export ignora PyYAML per essere
riproducibile.

### K9 — La Tappa 1 presuppone infrastruttura che non esiste
**Ambito:** item 2 (746-758), item 4 (775-780), piano di verifica (1075-1078).
**Fonte:** CD1, CD11, ST11, IR? (tavola dei test).

`.github/workflows/ci.yml` non esiste: c'è solo `lint.yml`, 12 righe, senza alcuno step
di installazione. Non esistono `tests/`, `pyproject.toml`, `conftest.py`,
`requirements.txt`. `pytest tests/` aggiunto così fallisce per comando assente, e
`ruff`/`shfmt` nel piano di verifica introducono due toolchain nuove senza item che le
installino, in un repository che dichiara come invariante la sola standard library.

**Da scrivere:** il nome del workflow da modificare, lo step di installazione, e la
decisione (dichiarata come deroga) su `pytest`, `ruff`, `shfmt` — oppure la loro rimozione.

### K10 — La Tappa 3 non è eseguibile nell'ordine dato
**Ambito:** Tappa 3 (1055-1068) contro item 18 (979-985), item 20 (1006-1012).
**Fonte:** IR11, IR12, CD12, CD14.

`docker compose up -d` precede la prima pubblicazione, ma il compose monta `current`, che
non esiste: Docker crea una directory vuota e lo swap successivo o fallisce o resta
invisibile. Inoltre `/srv/wiki-site` deve contenere `mirror/` (mai elencata), `.builds/`
(elencata come `builds/`), e l'unità systemd non dichiara `User=`, quindi gira come root
mentre il testo afferma che `publish.sh` gira "senza privilegi" — salvo poi usare
`docker run`, che richiede accesso al socket Docker.

**Da scrivere:** la sequenza di bootstrap (`current` → dove punta prima del primo build,
404 statica?), l'elenco completo delle directory, `User=wiki-site` e come quell'utente
accede a Docker, e la decisione se il socket Docker è accettabile.

## Major

| # | Finding | Ambito | Fonte |
| --- | --- | --- | --- |
| M1 | Normalizzazione di `markers[].target` non definita: per il cross-wiki non si sa se resta la stringa del Markdown o diventa `<wiki>/raw/...`; e "errore per qualunque altra forma" non enumera le forme | 361-376, 733-735 | IR6 |
| M2 | `status.json` non copre ciò che il piano chiede: mancano `attempts`, `schema_version`, `built_at` per release (la verifica del rollback lo presuppone), stato iniziale, classe di errore strutturata | 446-452, 1099-1101 | IR8, IR10 |
| M3 | "Codici di uscita distinti" senza numeri, e con `Type=oneshot` ogni codice diverso da 0 marca l'unità fallita: "niente da fare" deve essere 0 | 899, 907-909 | IR9, CD15, ST25 |
| M4 | `staging.py` si descrive in modo circolare ("copia hook, CSS, JS e config dallo staging"): manca la mappa repository → staging → container, dove vive il checkout di `wiki-site`, come si riscrive `mkdocs.yml` per la `nav` (MkDocs non accetta `nav` da riga di comando) | 918-928 | IR10, CD19, ST3 |
| M5 | Nessun item assegna l'iniezione di SHA e link `/status/` nella home generata: O8 lo richiede, l'item 13 elenca solo wiki e conteggi | 918-919, 143-145 | IR11 |
| M6 | `notify.py` non ha contratto: `state.json` senza schema, chiavi di confronto, URL e timeout della sonda, metodo per "container fermo", nome del container, formato del messaggio, deduplicazione; la soglia "in ritardo oltre 60 minuti" è ambigua | 953-964 | IR13, ST16 |
| M7 | `mkdocs.yml` è una lista della spesa: `not_in_nav` non toglie le pagine dalla navigazione (zittisce solo i warning), mancano struttura `palette`, `site_name`, percorsi degli hook e degli asset, comando di build, e `md_in_html` che il prototipo ha e il piano omette | 826-836 | IR14, CD20, ST8 |
| M8 | Nessun item dice chi costruisce `wiki-site-build:1`, quando, con quale procedura, e cosa fa `publish.sh` se l'immagine manca | 966-974, 1072-1074 | IR15 |
| M9 | La fixture non può coprire i casi dichiarati: "due wiki, tre pagine, quattro file più due raw" richiede almeno 9 file (2 `SCHEMA.md`, 2 `index.md`, 3 pagine, 2 raw), e "pagina con più di 8 entranti" è impossibile con tre pagine; manca come si testano gli hook | 1023-1043 | IR16, CD12, ST12 |
| M10 | O11 richiede una regola in `DOCKER-USER` e nessun item la crea; mancano la creazione dell'utente `wiki-site`, l'ownership di `.builds/`, `status/`, `log/`, e il passo che rende riproducibile la revoca della deploy key | 1055-1068, 161-166 | IR17, CD14 |
| M11 | `${BIND_ADDR}` in `compose.yml` non viene da `EnvironmentFile` di systemd: `docker compose` interpola da `.env` o dall'ambiente del processo, quindi `up -d` a mano produce un errore o un binding su tutte le interfacce | 979-985, 1057 | CD13 |
| M12 | La deploy key in `/etc/wiki-site/deploy-key` non viene usata da git senza `GIT_SSH_COMMAND`, `core.sshCommand` o `~/.ssh/config`: nessuno dei tre è dichiarato | 1062-1068 | CD14 |
| M13 | Il backlink cross-wiki e il `context` non hanno produttore (né la regola di estrazione: finestra, confine di parola, escape) | 370-371, 729-738 | CD10, IR7 |
| M14 | Il numero atteso di voci dell'indice non è derivabile dal modello: va aggiunto un campo (`sections` per pagina) o cambiato il controllo in "una voce per pagina del modello" | 869-874 | IR20, CD17, ST13 |
| M15 | L'item 3 (`SCHEMA.md`) parte da una premessa in parte falsa: `^[general-knowledge]` è già documentato (SCHEMA:65), gli esempi YAML esistono (SCHEMA:28-41), e `summary` è già in entrambi i domini (SCHEMA:32 e 54) — il chiarimento proposto contraddirebbe la baseline | 763-773 | CD18, ST6, ST7 |
| M16 | L'item 2 descrive test sull'albero reale del repository mentre l'item 22 motiva le fixture per non dipenderne, e non dice l'esito atteso del caso "frontmatter malformato" | 746-758 | ST11, IR16 |

## Nit

| # | Finding | Fonte |
| --- | --- | --- |
| N1 | Bordo del tetto a 8 non definito (con 8 entranti: `<details>` o no? con 9: 8+1 o 9?) e "mostra tutti (N)" non dice se N è il totale o il resto | IR18, ST22 |
| N2 | `--json` e `--out` insieme: precedenza, comportamento, atomicità della scrittura | IR19, CD? |
| N3 | "copia le 7 wiki" è un numero cablato mentre `wikis()` le scopre dal filesystem | CD? |
| N4 | `Persistent=true` è un no-op su un timer `OnBootSec`/`OnUnitActiveSec` | CD?, ST? |
| N5 | Due cadenze (15 minuti e 21:00) in una sola unità `.timer` con un solo `ExecStart`: l'heartbeat non è distinguibile — serve un secondo timer con `--heartbeat` | IR13, ST16 |
| N6 | `status.py`, `notify.py`, `publish.sh`, `query-fold.js` e le unità systemd non hanno test, mentre la regola dichiarata chiede un test per ogni ramo di `validate.py` | CD? |
| N7 | La regola "ogni ramo di `validate.py` ha un test" non è soddisfatta: otto controlli, tre casi nell'item 23 | ST19 |
| N8 | L'item 24 cita `SITEMAP.md` "se la directory lo contiene" (condizione senza regola) e "copia finale del draft" senza dire da dove a dove; `docs/designs/wiki-site.md` non compare nell'albero della Tappa 2 | CD?, ST4 |
| N9 | Etichetta `Citata da` usata in un punto e "mostra tutti" nell'altro; il test dell'item 23 verifica un testo che non gli viene indicato | ST22 |
| N10 | "nello stile dei comandi esistenti" non identifica cosa si replica (forma della CLI, errori, sola stdlib) | ST26 |

## Cosa ha verificato l'Elephant

| Verifica | Esito |
| --- | --- |
| `ROOT` costante di modulo; `wikis()`, `pages()`, `raws()`, `lint()` senza parametro | K6 confermato |
| `check_wikilink`/`check_mdlinks` annidate in `lint()`; `inbound` dizionario locale | K6, K7 confermati |
| `import yaml` con fallback (`tools/wiki.py:28-31`) e commento "fetch runs through uv" | K8 confermato |
| `.github/workflows/` contiene solo `lint.yml`; assenti `tests/`, `pyproject.toml`, `requirements.txt` | K9 confermato |
| `SCHEMA.md:32` (`type: transcript | article | summary`) e `:54` (`… | summary`) | M15 confermato: `summary` esiste in entrambi i domini |
| `README.md` ha una tabella Wiki/Dominio in inglese, `index.md` inizia con "# Wiki Index": nessun titolo italiano per wiki | il campo `wikis[].title` non ha sorgente: confermato |
| Sito del prototipo: 196 file HTML = 125 pagine + 62 raw + 8 `index.html` + `404.html` | "195 pagine costruite" corretto; il 187 della mia risoluzione era un errore |
| `git status --porcelain` prima e dopo la wave | invariato |

## Non verificato

La semantica di caricamento degli hook in MkDocs 1.6.1 (se `import model` da
`hooks/render.py` funzioni senza manipolare `sys.path`), il comportamento di
`not_in_nav`, e le versioni delle immagini: nessun pacchetto MkDocs è installato in questo
ambiente e non è stato eseguito nessun build.
