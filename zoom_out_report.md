# Stage 5 — Initial Zoom Out Review

* **Target:** `/root/development/wiki-site/docs/designs/wiki-site.md`
* **Range:** 1..969 (documento intero, 969 righe)
* **Modalità:** REVIEW (document audit), preset interactive
* **Reviewer:** `egm-zoom-out` (Skeptical Architect), contesto fresco, sola lettura
* **Standard:** `resources/document_standards/zoom-out-standard.md`
* **GoldfishContext:** `/root/development/llm-wiki/README.md`, `SCHEMA.md`, `tools/wiki.py` (+ ispezione dell'albero del vault e del prototipo in `~/.hermes/cache/scratch/proto-mkdocs`)

Evidenza di lettura dichiarata dal reviewer:

```text
Read: .../agents/zoom_out.md, .../document_standards/zoom-out-standard.md,
      /root/development/wiki-site/docs/designs/wiki-site.md:1-969
```

## Esito

16 finding: **1 Blocker, 7 Major, 8 Nit**. Il documento passa il controllo di
sufficienza del contesto (§0 dello standard): obiettivo, problema radice e
vincoli sono determinabili. Le violazioni sono di sovradimensionamento e di
contratti incompleti, non di comprensibilità.

| # | Sev | Titolo | Ambito |
| --- | --- | --- | --- |
| F1 | Blocker | Il contratto `model.json` non trasporta i 180 link relativi, e O1 non è più verificabile dalla fonte | 311-332, 104-107, 227, 811-814 |
| F2 | Major | La validazione del frontmatter è presentata come nuova ma il linter la fa già | 619-629 |
| F3 | Major | O7 chiede "una riga di contesto" per i backlink: nessun campo, nessun item | 128-130 vs 327, 737-744 |
| F4 | Major | Il breadcrumb di O2 non è prodotto da nessun item e dipende da una feature del tema che il documento stesso misura come insufficiente | 108-112, 238, 712-715, 734-747 |
| F5 | Major | `status.html`/`status.json` montati come singoli file + scrittura per `rename`: la pagina di stato si congela | 856-859, 824-827, 396-400 |
| F6 | Major | Introduce `pytest` in un repo che ha per invariante la sola stdlib, e nomina un file CI che non esiste | 635-647, 661-666 |
| F7 | Major | Il piano dipende dal prototipo che vive in una directory di scratch potato, e chiama "creazione" codice che esiste già | 580-591, 668-698 |
| F8 | Major | Tre artefatti in un documento, con i contratti scritti due volte | 215-554 vs 608-969 |
| F9 | Nit | Cinque release per un rollback che ne richiede una | 427-429, 392, 786 |
| F10 | Nit | `MemoryMax=768M` come seconda barriera su passi che leggono 622 kB | 877-882 |
| F11 | Nit | Tre file di stato persistente più `log/`, che possono divergere | 781-791, 831-841, 923-925 |
| F12 | Nit | 127 righe di hook divise in tre moduli più un loader; il conteggio è sbagliato (141 e 9, senza CSS) | 673-698, item 7-9 |
| F13 | Nit | Il piano di verifica collauda l'IP mentre l'unica domanda aperta riguarda il nome | 944-946 vs 168-171, 424-426 |
| F14 | Nit | `md_in_html` presente nel prototipo e assente dalla configurazione del piano | 712-721 |
| F15 | Nit | La forma del link cross-wiki è scritta con la profondità sbagliata (una `../` invece di due) | 89, 36, 90, 624 |
| F16 | Nit | Due voci della tabella dei numeri non sono formulate in modo verificabile | 230, 93 |

## Finding

### F1 — Blocker: il contratto del modello non copre i link che O1 promette

`pages[].links` esemplifica solo `{"target": "dave-rensin", "kind": "wikilink"}`.
I 180 link relativi `../../<wiki>/<tipo>/<slug>.md` non hanno rappresentazione nel
contratto, quindi il modello non contiene i 984 link di O1 e `validate.py` non può
verificarli dalla fonte: la verifica si riduce a un confronto di conteggi
(riga 107), che non prova l'esistenza dei bersagli. Il linter già produce quella
lista (`tools/wiki.py:184-190`, `check_mdlinks` raccoglie i target e ne verifica
l'esistenza): il dato esiste, manca dal contratto. Poiché la Tappa 1 deve essere
mergiata prima che la Tappa 2 possa costruire (righe 610-611), il contratto
incompleto è un vincolo già pubblicato su `main` quando il secondo repository
comincia.

*Rimedio:* emettere in `links[]` anche i relativi come
`{"target": "<wiki>/<tipo>/<slug>", "kind": "relative"}`, riusando l'output di
`check_mdlinks`; la verifica di O1 diventa un confronto insieme-contro-insieme
prima del build.

Nello stesso giro: `raws[].cited_by` (329-330) non è consumato da nessuno dei 24
item. È un campo speculativo che due repository dovrebbero mantenere.

Verifica Elephant: confermata. `tools/wiki.py:184-190` esiste e ha la forma
descritta; il contratto alle righe 311-332 non ha alcun ramo per i link relativi.

### F2 — Major: la seconda validazione del frontmatter

L'item 1 dichiara di aggiungere "tre cose che oggi non esistono da nessuna parte",
e la seconda è la validazione dei campi obbligatori del frontmatter. Non è vero:
`PAGE_KEYS` è alla riga 38 di `tools/wiki.py`, e `lint` verifica campi mancanti
(204-206), `confidence` contro l'enum (209-210) e i marcatori verso file
inesistenti (230). Anche i "metadati utili al sito" (`aliases`, `tags`,
`confidence`, `sources`) sono già parsati dal linter. Una seconda validazione crea
due sorgenti di verità per la stessa regola, cioè il difetto che la riga 346
attribuisce allo schema.

*Rimedio:* `export` eredita il risultato di `lint` e fallisce se è rosso; il delta
reale dell'item si riduce a (a) `linkable` per i marcatori, (b) errore sulle forme
di marcatore non previste, (c) aggregati derivati (`wikis[].pages`, `updated`).

Verifica Elephant: confermata (righe 38, 200-210, 230 di `tools/wiki.py`).

### F3 — Major: O7 promette una cosa che il contratto non rappresenta

O7 chiede "titolo, wiki di appartenenza e una riga di contesto". Il modello
rappresenta il backlink come `{slug, wiki}` (327), l'hook (737-744) produce titolo,
badge e tetto a 8, nessun item estrae la frase che contiene il link. L'obiettivo
resta senza percorso di implementazione e la sua verifica non può che fallire.

*Rimedio:* togliere il requisito, non aggiungere codice — 8 voci in fondo pagina su
375 px, per un beneficio che nessun CUJ misura. Se resta, va aggiunto `context` al
contratto e un item che lo produca.

### F4 — Major: il breadcrumb è delegato a `navigation.path`, che il documento misura come parziale

O2 richiede `home › wiki › tipo › titolo`. La tabella di verifica dice che
`navigation.path` "mostra gli antenati, non il titolo della pagina" (238). Il piano
abilita la feature (item 6) e nessun hook genera il titolo: il criterio di successo
non è coperto. In più `validate.py` conta elementi generati dal tema (812,
849-851), quindi una futura migrazione rompe insieme la resa e la rete di sicurezza.

*Rimedio:* generare il breadcrumb in `render.py` dal modello (`wiki`, `kind`,
`title` sono già nel contratto) e disabilitare `navigation.path`; far contare a
`validate.py` invarianti nostri (`<sup>` emessi dall'hook, attributi `data-`, link
confrontati col modello) invece di selettori del tema.

### F5 — Major: mount di singoli file più scrittura atomica si annullano

`status.py write` scrive con temporaneo più `rename` per garantire atomicità;
`compose.yml` monta `status.html` e `status.json` come singoli file. Un bind mount
di un file lega l'inode, non il path: dopo il primo `rename` il container continua
a servire l'inode montato all'avvio. Il guasto è silenzioso, cioè la forma peggiore
prevista dal documento (68-71), e colpisce proprio O9.

*Rimedio:* montare una directory (`/srv/wiki-site/status/`) e servire
`/status.html` da lì. Un mount solo, nessun codice in più.

Verifica Elephant: la semantica Docker dell'inode è quella descritta; il difetto è
reale e non dipende dal filesystem.

### F6 — Major: `pytest` dove l'invariante è la sola stdlib, e un file CI inesistente

`README.md`, `SCHEMA.md` e il docstring di `tools/wiki.py` dichiarano che gli
strumenti girano con `python3` puro; le dipendenze esistono solo nel blocco `uv` di
`fetch`. Il repository non ha `tests/`, né `pyproject.toml`, né `requirements`. La
sola CI è `.github/workflows/lint.yml`, 167 byte, con un unico passo
`python3 tools/wiki.py lint`. L'item 4 parla di "aggiunge il passo `pytest tests/`
accanto al lint già presente" in `.github/workflows/ci.yml`: file che non esiste.

*Rimedio:* le quattro verifiche dell'item 2 sono `assert` su JSON: un
`python3 tests/test_export.py` senza dipendenze e una riga in più in `lint.yml`. Se
`pytest` è una decisione, va dichiarata come deroga all'invariante, non nascosta in
un item.

Verifica Elephant: confermata (`ls pyproject.toml requirements*.txt tests` →
assenti; `cat .github/workflows/lint.yml` → un solo passo).

### F7 — Major: il piano poggia su una directory di scratch e non la protegge

Tutti i numeri del documento vengono da `~/.hermes/cache/scratch/proto-mkdocs`
(riga 584), che esiste ancora (15 MB, `mkdocs.yml` 36 righe, `hooks/wiki.py` 141
righe, `docs/javascripts/query-fold.js` 9 righe). È cache di sessione, soggetta a
potatura per inattività, elencata fra le "risorse aggiuntive" come se fosse un
artefatto durevole. Gli item 6, 7, 8, 9, 11 sono marcati "creazione" per file che
il prototipo contiene già in forma misurata; il piano non dice mai "porta il
prototipo". L'unico item che è davvero lavoro nuovo è il 10 (`extra.css`): nel
prototipo non esiste alcun foglio nostro.

*Rimedio:* primo commit del nuovo repository = copia verbatim di `mkdocs.yml`,
`hooks/` e `javascripts/` dal prototipo; gli item corrispondenti diventano
"adattamento da prototipo". Il costo residuo dichiarato diventa la validazione, che
è la parte nuova.

Verifica Elephant: confermata, con la precisazione sul CSS.

### F8 — Major: tre artefatti in un documento, contratti scritti due volte

Il piano tecnico contiene già dettagli di livello implementativo — schema del
modello (311-332), `status.json` (387-394), valori CSS e di configurazione — e il
piano di implementazione li ripete per file: `model.json` due volte (311-332 /
619-629), `status.json` due volte (388-393 / 824-827), il budget dell'indice di
ricerca quattro volte (234-235 / 752-757 / 814 / 941-942). Ogni campo che cambia ha
tre punti da aggiornare in un documento che esiste per essere letto da una sessione
nuova (918). Il rischio non è teorico: F1 nasce da un contratto descritto in prosa
in un punto e per file in un altro.

*Rimedio:* il piano tecnico tiene le interfacce una volta; il piano di
implementazione le cita e si limita a "quale file, quale interfaccia consuma, come
si verifica". I valori puntuali (`17 px`/`1,55`, `try_files`, `Nice=10`,
`IOSchedulingClass=idle`) appartengono al repository.

### F9 — Nit: cinque release

O10 chiede un comando unico di ritorno a uno SHA precedente. Corrente più quattro,
~21 MB l'una, è profondità speculativa, e si porta dietro il ramo di potatura "dopo
uno swap riuscito, mai prima" nel percorso critico della pubblicazione.
*Rimedio:* corrente più una precedente. `.builds/<sha>/` è indicizzato per SHA: se
servisse più profondità si allunga senza toccare altro.

### F10 — Nit: `MemoryMax` sull'unità

I requisiti non funzionali dichiarano che il piano non introduce colli di
bottiglia (568-570); il rischio di memoria argomentato (72-75) è il build, che ha
già `--memory 512m`. `git archive`, `export` e `validate` su 195 file non hanno un
profilo di memoria da contenere.
*Rimedio:* tenere il limite del container; oppure dichiarare la misura che
giustifica il secondo tetto. `Nice` e `IOSchedulingClass` restano: coerenti col
resto dell'host e a costo zero.

### F11 — Nit: tre file di stato

`status.json`, `failed.txt`, `state.json` descrivono lo stesso oggetto e possono
divergere; `failed.txt` che dice fallito mentre `status.json` dice ok è la stessa
classe di guasto che il documento teme, spostata di un file.
*Rimedio:* un solo file di stato con SHA falliti e ultimo stato notificato; `log/`
resta per la traccia testuale, che non è stato.

### F12 — Nit: tre moduli per 141 righe, e il conteggio è sbagliato

Il documento misura il proprio costo in "127 righe di hook più 8 di JS" (380-381,
442-444, 585) e su quel numero costruisce il confronto con l'alternativa A. Il
prototipo ha `hooks/wiki.py` = **141 righe** e `docs/javascripts/query-fold.js` =
**9**, quindi 150, non 135. La conclusione (la stima del brief, 150-250 righe, tiene)
sopravvive; il numero no, ed è ripetuto quattro volte.
*Rimedio:* riallineare i numeri o dichiarare cosa si è escluso dal conteggio; un
modulo solo con tre funzioni invece di `model.py` + `render.py` +
`search_index.py` con una superficie di API non specificata.

Verifica Elephant: confermata con `wc -l` (141 / 9).

### F13 — Nit: il collaudo aggira l'unica domanda aperta

D5 riguarda `hermes.tail883eb1.ts.net` ed è la sola domanda che può cambiare il
piano (529-532); il piano di verifica collauda `http://192.168.50.192:8088/`, che è
esattamente il punto da verificare. Un collaudo verde su IP non dice nulla su O2
("URL condivisibile e memorizzabile").
*Rimedio:* collaudo con il nome scelto da D5, IP come ripiego documentato per la
diagnosi.

### F14 — Nit: `md_in_html` manca

L'item 6 dichiara di essere "la configurazione del prototipo meno due cose" (719) e
elenca `tables, attr_list`; il prototipo ha anche `md_in_html` (`mkdocs.yml:30`), e
l'item 8 emette `<details>` con dentro una lista markdown (741-742).
*Rimedio:* riportare `md_in_html` e sostituire "il prototipo meno due cose" con
l'elenco effettivo delle differenze.

Verifica Elephant: confermata (`mkdocs.yml:27-30`).

### F15 — Nit: profondità sbagliata nella forma del link cross-wiki

Il documento scrive `](../<wiki>/...)` per i 180 link relativi (riga 89).
`SCHEMA.md:71` prescrive `[Proxmox VE](../../homelab/entities/proxmox-ve.md)`, cioè
due livelli, perché il link parte dalla pagina. Ho contato il corpus: 180 link
relativi nel corpo, **tutti con due `../`**, e 69 marcatori `^[../<wiki>/raw/...]`
con una sola `../` (quella è la forma relativa alla radice della wiki, coerente con
`sources: raw/...` di `SCHEMA.md:56`). Le due forme hanno contesti diversi e il
documento ne documenta una sola, quella sbagliata per il caso che cita. Dato che
l'item 1 introduce un errore "per qualunque altra forma", la distinzione va scritta
o il contratto rifiuterà metà dei link veri.

Verifica Elephant: confermata nei conteggi (180 / 69); il dettaglio sul campo
`sources:` era impreciso nel finding originale ed è corretto qui.

### F16 — Nit: due voci della tabella dei numeri

* Riga 230, "255 sorgenti distinte citate": l'unità non è dichiarata. Verificata
  dall'Elephant: 255 è il numero di coppie distinte (pagina, bersaglio del
  marcatore), su 80 bersagli distinti e 124 pagine con almeno un marcatore
  linkabile. Il numero regge, la formulazione no: va scritto "255 coppie
  pagina-sorgente".
