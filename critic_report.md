# Stage 7 — Critic Review

* **Target:** `/root/development/wiki-site/docs/designs/wiki-site.md`
* **Range:** FeatureRange 1..607 (confine `## Piano di implementazione dettagliato`, riga 608)
* **Modalità:** REVIEW, preset interactive
* **Reviewer (dispatch diretto, contesto fresco):**
  * `egm-design` → `agents/design_critic.md`, standard `feature-spec-standard`
  * `egm-structural` → `agents/structural.md`, standard `feature-spec-standard`
  * `egm-style` → `agents/style.md`, nessuno standard assegnato
* **GoldfishContext:** `/root/development/llm-wiki/README.md`, `SCHEMA.md`, `tools/wiki.py`

Evidenza di lettura dichiarata dai tre reviewer:

```text
Read: .../agents/design_critic.md, .../feature-spec-standard.md, <target>:1-607
Read: .../agents/structural.md, .../feature-spec-standard.md, <target>:1-607
Read: .../agents/style.md, none, <target>:1-607
```

## Esito

99 finding grezzi (**8 Blocker, 40 Major, 51 Nit**): 24 design, 26 structural, 49 style.
Dopo deduplicazione e verifica: **46 finding consolidati — 8 Blocker, 20 Major, 18 Nit**.
La matrice in fondo mappa ogni finding grezzo al consolidato.

Il verdetto: la specifica è sopra la media quanto a motivazione e alternative, e
sotto la soglia minima quanto a verificabilità. Tre dei suoi numeri portanti non
reggono a un controllo indipendente (215 file e non 195; la ripartizione non chiude;
858 contro 804 senza derivazione), due asserzioni sul linter sono false, una sezione
obbligatoria dello standard manca, e la pipeline di fetch descritta non è eseguibile
come scritta.

## Blocker

### B1 — La pipeline di fetch non è eseguibile come descritta
**Ambito:** 256-279 (diagramma e svolgimento di `publish.sh`). **Fonte:** D1.

`git ls-remote` restituisce nomi di ref e object id: non popola nessun object
database locale. `git archive <sha>` subito dopo non ha oggetti da impacchettare, e
`git archive --remote=` non funziona su GitHub, che non espone `upload-archive`. La
prima riga del diagramma ("`git ls-remote` + `git archive` di `refs/heads/main`") non
descrive quindi nessuna sequenza eseguibile, e su quella riga poggiano validazione,
swap, O1 e O8.

**Raccomandazione:** sostituire il passo con un mirror bare locale
(`git init --bare` + `git fetch --depth 1 origin main`, poi `git archive` dal
mirror), oppure con il tarball API autenticato. Dichiarare dove vive il mirror, come
viene ripulito, e che con `--depth 1` il rollback è limitato agli SHA ancora
presenti.

### B2 — Il rollback viene annullato dal ciclo successivo
**Ambito:** 138-141 (O10), 274-276, 392, 571-573. **Fonte:** D2.

Dopo un rollback a `154c360`, il ciclo successivo (≤15 min) legge
`origin_main_sha = 3a1495a`, lo trova diverso da `published_sha`, e ripubblica il
commit appena scartato. `status.json` non ha un campo che esprima "questo SHA è
stato scelto". Il rollback di O10 dura un quarto d'ora.

**Raccomandazione:** `pinned_sha` in `status.json`, uscita immediata del publisher
quando il pin è attivo, regola scritta su chi lo toglie (merge nuovo? comando?), e un
criterio di verifica: tre cicli dopo un rollback non cambiano `published_sha`.

### B3 — Lo stato del sito si contraddice in tre punti
**Ambito:** 131-137 (O8/O9), 207-213 (CUJ 4), 265-269, 356, 387-394, 856-859.
**Fonte:** D3, S10, Y16, Y18 (+ Stage 5 F5).

