# Stage 8 — Intermediate Zoom Out Review

* **Target:** `/root/development/wiki-site/docs/designs/wiki-site.md`
* **Range:** 1..1111 (documento intero, dopo la risoluzione dello Stage 7)
* **Modalità:** REVIEW, preset interactive
* **Reviewer:** `egm-zoom-out` (Skeptical Architect), contesto fresco, sola lettura
* **Standard:** `resources/document_standards/zoom-out-standard.md`
* **Contesto assegnato:** `zoom_out_report.md` (Stage 5, per verificare la chiusura dei finding), più `llm-wiki/README.md`, `SCHEMA.md`, `tools/wiki.py`

Evidenza di lettura:

```text
Read: .../agents/zoom_out.md, .../document_standards/zoom-out-standard.md,
      /root/development/wiki-site/docs/designs/wiki-site.md:1-1111
```

## Esito

**20 finding: 2 Blocker, 6 Major, 6 Nit nuovi, più la verifica dei 16 del primo zoom-out**
(3 chiusi, 4 parziali, 9 aperti).

Il giudizio complessivo: il documento ha chiuso i difetti di testo e ha lasciato
aperti quelli strutturali, e le correzioni hanno introdotto due guasti nuovi che
valgono più di quelli che hanno chiuso. Il primo è il mount di `current`, che annulla
lo swap atomico. Il secondo è l'ordine delle scritture in `publish.sh`: lo stato viene
scritto solo alla fine del percorso felice, quindi proprio i fallimenti che O9 e O10
promettono di rendere visibili restano invisibili. Entrambi sono conseguenze dirette
della risoluzione, non difetti preesistenti.

## Stato dei finding del primo zoom-out

| # | Tema | Stato | Prova nel testo |
| --- | --- | --- | --- |
| F1 | Contratto del modello senza i 180 link relativi | **parziale** | `links[]` ha ancora solo l'esempio wikilink (369); `kind` implica un ramo mai scritto; nessun item conta i 984 |
| F2 | Validazione frontmatter presentata come nuova | **aperto** | 732-737, testo invariato; `wiki.py:204-210` fa già quel controllo |
| F3 | O7 chiede il contesto, il contratto non lo produce | **parziale, direzione sbagliata** | campo aggiunto (371), ma nessun item lo produce (729-740) e `render.py` non lo rende (857-860) |
| F4 | Breadcrumb delegato a `navigation.path` | **aperto** | 269 dice che non dà il titolo; O2 (110-112) chiede quattro livelli; nessun hook lo genera |
| F5 | Mount di singoli file con `rename` | **chiuso per `status/`** | 979-985 monta la directory e spiega l'inode |
| F6 | `pytest` e `ci.yml` inesistente | **aperto** | 746-749, 775-780 |
| F7 | Prototipo in scratch, file dichiarati "creazione" | **aperto** | 693-696; item 6-9, 11 ancora "creazione" (824-892) |
| F8 | Tre artefatti, contratti scritti più volte | **aperto e peggiorato** | `1.492` 7 volte, `300 kB` 4, `973` 3, campi di `status.json` in 3 punti |
| F9 | Cinque release | **aperto** | 495-497, 906-907 |
| F10 | `MemoryMax=768M` senza misura | **parziale** | 1008-1012 motiva il tetto, non lo misura |
| F11 | Tre file di stato | **aperto, con un quarto** | `failed.txt`, `state.json`, `status.json`, più `releases[]` (451) |
| F12 | 127+8 righe dichiarate, 141+9 misurate; tre moduli | **aperto** | 438-439, 513, 694; prototipo 141 e 9 |
| F13 | Collaudo sull'IP | **chiuso** | 1084-1086 |
| F14 | `md_in_html` assente dalla configurazione | **aperto** | 830 elenca solo `tables, attr_list` |
| F15 | Profondità del link cross-wiki | **chiuso** | 91 con `../../`, 92 distingue il marcatore |
| F16 | Due misure in una riga della tabella | **parziale** | 261 corretto; 95 contiene ancora due misure sotto "Link entranti per pagina" |

## Finding nuovi

### N1 — Blocker: `current` è un link montato, quindi lo swap non arriva a nginx

**Ambito:** 300, 676, 902-903, 979-985.

Il rollout dice che `current` è un link (676) e il servizio lo monta come sorgente di
bind (979-980). Docker risolve il percorso quando crea il container: dopo lo swap il
container continua a servire la directory legata al primo avvio. È lo stesso guasto
che il documento ha corretto per `status/` spiegando la semantica dell'inode (982-984),
non applicato al mount principale. Il sintomo è quello che il documento dichiara
peggiore: contenuto vecchio, `published_sha` corretto, nessun controllo che se ne
accorga.

