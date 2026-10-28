# Consolidated Review Report: EGM Multi-Lens Audit

*   **Target Document:** [`docs/designs/wiki-site.md`](file:///root/development/wiki-site/docs/designs/wiki-site.md) (Righe 1–514)
*   **Data dell'Audit:** 2026-10-28
*   **Modalità di Esecuzione:** `Consolidated report with all review rubrics` (Uninterrupted Full Flow)
*   **Lenti e Subagenti Partecipanti:**
    1. **`egm-zoom-out`** (Stage 5 — Skeptical Architect & YAGNI)
    2. **`egm-comprehension`** (Stage 6 — Naive Reader & Re-implementation Standard)
    3. **`egm-design`** (Stage 7 — Hostile Design Critic)
    4. **`egm-structural`** (Stage 7 — Structural Layout Validator)
    5. **`egm-implementation-readiness`** (Stage 9 — Implementation Tech Lead)
    6. **`egm-codebase-details`** (Stage 9 — Codebase Details Auditor)
*   **Codice del Vault di Riferimento:** `/root/development/llm-wiki`

---

## 1. Executive Summary & Verdetto

L'architettura proposta (transizione da hosting locale Docker/Nginx su Proxmox LXC 111 a pubblicazione serverless su GitHub Pages con Zensical e GitHub Actions) è stata **convalidata positivamente nella sua direzione strategica** da tutti i revisori:
- Elimina alla radice i rischi di OOM per l'agente locale Hermes (4 GB di RAM senza swap).
- Azzera il debito operativo (niente demoni, reverse proxy né porte aperte sull'LXC).
- Realizza un airgap logico a whitelist che protegge trascrizioni integrali private (`raw/`) e log interni (`.agents/`).

Tuttavia, l'audit incrociato ha rilevato **6 Blocker** tecnici e architetturali, **8 Major** e **5 Nit** che impediscono l'eseguibilità del piano e la conformità agli standard di produzione senza decisioni arbitrarie o fallimenti a runtime.

**Verdetto Finale:** **REVISE (Azione Richiesta)** — I 6 Blocker e gli 8 Major devono essere integrati nel documento di design prima di passare all'implementazione del codice.

---

## 2. Tabella di Riconciliazione (Traceability Matrix)

| ID Consolidato | Severità | Oggetto del Finding | Lenti Rilevanti |
| :--- | :---: | :--- | :--- |
| **B1** | 🔴 Blocker | Flag CLI `--site-dir` non supportato da Zensical | `egm-codebase-details` |
| **B2** | 🔴 Blocker | Inversione setup dipendenze (`pyyaml`) e assenza `checkout` in CI | `egm-comprehension`, `egm-implementation-readiness` |
| **B3** | 🔴 Blocker | Falsi positivi su link con ancore (`#heading`) nel quality gate | `egm-design`, `egm-comprehension` |
| **B4** | 🔴 Blocker | Mancata intercettazione marcatori cross-wiki `^[../...]` | `egm-codebase-details` |
| **B5** | 🔴 Blocker | Mancata riscrittura di `<wiki>/index.md` (sezione `## Raw Sources`) | `egm-zoom-out`, `egm-implementation-readiness` |
| **B6** | 🔴 Blocker | Assenza della sezione obbligatoria `## Risks & Mitigations` | `egm-structural`, `egm-design` |
| **M1** | 🟡 Major | Configurazione ricerca Zensical con lingua italiana per CUJ 3 | `egm-design` |
| **M2** | 🟡 Major | Drift numerico del vault (136 note, 65 raw) e test unitari fragili | `egm-codebase-details`, `egm-comprehension` |
| **M3** | 🟡 Major | Markup HTML `.wiki-grid` e aggregazione contatori Home | `egm-implementation-readiness`, `egm-comprehension` |
| **M4** | 🟡 Major | Specifica CLI `argparse`, pulizia staging e namespace directory | `egm-implementation-readiness`, `egm-zoom-out` |
| **M5** | 🟡 Major | Riuso delle primitive esistenti da `tools/wiki.py` | `egm-zoom-out`, `egm-codebase-details` |
| **M6** | 🟡 Major | Fallback su metadati raw mancanti e deduplicazione footnote | `egm-design`, `egm-codebase-details` |
| **M7** | 🟡 Major | Esecuzione `pytest` in CI e parametri esatti deploy action | `egm-implementation-readiness`, `egm-codebase-details` |
| **M8** | 🟡 Major | Apertura link esterni in nuova scheda (`{:target="_blank"}`) e CSS | `egm-implementation-readiness` |
| **N1** | 🟢 Nit | Fallback per wikilink privi di alias (`[[slug]]`) | `egm-design`, `egm-comprehension`, `egm-codebase-details` |
| **N2** | 🟢 Nit | Localizzazione delle categorie nel breadcrumb | `egm-zoom-out`, `egm-comprehension`, `egm-codebase-details` |
| **N3** | 🟢 Nit | Prevenzione subdomain takeover tramite verifica DNS TXT | `egm-design` |
| **N4** | 🟢 Nit | Comandi bash eseguibili nel Verification Plan | `egm-implementation-readiness` |
| **N5** | 🟢 Nit | Elenco esplicito file da dismettere nella sandbox locale | `egm-implementation-readiness` |

---

## 3. Dettaglio dei Finding Consolidati

### 🔴 BLOCKERS

#### B1 — Incompatibilità CLI di Zensical: Flag `--site-dir` non supportato
*   **Problema:** La CLI di Zensical supporta unicamente i flag `-f/--config-file`, `-c/--clean`, `-s/--strict` e non riconosce `--site-dir`. Eseguire `zensical build --site-dir public_dist` provoca un errore di parsing CLI (`unexpected argument '--site-dir'`) e arresta la CI con codice 1. Inoltre, il template `mkdocs.yml` ometteva la direttiva `site_dir`, facendo compilare Zensical nella cartella predefinita `staged_site/site/`.
*   **Risoluzione Obbligatoria:**
    1. Aggiungere esplicitamente `site_dir: ../public_dist` all'interno di `templates/mkdocs.yml`.
    2. Rimuovere `--site-dir public_dist` da tutti i comandi di compilazione, invocando unicamente:
       ```bash
       zensical build --clean --config-file staged_site/mkdocs.yml
       ```

#### B2 — Inversione nell'Installazione Dipendenze e Omissione di `actions/checkout` in CI
*   **Problema:** Il workflow `.github/workflows/deploy.yml` ometteva lo step `actions/checkout@v4`, avviando i comandi su un workspace vuoto. Inoltre, lo script di staging `tools/build_public_site.py` (che importa `yaml`) veniva eseguito allo Step 2, mentre l'installazione di `pyyaml` era pianificata solo allo Step 3 con Zensical, causando un immediato `ModuleNotFoundError: No module named 'yaml'`.
*   **Risoluzione Obbligatoria:**
    Configurare l'inizio del job di CI con:
    ```yaml
    permissions:
      contents: read
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup uv and Python
        uses: astral-sh/setup-uv@v3
        with:
          python-version: "3.12"
          enable-cache: true

      - name: Install dependencies
        run: uv pip install --system zensical pyyaml pytest
    ```

#### B3 — Falsi Positivi su Link Markdown con Ancore di Sezione (`#heading`) nel Quality Gate
*   **Problema:** Quando una nota contiene un link interno a una sezione (es. `[Concetto](section/slug.md#heading)` o `[[slug#heading|alias]]`), la verifica `os.path.exists()` eseguita sul target grezzo valuta il percorso letterale `.../slug.md#heading`, restituendo `False` e facendo fallire lo script con `sys.exit(1)`.
*   **Risoluzione Obbligatoria:**
    1. In Fase C: dividere il bersaglio del wikilink su `#`: `base_slug, _, anchor = raw_target.partition('#')`. Recuperare il percorso da `slug_map` per `base_slug` e riappendere `#anchor` al link relativo.
    2. In Fase D: separare il percorso prima del check su disco: `file_path, _, _ = link_target.partition('#')`.

#### B4 — Mancata Intercettazione dei Marcatori Cross-Wiki (`^[../...]`)
*   **Problema:** Le note del vault usano marcatori di provenienza cross-wiki quali `^[../software-engineering/raw/transcripts/nome.md]`. La regex documentata nel piano `\^\[(raw/(?:transcripts|articles)/[^\]]+)\]` cerca solo stringhe che iniziano con `raw/`, ignorando i marcatori cross-wiki. Questi rimarrebbero non convertiti come testo grezzo nell'HTML pubblico.
*   **Risoluzione Obbligatoria:**
    Adottare la regex universale `re_marker = re.compile(r"\^\[([^\]]+)\]")`. Risolvere il target rispetto alla directory della nota corrente tramite `os.path.normpath(os.path.join(os.path.dirname(page_path), marker_target))` e confrontarlo con la mappa delle raw indicizzata per percorso relativo alla radice del vault.

#### B5 — Mancata Specifica della Riscrittura di `<wiki>/index.md` (Rischio Broken Links)
*   **Problema:** I file `index.md` delle 7 wiki contengono una sezione `## Raw Sources` che collega file raw (esclusi dal sito). La mancata rimozione programmata di questa sezione causa link orfani che innescano il blocco `exit 1` del quality gate. Inoltre, omettere `<wiki>/index.md` impedisce il funzionamento di `navigation.indexes`.
*   **Risoluzione Obbligatoria:**
    Nella Fase 3 dello script, copiare `<wiki>/index.md` in `staged_site/docs/<wiki>/index.md` troncando programmaticamente la sezione `## Raw Sources` (dal titolo `## Raw Sources` fino a fine file) e traducendo i wikilink delle sezioni curate sovrastanti.

#### B6 — Assenza della Sezione Autonoma `## Risks & Mitigations`
*   **Problema:** Lo standard `feature-spec-standard.md` richiede tassativamente una sezione H2 dedicata ai rischi, modalità di guasto e mitigazioni. Nel testo attuale le considerazioni sui rischi erano disperse tra Problem, NFR e Alternative.
*   **Risoluzione Obbligatoria:**
    Inserire nel documento una sezione strutturata `## Risks & Mitigations` che tratti: rotazione PAT (365 gg), data leak via whitelist, rottura link con quality gate, crescita dell'indice di ricerca client-side e prevenzione del takeover CNAME.

---

### 🟡 MAJORS

#### M1 — Configurazione Ricerca Zensical con Lingua Italiana (CUJ 3)
*   Aggiungere in `templates/mkdocs.yml` la configurazione esplicita del plugin search:
    ```yaml
    plugins:
      - search:
          lang: it
          separator: '[\s\-\.]+'
    ```
    per garantire la tokenizzazione e il folding dei caratteri accentati ("perche" -> "perché").

#### M2 — Allineamento Metriche Reali del Vault e Test Unitari Resilienti
*   Aggiornare i conteggi nel documento alle cifre effettive rilevate sul filesystem al 28 ottobre 2026: **136 pagine curate** e **65 sorgenti raw** (in 7 wiki).
*   Nei test unitari `tests/test_build_public_site.py`, utilizzare fixture sintetiche isolate (`tmp_path`) per testare la logica di trasformazione, evitando asserzioni rigide su conteggi scalari (`assert len(...) == 125`) che fallirebbero al primo commit sui contenuti.

#### M3 — Markup HTML `.wiki-grid` e Aggregazione Dati Homepage
*   Fornire nel design il template esatto per la Home:
    ```markdown
    <div class="wiki-grid">
      <a href="ai-llm/" class="wiki-card">
        <div class="wiki-card__title">AI & LLM</div>
        <div class="wiki-card__desc">Intelligenza artificiale, modelli locali e coding agents.</div>
        <div class="wiki-card__meta">39 note curate · Aggiornato: 2026-10-25</div>
      </a>
      <!-- ... altre 6 card ... -->
    </div>
    ```
*   Dichiarare la mappa statica `WIKI_INFO` nello script con i titoli italiani e le descrizioni, aggregando dinamicamente il numero di note e il `max(updated)`.

#### M4 — Specifica CLI `argparse` e Pulizia Preventiva di `staged_site`
*   Configurare `argparse` con `--source` (default `.`) e `--output` (default `staged_site`).
*   Imporre la pulizia iniziale: `if os.path.exists(output_dir): shutil.rmtree(output_dir)`.
*   Preservare la gerarchia delle directory: `staged_site/docs/<wiki>/<categoria>/<slug>.md` per evitare collisioni di percorsi e alimentare il breadcrumb a 4 livelli.

#### M5 — Riuso delle Primitive Esistenti da `tools/wiki.py`
*   Specificare che `tools/build_public_site.py` importa le funzioni collaudate di discovery e parsing da `tools.wiki` (`from wiki import wikis, pages, raws, split, PAGE_DIRS`), evitando duplicazioni di codice e disallineamenti con il linter.

#### M6 — Fallback Metadati Raw e Deduplicazione Footnote per Pagina
*   Gestire i campi opzionali del frontmatter raw: `author = fm.get("author") or "Autore sconosciuto"` e `title = fm.get("title") or filename`.
*   Deduplicare i richiami bibliografici nella singola pagina: se una fonte è citata più volte, riutilizzare lo stesso indice `[^cit-1]`, inserendo una sola riga bibliografica a piè di pagina.

#### M7 — Inclusione di `pytest` in CI e Parametri Esatti per `peaceiris/actions-gh-pages`
*   Aggiungere nel workflow CI:
    ```yaml
    - name: Run unit tests
      run: uv run pytest tests/ -v
    ```
*   Fornire lo snippet completo per il deploy cross-repo con `personal_token: ${{ secrets.PUBLIC_WIKI_DEPLOY_TOKEN }}` e `external_repository: andypanix/wiki-site`.

#### M8 — Apertura Link Esterni in Nuova Scheda e Foglio di Stile Completo
*   Formattare i collegamenti alle fonti con l'attributo Markdown per nuova scheda: `[^cit-N]: Fonte: [{title}]({source_url}){:target="_blank"} ({author}) ↗`.
*   Includere in `templates/extra.css` i selettori CSS concreti per `.wiki-grid`, `.wiki-card`, `.md-typeset` e le palette carta/inchiostro.

---

### 🟢 NITS

1.  **N1 — Fallback Wikilink senza Alias (`[[slug]]`)**: Se l'alias è omesso, il testo del link Markdown generato è il `title` formale presente in `slug_map[slug]["title"]`.
2.  **N2 — Localizzazione Categorie nel Breadcrumb**: Generare file index di categoria con titolo localizzato (`# Concetti`, `# Entità`, `# Confronti`, `# Query`) per garantire che `navigation.path` mostri i termini italiani.
3.  **N3 — Prevenzione Subdomain Takeover**: Aggiungere nei requisiti preliminari la verifica DNS del dominio tramite record TXT `_github-pages-challenge-andypanix` su GitHub prima dell'associazione CNAME.
4.  **N4 — Comandi Bash di Verifica**: Inserire nel Verification Plan i comandi di verifica pronti all'uso per airgap, assenza di residui `[[` o `^[` e dimensione dell'indice gzip.
5.  **N5 — Elenco File Locali da Dismettere**: Elencare esplicitamente i file obsoleti da rimuovere dalla sandbox locale (`Dockerfile`, `compose.yml`, `nginx.conf`, unità systemd).

---

## 4. Piano di Azione Raccomandato

Per finalizzare il documento di design e renderlo immediatamente cantierabile:
1. Applicare al file [`docs/designs/wiki-site.md`](file:///root/development/wiki-site/docs/designs/wiki-site.md) le correzioni ai 6 Blocker e agli 8 Major identificati.
2. Inserire la sezione formale `## Risks & Mitigations`.
3. Sincronizzare il testo con le metriche correnti del vault (136 note curate, 65 raw).
4. Procedere con l'avvio della fase di implementazione (Tappa 1 nel vault privato, Tappa 2 nel repo pubblico).