(a) O8 e le righe 303-304 dicono che la home è deterministica, "nessun timestamp";
CUJ 4 pretende che dal fondo della home si leggano ora dell'ultimo fetch, ora ed
esito dell'ultimo tentativo — dati che cambiano a ogni ciclo, non a ogni build.
(b) O9 dichiara "quattro informazioni", CUJ 4 ne chiede cinque (la differenza fra i
due SHA), e `status.json` porta comunque `origin_main_sha`.
(c) `status.html` e `status.json` sono montati come singoli file (856-859) mentre
`status.py` li scrive con temporaneo più `rename`: un bind mount di file lega
l'inode, quindi dopo il primo `rename` nginx serve il file vecchio. Il guasto è
silenzioso, cioè esattamente quello che O9 vuole impedire.

**Raccomandazione:** la home porta solo lo SHA del proprio build e un link a
`/status.html`; ogni dato temporale vive in `/status.html`, servito da un mount di
directory (`/srv/wiki-site/status/`), non di file; CUJ 4 riscritto di conseguenza;
O9 portato a cinque informazioni o CUJ 4 ridotto a quattro.

### B4 — Manca la sezione Rischi e mitigazioni
**Ambito:** fra 558 e 580; materiale sparso in 60-75, 383-405, 558-576.
**Fonte:** S1, D12.

Lo standard impone che la specifica dichiari modalità di guasto, implicazioni di
sicurezza, limiti e come saranno mitigati. Nel documento non esiste un titolo
equivalente: i tre rischi di 65-75 sono problemi senza contropartita, le mitigazioni
stanno altrove e non sono mai ricondotte al rischio che coprono.

**Raccomandazione:** tabella `Rischio | Modalità di guasto | Rilevazione |
Mitigazione | Criterio di verifica`, con almeno: credenziale sull'host condiviso con
l'agente, fallimento transitorio che blocca uno SHA, esposizione della porta, disco
pieno durante lo swap, `status.json` non scrivibile, differenza di clock (i confronti
"oltre 60 minuti" vanno fatti su epoch, non su stringhe con offset).

### B5 — Il contratto del modello non copre due requisiti che il documento gli attribuisce
**Ambito:** 57, 120-130 (O5, O7), 311-332, 372-374. **Fonte:** D7, S23 (+ Stage 5 F3, Stage 6 4 e 9).

`backlinks: [{slug, wiki}]` non ha la "riga di contesto" che O7 pretende, e la riga
373 vieta di ricalcolarla nel generatore: il requisito non è realizzabile col
contratto dato. O5 chiede che le fonti compaiano in fondo, ma il JSON espone
`markers` e non `sources` (il frontmatter `sources` è validato separatamente dai
marcatori: `wiki.py` 213-232). Inoltre la riga 57 dice che il linter "già calcola e
butta via" la contabilità dei link: `lint` calcola `inbound` solo per i wikilink
(`wiki.py` 233-240), e i 180 link markdown cross-wiki finiscono in `links` ma non in
`inbound`. Un export che riusa quella mappa perde in silenzio tutti i backlink
cross-wiki.

**Raccomandazione:** aggiungere `context` a `backlinks` e `sources` alla pagina;
correggere la riga 57 in "il linter conta i link entranti per i soli wikilink, e solo
per individuare gli orfani"; aggiungere a O7 un caso di accettazione cross-wiki.

### B6 — O11 non ha né meccanismo né verifica
**Ambito:** 142-143 (O11), 267-269, 356, 564-566, 946. **Fonte:** D6 (+ Stage 6 13).

"Raggiungibile solo da LAN e Tailscale" è asserito. `-p 8088:80` lega `0.0.0.0` e
`::`; Docker inserisce le proprie regole in `nat`/`filter` prima di quelle
dell'host, quindi un firewall dell'host non basta; e il tailnet, con le ACL di
default, dà accesso a ogni device. Nessun criterio di verifica, in un documento che
altrove misura tutto.