**Correzione proposta:** un mount solo, `/srv/wiki-site` in sola lettura, con
`root /srv/wiki-site/current/;` in `nginx.conf`: la risoluzione del link avviene per
richiesta. Sparisce anche il secondo mount e la spiegazione dell'inode.

### N2 — Blocker: `status.json` e `/status/` si scrivono solo in fondo al percorso felice

**Ambito:** 148-152, 394-397, 894-909, 958-961.

La sequenza dell'item 12 mette la scrittura di `status/index.html` e `status.json`
dopo swap e potatura. Tutti i guasti che il documento promette di rendere visibili —
fetch rotto, lint rosso, build rotto, validazione rossa, spazio sotto 2 GB — escono
prima, con un codice distinto (908-909) e senza scrivere niente. Quindi O9 ("ora
dell'ultimo tentativo e suo esito"), O10 ("ultimo tentativo fallito alle HH:MM") e
CUJ 4 non sono soddisfatti proprio nei casi che nominano, e `notify.py`, che legge
`status.json`, non ha cosa leggere. La riga 461-462 rivendica che la pagina di stato
"non si congela quando il build fallisce": non si congela, non viene aggiornata.
Secondo caso, stesso ordine: l'uscita immediata "se lo SHA è pubblicato" precede la
scrittura, quindi `fetched_at` non avanza nei cicli inattivi e la soglia "in ritardo
oltre 60 minuti" scatta su un sistema sano.

**Correzione proposta:** un `trap ... EXIT` che chiama un solo `status.py write` con
esito e `fetched_at`, e `status.py render` subito dopo, sullo stesso ramo dei codici
di uscita; `fetched_at` lo aggiorna il passo di fetch, non quello di pubblicazione.

### N3 — Major: la pipeline è descritta due volte e le due versioni non coincidono

**Ambito:** 288-318 contro 894-903. **Regressione della risoluzione.**

Il blocco 1 dice: fetch nel mirror, SHA letto dal mirror, `git archive` (307-318). La
voce dell'item 12 dice ancora `git ls-remote` dello SHA e poi `git archive`, senza il
fetch nel mirror. È la deriva da doppia descrizione che F8 segnala in astratto.

**Correzione proposta:** una sola descrizione (307-318); l'item 12 cita i passi e non
li riscrive.

### N4 — Major: `pinned_sha` contraddice O8 ed è stato morto

**Ambito:** 141-145, 151-160, 450, 1099-1101. **Regressione della risoluzione.**

O10 dice che, finché il pin è valorizzato e diverso da `origin_main_sha`, il publisher
non pubblica (156-158), e due righe dopo che "un nuovo merge e il comando
`--unpin` lo azzerano" (158-159). Se il merge azzera il pin, il pin non blocca mai
niente; se non lo azzera, il pin viola O8. La verifica proposta passa nei due casi,
quindi non distingue.

**Correzione proposta:** il merge non azzera il pin: lo azzera solo `--unpin`, e O8 va
qualificato con "salvo pin attivo". In alternativa, `pinned_sha` e `--unpin`
spariscono e il rollback resta un'operazione puntuale, con la durata di un ciclo.

### N5 — Major: "intervallo crescente" non è esprimibile col timer dichiarato

**Ambito:** 152-155, 445-453, 652, 1008-1010. **Regressione della risoluzione.**

Il timer passa ogni 15 minuti, uguale per tutti gli SHA: un backoff richiede di
ricordare per SHA quante volte si è fallito e quando riprovare. `status.json` non ha
né `attempts` né `next_attempt_at`, e nessun altro file è dichiarato. La verifica
"tre errori di rete simulati, poi pubblicazione entro 60 minuti" non ha un oggetto da
ispezionare.

**Correzione proposta:** o si dichiara "un tentativo per ciclo, tre cicli, poi nella
lista dei falliti" — che è ciò che il timer già fa, senza stato nuovo — oppure si
scrive dove vive `attempts` e come si conta il differimento.

### N6 — Major: l'hook di ricerca confronta con un numero che il modello non contiene

**Ambito:** 869-874 contro 354-377, 266.

`search_index.py` fallisce il build "se il numero di voci non è quello atteso secondo
il modello". Il modello espone pagine, wiki e marcatori, non le 973 sezioni: il valore
atteso non è derivabile dal contratto. Stessa classe di F1: si valida una grandezza
che la fonte non trasporta.

**Correzione proposta:** l'asserzione utile è che le voci indicizzate siano una per
pagina del modello (`len(pages)`), più "nessun URL `/raw/`" e "≤300 kB".

### N7 — Major: il registro dei rischi duplica obiettivi, requisiti e piano di verifica

**Ambito:** 647-656 contro 98-172, 659-684, 1070-1104.

Le cinque righe della tabella ripetono, una per una, cose già scritte: revoca della
chiave (1062-1068), tre tentativi (O10), porta e `DOCKER-USER` (O11), disco pieno
(904-905). L'unica informazione nuova è la riga sull'orologio, che è una nota di
implementazione e starebbe nei requisiti non funzionali. Per un artefatto di ~140
righe di hook e quattro script, il documento tiene ora 13 obiettivi, 8 non-obiettivi,
6 domande, 13 alternative, 5 rischi e 24 item, e ogni registro va aggiornato quando
cambia un campo.

**Correzione proposta:** la tabella resta (lo standard feature-spec la richiede), ma
ogni riga rimanda a O-n invece di riscrivere la mitigazione; la riga sull'orologio
diventa un requisito non funzionale.

### N8 — Major: sette alternative respinte con lo stesso stampo, che duplicano argomenti già presi

**Ambito:** 565-645 contro 432-436, 267-268, 477-486.

G-M occupano 80 righe con Pro/Contro/Motivo ciascuna: G duplica D2, H duplica D3, K
duplica il testo sul folding, L duplica i metodi PyPI, M duplica il metro del
documento. Le decisioni sono già prese e già motivate altrove.

**Correzione proposta:** A e B restano al trattamento pieno; C-F in una riga ciascuna
("scartata per: Node, hosting terzo, nessun wikilink"); G-M in un elenco di una riga
con il motivo, senza i tre punti.

### N9 — Nit: `staging.py` si descrive in modo circolare, e l'item 24 cita un file che può non esistere

**Ambito:** 918-924, 1045-1051.

"copia hook, CSS, JS e config dallo staging" — li copia dal repository dentro lo
staging. L'item 24 cita `SITEMAP.md`, un residuo di template senza oggetto nel
sistema.

### N10 — Nit: `ruff` e `shfmt` entrano dal piano di verifica e da nessun'altra parte

**Ambito:** 1077-1078.

Il collaudo impone due toolchain nuove in un repository che dichiara come invariante
la sola standard library (727-728). Nessun item le installa, nessuna deroga è
dichiarata: stessa forma di F6, che resta aperta.

### N11 — Nit: `sources` e `cited_by` sono la stessa relazione nei due versi

**Ambito:** 372-375.

`pages[].sources` elenca le raw citate; `raws[].cited_by` elenca le pagine che le
citano: una si deriva dall'altra, e `cited_by` non è consumato da nessun hook.

### N12 — Nit: `releases[]` duplica la directory e la regola di potatura

**Ambito:** 451, 906-907, 676.

L'elenco è ricavabile da `.builds/`, che è indicizzato per SHA; scritto in
`status.json` è un secondo posto da tenere allineato, e nessuno lo legge tranne la
pagina di stato che lo stampa.

### N13 — Nit: due trigger in un timer, un solo `ExecStart`, heartbeat non distinguibile

**Ambito:** 1017-1019, 959.

`notify.py` non riceve argomenti e non può sapere se è stato invocato alle 21:00 o
alle 21:15: l'heartbeat "alle 21:00" dipende da un confronto di orologio dentro lo
script, che il testo non dichiara.

### N14 — Nit: O11 verifica l'IPv6 mentre il bind è IPv4

**Ambito:** 162-166. **Regressione della risoluzione.**

Il bind dichiarato è `${BIND_ADDR}` con l'indirizzo LAN (IPv4); la verifica chiede che
fallisca un `curl` verso l'IPv6 dell'host. Il verde su IPv6 non dice niente sulla
regola `DOCKER-USER` per IPv4, che è il percorso descritto dal rischio.

## Nota di metodo

Cinque finding di questo stage (N3, N4, N5, N14, e la parte di F8 sull'aumento delle
duplicazioni) sono regressioni introdotte dalla risoluzione dello Stage 7: due
descrizioni divergenti della stessa pipeline, un pin che non pinna, un backoff che il
timer non può esprimere, una verifica su un protocollo diverso da quello del bind. Li
ho verificati sul testo: sono reali come li descrive il reviewer.

Segnalo anche che F3 (contesto dei backlink) è stato chiuso **nella direzione
sbagliata**: il campo è entrato nel contratto e nessun item lo produce, il che è
peggio di un requisito senza campo, perché ora il contratto promette un dato che
nessuno calcola.