* Riga 93: la voce si intitola "Pagine senza link entranti" con valore 0, ma la
  colonna "come è misurato" contiene il massimo dei link entranti (36, poi 23).
  Sono due misure in una riga, in una tabella che il documento presenta come tutta
  misurata.

## Cosa ha verificato l'Elephant (oltre al reviewer)

| Verifica | Esito |
| --- | --- |
| `tools/wiki.py` 38, 200-210, 230: `PAGE_KEYS`, validazione frontmatter e confidence | confermato, F2 regge |
| `tools/wiki.py:184-190`: `check_mdlinks` raccoglie i link e ne verifica l'esistenza | confermato, F1 regge |
| `.github/workflows/`: solo `lint.yml`, 167 B, un passo | confermato, F6 regge |
| `pyproject.toml`, `requirements*.txt`, `tests/` | assenti |
| `proto-mkdocs`: `hooks/wiki.py` 141 righe, `query-fold.js` 9, `mkdocs.yml` 36, nessun CSS nostro | confermato, F7 e F12 reggono |
| `mkdocs.yml:30` ha `md_in_html` | confermato, F14 regge |
| Corpus: 125 pagine, 1.507 marcatori (1.492 linkabili + 15 `general-knowledge`) | confermato, i numeri di riga 90 reggono |
| Corpus: 180 link relativi nel corpo, tutti con `../../`; 69 marcatori con `../` | confermato, F15 regge con la correzione |
| "255 sorgenti distinte" = 255 coppie (pagina, bersaglio) su 80 bersagli | numero confermato, formulazione da correggere (F16) |
| `git status --porcelain` prima e dopo la wave del reviewer | invariato (`?? concepts_brief.md`, `?? design_draft.md`, `?? docs/`): nessuna scrittura da parte del reviewer |

## Non verificato

Non usato come argomento, e da non trattare come fatto: la data di fine vita di
Material, i metadati PyPI dei tre plugin, la composizione di
`squidfunk/mkdocs-material:9.8.0`, il comportamento di Zensical, e i pesi
compressi (14,1 kB per pagina, 217 kB per l'indice) che vengono dal prototipo e che
non ho rimisurato.