**Raccomandazione:** binding esplicito (indirizzo LAN o tailscale, oppure
`127.0.0.1` più reverse proxy) o regola in `DOCKER-USER`; verifica: `curl` verso
l'IP pubblico IPv6 e verso la porta da fuori deve fallire, `ss -ltnp` deve mostrare
il bind atteso.

### B7 — Un fallimento transitorio blocca lo SHA fino al merge successivo
**Ambito:** 140-141 (O10), 274-276, 402-405, 416-419 (D3). **Fonte:** D4, D20.

`failed.txt` è una lista booleana senza invalidazione: un errore di rete o un OOM
mette `X` fra i falliti e da quel momento il publisher esce a ogni ciclo finché non
arriva un merge *diverso*. Se `main` torna a uno SHA già fallito (revert), il sito
non si aggiorna più e lo stato dice "niente da fare". Contraddice O8 ("entro 60
minuti da un merge") e il principio dichiarato alle righe 68-71. Nello stesso tema:
il controllo di freschezza dell'heartbeat non copre la morte dell'LXC, dove
publisher e watcher tacciono insieme.

**Raccomandazione:** contatore per SHA (`attempts`, `last_error`, `first_failed_at`)
con politica di retry scritta e messaggio distinto fra "fallito, ritenterò" e
"fallito, rinuncio"; qualificare O8 con "salvo fallimenti registrati"; dichiarare
che l'heartbeat è un dead-man's switch che richiede un umano, o spostarlo su un
secondo host.

### B8 — Manca l'alternativa a costo zero, e non ci sono criteri di scelta
**Ambito:** 431-554, 152-162. **Fonte:** D11.

Il problema dichiarato è "leggere la wiki dal telefono". L'alternativa che può
cancellare l'intero progetto — app Obsidian mobile con il vault sincronizzato — non
è valutata, e il non-obiettivo della riga 162 parla di *scrittura*, non di lettura.
La regola delle alternative dello standard non è soddisfatta: dodici opzioni sono
confrontate in prosa, senza una funzione obiettivo dichiarata, e la scelta finale
non è fra le dodici.

**Raccomandazione:** aggiungere "Alternativa M: Obsidian mobile" con pro, contro e
motivo del rifiuto (sincronizzazione a pagamento, nessun URL condivisibile, nessuno
stato verificabile del build), e una riga che dichiari i criteri di scelta adottati.

## Major

### M1 — Nessun inventario dei segreti, e il rischio è solo spostato
**Ambito:** 72-75, 274, 402-405, 563-567, 924-925. **Fonte:** D5 (+ Stage 6 3).

Servono almeno due credenziali sull'host (lettura del repository, bot Telegram), e
il documento non dice dove vivono, con quali permessi, con quale utente gira
`publish.sh`, né come si ruotano. Il rischio 3 è mitigato per i container, non per
l'host dove girano l'agente e i suoi privilegi.
**Da fare:** utente dedicato, `/etc/wiki-site/env` 0600, tipo di credenziale
(deploy key in sola lettura), procedura di rotazione, e scenario "credenziale
revocata" come caso di test (deve produrre `last_attempt_ok: false`, non silenzio).

### M2 — La verifica di O1 non è progettata e il suo conteggio mescola grandezze
**Ambito:** 104-107, 226-229, 244-249. **Fonte:** D9, S26, Y9.

"Una validazione nostra, dopo il build" non è descritta: non si sa se legge l'HTML o
ricostruisce i link dal modello (nel secondo caso la garanzia è tautologica). Il
criterio mescola link e marcatori: i link di pagina attesi sono 984, gli apici
attesi sono 1.492, non 1.507, perché i 15 `general-knowledge` non diventano link.
**Da fare:** tre asserzioni separate (984 link, 1.492 apici, zero `[[` residue), e il
validatore specificato (input, controlli, limite di righe).

### M3 — Due conteggi per lo stesso insieme di link, e un vincolo che poggia sul numero
**Ambito:** 88, 104-107, 360-362. **Fonte:** D10, S7, Y40.

804 wikilink, 984 link attesi, 858 "link assoluti": tre numeri per la stessa
famiglia, e 858 non è derivabile dagli altri due. Il vincolo "il sito è servito sulla
radice del nome" è reale (link root-relativi con `site_url` vuoto) ma la sua
giustificazione numerica non è verificabile.
**Da fare:** rifare il conteggio sui soli file pubblicati, riportarlo come stessa
grandezza della riga 88, e sostituire il numero con la prova del comportamento del
bridge (versione, opzione, esempio di `href`).

### M4 — Il conteggio dei file non chiude, e la ripartizione non è quella dell'archivio
**Ambito:** 220-221, 584, 940. **Fonte:** D14, S8. **Verificato dall'Elephant.**

A `3a1495a` l'archivio ha **215 file**: 125 pagine curate, 62 raw, 7 `index.md`,
7 `SCHEMA.md` (più uno di radice), 7 `log.md`, e 7 file fuori dalle wiki (`.github/`,
`.gitignore`, `README.md`, `SCHEMA.md`, `tools/wiki.py`, `.agents/.../SKILL.md`,
`verify_all_wikis.py`). Il documento dice 195 file "di cui 125 pagine, 62 raw, 7
indici": la somma dà 194, il 195° non è nominato, e l'insieme dichiarato esclude i
`SCHEMA.md` che il linter usa per riconoscere le sette wiki. Una delle due
affermazioni è falsa, e questa è la prima riga della tabella che dovrebbe provare
che il piano è misurato.
**Da fare:** riportare la ripartizione completa per categoria con il comando usato.

### M5 — Un `export` rotto su `main` blocca tutti gli SHA successivi
**Ambito:** 312-315, 336-347. **Fonte:** D13 (+ Stage 6 1).

`wiki.py export` vive nel repository che l'agente scrive; un commit che lo rompe, o
che cambia lo schema senza bump, fa fallire ogni build successivo, che finiscono
tutti nella lista dei falliti (B7). Non è detto cosa fa il generatore davanti a uno
`schema_version` sconosciuto.
**Da fare:** classe di guasto distinta nello stato ("modello non producibile"), il
generatore rifiuta versioni non riconosciute invece di degradare, e la regola che
una modifica a `wiki.py` è una modifica al contratto di build.

### M6 — Il test proposto per `export` è tautologico
**Ambito:** 337-341. **Fonte:** D22.

"Il test confronta l'insieme delle pagine esportate con quello che il linter
considera valido" confronta due chiamate alla stessa funzione (`pages()`): se
`export` riusa `pages()`, non può fallire nemmeno quando salta pagine malformate.
**Da fare:** confronto con un'enumerazione indipendente (walk su `PAGE_DIRS`) e con
l'insieme delle pagine a lint pulito, più un caso con pagina malformata.

### M7 — "Le cinque domande del brief" ne elenca sei, e D5 è scritta due volte
**Ambito:** 164-176, 407-429, 531. **Fonte:** S2, S3, S4, Y1.

Il titolo dice cinque, l'elenco contiene D1..D6, e la numerazione di `## Domande
aperte` parte da 5. D5 è trattata in due punti con contenuti diversi (168-171 e
424-426), quindi il "registro delle domande" non è la fonte unica.
**Da fare:** rinumerare 1-6, un solo posto per D5, e rimandi per titolo invece che
per sigla del brief.

### M8 — "Una sola domanda resta senza risposta" mentre restano due decisioni aperte
**Ambito:** 166, 342-344, 369-371. **Fonte:** Y4, Y17, D18.

La riga 166 dichiara chiuso tutto tranne D5, poi il piano lascia aperte la soglia
minima di SHA ("va scritto") e la resa di `general-knowledge` ("va deciso"), e O6 li
dà già per decisi.
**Da fare:** chiudere i due punti nel piano tecnico (SHA minimo = il primo con il
modello; `general-knowledge` = marcatore non toccabile) e rimandare lì.

### M9 — Gli obiettivi non sono misurabili come la premessa dichiara
**Ambito:** 101-102, 108-150. **Fonte:** S9, D16, Y48. **Verificato dall'Elephant.**

La riga 101 dice "se non si può misurare, non è in questa lista". Nel documento la
parola `Verifica:` compare **tre volte** (righe 105, 110, 119: O1, O2, O4), non
cinque come riportato dai reviewer: O8 e O9 non hanno una riga di verifica. O3, O5,
O6, O7, O10, O11, O12, O13 non ne hanno. Inoltre O2 affida la propria verifica al
ripristino dello scorrimento, che è comportamento del browser, non del sito.
**Da fare:** separare in O2 "URL e breadcrumb" (verificabile con `curl`) da "scroll
conservato" (test manuale, con la nota che dipende dal browser); dare criteri
strutturali a O5, O6, O13 (assenza della tabella di frontmatter nell'HTML, box del
`<sup>` ≥44 px nei devtools, `git status --porcelain` vuoto sul repo della wiki).

### M10 — O3 (lettura senza JavaScript) non è verificato e non è delimitato
**Ambito:** 113-115, 128-130, 224-243. **Fonte:** D15 (+ Stage 6 25).

Nessuna delle quindici righe della tabella misura "la pagina si legge con JS
disattivato", e "mostra tutti (N)" è il punto in cui la scelta `<details>` contro JS
decide l'esito di O3.
**Da fare:** una riga di verifica "JS off: pagina, indice, breadcrumb, wikilink,
backlink", e `<details>` come scelta vincolata.

### M11 — CUJ 3 promette un comportamento che la ricerca del tema non ha
**Ambito:** 199-205, 375-378. **Fonte:** D17, Y35.

La ricerca di Material è lato client e non cambia l'URL mentre si digita: senza
`search.share` il gesto indietro non riporta ai risultati.
**Da fare:** abilitare `search.share` e dichiarare il formato dell'URL, oppure
togliere la promessa dalla CUJ.

### M12 — "Quattro pezzi, tre dei quali non sono codice nostro" non regge
**Ambito:** 253-287. **Fonte:** S6, Y10.

Il diagramma numera cinque blocchi, la prosa ne elenca quattro rinumerati 1-4 (il
blocco `[3]` del diagramma non ha voce in prosa), e "tre dei quali non sono codice
nostro" è falso: `publish.sh` e il watcher sono nostri.
**Da fare:** "cinque blocchi nel diagramma, quattro componenti: `publish.sh` e il
watcher sono codice nostro, il build è un'immagine derivata, nginx è di terze parti".

### M13 — "Generatore" indica due oggetti diversi
**Ambito:** 291, 309, 345-347, 374, 438. **Fonte:** Y19, Y20.

Nei primi quattro punti "generatore" è la pipeline MkDocs; nel titolo
dell'Alternativa A è il programma che si sarebbe scritto da zero. La frase chiave
(la regola vive "nel generatore") diventa ambigua. Stessa ambiguità per il
ripiegamento degli accenti, attribuito al publisher in un punto (375) e agli hook in
un altro (380).
**Da fare:** riservare "generatore" all'Alternativa A, usare "la build MkDocs" e "lo
staging" altrove; dire che il folding avviene nell'hook `search_index.py`.

### M14 — Concorrenza, igiene del filesystem e spazio libero
**Ambito:** 427-429, 777-791. **Fonte:** D19 (corretto).

Il lock è previsto (`flock`, riga 781): su questo il finding grezzo è impreciso.
Restano due buchi: non è detto se `src/` e `staging/` vengono rimossi dopo lo swap
(se restano, il conto non è "un centinaio di MB"), e non c'è una soglia di spazio
libero che scriva `last_attempt_error` invece di fallire a metà swap.
**Da fare:** cleanup dopo lo swap e controllo di spazio prima del build.

### M15 — Immagine pinnata senza cadenza né procedura di ricostruzione
**Ambito:** 144-147 (O12), 240, 574-576, 591. **Fonte:** D21.

Python 3.11, MkDocs 1.6.1 e Material 9.8.0 restano fermi fino a maggio 2027: nessuna
patch di sicurezza. Non è detto dove e quando si costruisce l'immagine, il che è in
tensione con O12.
**Da fare:** cadenza trimestrale con bump e riesecuzione della validazione, e la
macchina da cui si costruisce.

### M16 — "I due hook" in un piano che ne crea tre
**Ambito:** 364-381, 673-681, 716. **Fonte:** S5 (+ Stage 6 23).

Titolo "due hook", corpo con tre trasformazioni, piano con tre moduli.
**Da fare:** "i due hook di trasformazione più il modulo condiviso del modello".

### M17 — L'Alternativa I si regge su un'affermazione falsa sul linter
**Ambito:** 512-521. **Fonte:** Y2. **Verificato dall'Elephant.**

"Il linter verifica solo che contenga `[[slug|`, non che sia completo" è falso:
`tools/wiki.py` verifica per ogni pagina che sia elencata in `index.md` (242-243),
che ogni wikilink dell'indice risolva (252-253), e che l'intestazione riporti i
conteggi giusti (255-259). Quello che non verifica è la struttura per sezioni e il
loro ordine.
**Da fare:** riformulare il motivo del rifiuto su ciò che è vero (sezioni vuote,
`## Raw Sources` solo in alcune wiki, ordine non verificato).

### M18 — Due misure della ricerca incompatibili fra loro
**Ambito:** 236-237. **Fonte:** Y3.

"`perché` trova 20 documenti" e, con l'indice ripiegato, "`perche` e `perché` trovano
entrambi 80 documenti": la stessa interrogazione dà 20 e 80 senza che sia detto
quale indice o quale unità cambi.
**Da fare:** dichiarare per ogni riga l'indice usato e l'unità (pagine? sezioni?), e
spiegare il salto 20 → 80.

### M19 — Il budget dell'indice è misurato su un artefatto diverso da quello spedito
**Ambito:** 234-237, 355-357, 749-757. **Fonte:** D8 (corretto).

Il meccanismo di esclusione delle raw esiste (l'hook le toglie, riga 752): su questo
il finding grezzo è impreciso. Il buco reale è la misura: i 217 kB / 973 voci sono
stati rilevati senza il passo di folding, mentre O4 promette ≤300 kB sull'indice
spedito, cioè dopo che il publisher vi ha aggiunto titoli ripiegati. Il margine di
83 kB non è garantito.
**Da fare:** rieseguire la misura sull'artefatto che viene spedito, e aggiungere a
O4 la verifica che nessun URL `/raw/` compaia nell'indice.

### M20 — Il CUJ 4 e il collaudo aggirano l'unica domanda aperta
**Ambito:** 168-171, 424-426, 944-946. **Fonte:** S11, Y15 (+ Stage 5 F13).

D5 riguarda `hermes.tail883eb1.ts.net` ed è la sola domanda che può cambiare il
piano; il collaudo però usa `http://192.168.50.192:8088/`, che è il punto da
verificare. E la riga 170 rimanda a "O2 sull'URL unico", che in O2 non esiste.
**Da fare:** collaudo con il nome scelto da D5, IP solo per diagnosi, e rimando
corretto (l'unicità dell'indirizzo sta in CUJ 1, non in O2).

## Nit

| # | Finding | Ambito | Fonte |
| --- | --- | --- | --- |
| N1 | Quattro famiglie di identificatori con quattro punteggiature (`O1 —`, `D5 —`, `CUJ 1:`, `Alternativa A:`) | 104, 168, 180, 438 | S17 |
| N2 | Riga orizzontale in 4 punti su 11 H2, senza regola ricostruibile | 7, 556, 578, 606 | S13 |
| N3 | "Marcatori di citazione" vale 1.492 a riga 36 e 1.507 in tabella, stessa etichetta | 36-39, 90 | S14 |
| N4 | La colonna "Come è misurato" contiene valori e non metodi (righe 83, 93) | 79-94 | S15, Y46, Stage 5 F16 |
| N5 | "973 voci" senza unità, poi usato in O4 come se fosse noto | 235, 940, 116-120 | S16, Y36, Stage 6 5 |
| N6 | Gli H3 alternano prima persona, titoli interrogativi e sintagmi nominali | 48, 60, 217 contro 251-407 | S18 |
| N7 | "Le ultime sei sono varianti architetturali" non regge per A (architettura) e L (plugin) | 433-435 | S19, Y40 |
| N8 | Blocco URL con tre convenzioni miste; `/status.html` è un'eccezione non motivata | 351-358 | S20 |
| N9 | Il vincolo "servito sulla radice del nome" vive solo in prosa, non fra i requisiti | 360-362 | S21, Y40 |
| N10 | Nessuna mappatura fra i 4 CUJ e i 13 obiettivi | 178-213 | S22 |
| N11 | Non-obiettivi non numerati e non citabili; "il brief" senza percorso alla prima occorrenza | 152-162, 173, 598-599 | S24, S25, Y21, Y49 |
| N12 | "Risorse aggiuntive" chiama il brief "numeri verificati" mentre 434-436 dice il contrario | 582-583 | S12 |
| N13 | Rimandi interni imprecisi (D5 → "O2 sull'URL unico"; "va scritto" senza soggetto) | 168-171, 342-344 | S11, Y15, Y17 |
| N14 | Metafore che sostituiscono il meccanismo: il sito "sembra" aggiornato, "aria di fresco" ×3, "il sito finge", "il fallimento si vede", "lo strumento regge", "sfonda/muore", "stato finto", "a destinazione" | 15-16, 68-71, 137, 213, 409, 281-282, 203-205, 504 | Y5, Y6, Y28, Y35, Y38, Y39, Y49 |
| N15 | Lessico incoerente: `sorgenti raw`/`raw`/`fonti` per lo stesso file; `giro`/`ciclo`/`tentativo`/`heartbeat` per lo stesso evento; `hook`/`plugin`/`publisher` | sparse | Y24, Y25, Y31, Y32, Y33, Y49 |
| N16 | Affermazioni non misurate nelle alternative: "resa mobile ottima/buona/la migliore", "progetto attivo", "zero lavoro", "nessuno riapre", "ogni sessione", "non presidiata", "il punto più esposto" | 452, 462, 470, 487, 445-447, 499, 538-540, 553 | Y11, Y14, Y41, Y43, Y44 |
| N17 | Ridondanze e avverbi decorativi: "X, non un dettaglio" ×3, "esattamente" ×2, "affatto", "davvero", "In pratica", frase 244-249 che ripete se stessa, nota di revisione sulle 149 pagine, elenco D1-D6 telegrafico con `fail-closed` non spiegato | 96-97, 244-249, 176, 300, 362, 574 | Y26, Y27, Y29, Y30, Y34, Y37, Y45, Y47 |
| N18 | Dettagli del contratto lasciati aperti: SHA breve (7) nei campi contro `.builds/<sha>` senza lunghezza; comportamento su `schema_version` sconosciuta; peso pagina (14,1 kB) senza dire se è HTML completo né con quale compressore; fine vita del tema senza fonte citata | 233, 313, 389-393, 242 | D23, D24 |

## Matrice di tracciabilità (finding grezzo → consolidato)

| Grezzo | → | Grezzo | → |
| --- | --- | --- | --- |
| D1 | B1 | S1 | B4 |
| D2 | B2 | S2, S3, S4 | M7 |
| D3 | B3 | S5 | M16 |
| D4 | B7 | S6 | M12 |
| D5 | M1 | S7 | M3 |
| D6 | B6 | S8 | M4 |
| D7 | B5 | S9 | M9 |
| D8 | M19 | S10 | B3 |
| D9 | M2 | S11 | M20, N13 |
| D10 | M3 | S12 | N12 |
| D11 | B8 | S13 | N2 |
| D12 | B4 | S14 | N3 |
| D13 | M5 | S15 | N4 |
| D14 | M4 | S16 | N5 |
| D15 | M10 | S17 | N1 |
| D16 | M9 | S18 | N6 |
| D17 | M11 | S19 | N7 |
| D18 | M8 | S20 | N8 |
| D19 | M14 | S21 | N9 |
| D20 | B7 | S22 | N10 |
| D21 | M15 | S23 | B5 |
| D22 | M6 | S24, S25 | N11 |
| D23, D24 | N18 | S26 | M2 |
| Y1 | M7 | Y18 | B3 |
| Y2 | M17 | Y19, Y20 | M13 |
| Y3 | M18 | Y21 | N11 |
| Y4, Y17 | M8 | Y24, Y25, Y31-33, Y49 | N15 |
| Y5-8, Y22-23, Y26-30, Y34, Y37-39, Y41-45, Y47 | N14, N16, N17 | Y9 | M2 |
| Y10 | M12 | Y35 | M11 |
| Y11, Y14, Y43, Y44 | N16 | Y36 | N5 |
| Y12, Y13, Y28 | N14 | Y40 | N7, N9 |
| Y15, Y16 | M20, B3 | Y46 | N4 |
| Y48 | M9 | | |

## Duplicati con gli stage precedenti

| Tema | Stage 5 | Stage 6 | Stage 7 |
| --- | --- | --- | --- |
| Backlink: riga di contesto senza casa nel contratto | F3 | 4 | B5 (D7, S23) |
| Frontmatter: doppia validazione / insieme dei campi | F2 | 15 | M5 (D13) |
| Prototipo in directory di scratch | F7 | 21 | M15 (D21, indiretto) |
| Breadcrumb delegato a `navigation.path` | F4 | 10 | — (non riproposto: assorbito in M9) |
| 858 contro 804 | — | 18 | M3 |
| 195 contro 194 | — | 18 | M4, ora quantificato |

Questi cinque temi sono quelli su cui conviene decidere una volta e chiudere:
sono emersi da tre contesti indipendenti con la stessa conclusione.

## Verifiche dell'Elephant (correzioni applicate)

| Verifica | Esito |
| --- | --- |
| `git ls-remote` non popola un object database; GitHub non espone `upload-archive` | B1 confermato |
| Riga 856-859: `status.html` e `status.json` **sono** montati in nginx | B3 corretto: il mount c'è, il difetto è il mount di singolo file con `rename` |
| Riga 781: `flock` è previsto | M14 corretto: il lock c'è, restano cleanup e soglia di spazio |
| Riga 752: l'hook toglie le raw dall'indice | M19 corretto: manca la misura sull'artefatto spedito, non il meccanismo |
| `grep 'Verifica:'` → righe 105, 110, 119 | M9 rafforzato: 3 obiettivi su 13, non 5 |
| `tools/wiki.py` 242-259: il linter verifica presenza in indice, risoluzione dei link e conteggi | M17 confermato: la riga 517-519 è falsa |
| `git ls-tree -r 3a1495a` → 215 file; 125 pagine, 62 raw, 7 `index.md`, 7 `SCHEMA.md` (8 con quello di radice), 7 `log.md`, 7 file fuori dalle wiki | M4 confermato e quantificato |
| `tools/wiki.py` 233-240: `inbound` solo per i wikilink | B5 confermato |
| `git status --porcelain` prima e dopo la wave dei tre reviewer | invariato: nessuna scrittura |

## Non verificato

Il comportamento di `obsidian-bridge` 1.3.1 sugli `href` assoluti, il funzionamento
di `navigation.path` e `search.share` sulla versione del tema citata, la data di
fine vita di Material, e i pesi compressi del prototipo: non sono stati rimisurati
né rieseguiti. Il finding M15 sulla cadenza dell'immagine si basa sulla data
dichiarata dal documento, non su una fonte verificata (vedi N18).
