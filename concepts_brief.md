# Pubblicazione Pubblica di Personal LLM Wiki su GitHub Pages con Zensical

* **Autore:** andypanix (con sintesi Hermes / EGM)
* **Data:** 2026-10-28

## Overview

Il progetto trasforma la base di conoscenza personale (`andypanix/personal-llm-wiki`, vault privato Obsidian con 7 wiki tematiche, 125 note curate e 64 sorgenti primarie) in un sito web statico pubblico ospitato su **GitHub Pages** (`wiki.andypanix.com`), eliminando completamente l'infrastruttura locale residente su Proxmox LXC 111 (container Docker, Nginx, timer systemd e dipendenze Node.js).

L'architettura adotta un modello serverless e deterministico Docs-as-Code:
1. L'ambiente LXC resta un puro client di authoring privato (Git + Obsidian + script CLI).
2. Il repository privato `personal-llm-wiki` ospita una GitHub Action che compila il sito a ogni push su `main`.
3. Uno script Python di staging (`tools/build_public_site.py`) opera da airgap logico: esclude categoricamente le trascrizioni integrali (`raw/`), i log interni e i prompt (`.agents/`), traduce i wikilink `[[slug|alias]]` in percorsi Markdown relativi standard, converte i marcatori di citazione `^[raw/...]` in note bibliografiche esterne con link al `source_url` originale (YouTube, articoli dell'autore) e blocca la compilazione con codice di errore 1 se rileva link orfani o interrotti.
4. Il generatore statico ad alte prestazioni **Zensical** compila le sole pagine curate applicando un tema minimalista **Typewriter** (ispirato a `obsidian-typewriter`, iA Writer ed Edward Tufte) tramite foglio di stile dedicato (`extra.css`).
5. L'output HTML statico compilato viene distribuito automaticamente su un repository pubblico dedicato (`andypanix/wiki-site`) tramite un Fine-Grained Personal Access Token (FG-PAT) con privilegi limitati, e servito su dominio personalizzato `wiki.andypanix.com` su radice `/`.

## Analysis

La sintesi multi-lente dei revisori Goldfish evidenzia i seguenti aspetti critici:

### 1. Eliminazione dell'Overhead Operativo Locale e Rischio Data-Leak (Skeptical Architect & SRE)
- **Zero demoni locali**: la migrazione a GitHub Pages elimina il rischio di OOM killer su LXC 111 (4 GB di RAM condivisi con l'agente), azzera le porte di rete esposte su LAN/Tailscale e rimuove la necessità di gestire certificati SSL locali.
- **Modello Default-Deny per la Privacy**: l'airgap di staging non deve basarsi su una blacklist di file da ignorare, ma su una **whitelist rigida** delle sole directory autorizzate (`entities/`, `concepts/`, `comparisons/`, `queries/`). Tutto ciò che non rientra in questo perimetro (`raw/`, `.agents/`, `log.md`, file di configurazione) viene ignorato alla radice.
- **Verifica del dominio `finanza-personale`**: confermato che la wiki contiene esclusivamente concetti educativi/metodologici per risparmiatori italiani e nessun dato patrimoniale privato o reale.

### 2. Ergonomia e Tipografia Typewriter (UX & Design)
- **Navigazione senza Top Tabs**: eliminare le schede orizzontali superiori evita il sovraccarico visivo causato da 7 etichette di dominio lunghe, concentrando la navigazione sulla sidebar ordinata e contestuale.
- **Homepage a 7 Card**: una griglia responsive con 7 tessere per i 7 domini (`ai-llm`, `devops`, `finanza-personale`, `homelab`, `software-engineering`, `tech-economy`, `tech-strategy`), ciascuna con abstract, conteggio note e data di aggiornamento.
- **Breadcrumb a 4 Livelli**: `Home › [Dominio Wiki] › [Tipo Pagina] › [Titolo]`, con stile monospaziato discreto e risalita rapida con un tocco a 375 px su smartphone.
- **Tipografia "Ink-on-Paper"**: contrasto morbido (fondo carta `#fcfbf9`, testo inchiostro antracite `#1a1a1c`, dark mode calda carbone `#161617`), corpo testo serif editoriale ad alta leggibilità (`Newsreader` o `Lora`) combinato con elementi strutturali e intestazioni in monospace (`JetBrains Mono`).

### 3. Integrità Sintattica e Trasformazione (Codebase Details & Implementation)
- **Risoluzione Wikilink**: sfruttando l'univocità globale degli slug certificata da `tools/wiki.py`, lo script calcola in-memory la mappa globale e trasforma ogni `[[slug|alias]]` nel percorso relativo `[alias](relpath.md)`.
- **Note di Provenienza e Citazioni Esterne**: i marcatori `^[raw/...]` vengono tradotti estraendo `source_url`, `author` e `title` dal frontmatter della raw corrispondente. La nota a piè di pagina diventa `[^cit-N]: Fonte: [{title}]({source_url}) ({author}) ↗`. I 15 marcatori `^[general-knowledge]` diventano note esplicative testuali senza link a vuoto.
- **Quality Gate Deterministico**: prima di invocare Zensical, lo script valida che tutti i percorsi Markdown bersaglio esistano fisicamente nella cartella di staging; se un link non risolve, la pipeline fallisce immediatamente impedendo la pubblicazione di pagine rotte.

### 4. Sicurezza CI/CD e Credenziali (Implementation Readiness)
- **Least Privilege (FG-PAT)**: il token ha scope limitato unicamente al repository pubblico di destinazione con permessi esclusivi `Contents: Read and Write`. Non ha permessi sul repo sorgente né su altri repository dell'account.
- **Manutenzione del Token**: i Fine-Grained PAT hanno una scadenza forzata massima di 365 giorni; è necessario annotare una procedura di rotazione annuale per evitare interruzioni improvvise della pipeline.
- **Compatibilità e Fallback Zensical**: Zensical accetta la medesima sintassi e configurazione `mkdocs.yml` di Material for MkDocs. Se Zensical dovesse presentare instabilità durante il build in CI, il fallback a `mkdocs-material` richiede la sola modifica di una riga nel workflow (`pip install mkdocs-material`).

## Goals vs. Non-Goals

### In-Scope Goals
* **G1 — Esclusione ermetica di `raw/` e `.agents/`**: Nessuna trascrizione integrale o prompt privato viene esposto nel repository pubblico o nell'HTML di output.
* **G2 — Risoluzione dei wikilink**: Tutti gli 804 wikilink del vault convertiti in link Markdown relativi standard verificati.
* **G3 — Citazioni esterne cliccabili**: Tutti i 1.492 marcatori `^[raw/...]` trasformati in note bibliografiche con collegamento diretto al `source_url` originale.
* **G4 — Blocco deterministico su link rotti**: Quality gate in pipeline che arresta il build in caso di link interni orfani o bersagli mancanti.
* **G5 — Zero infrastruttura locale**: Dismissione integrale di Docker, Nginx, reverse proxy e timer systemd sull'host Proxmox LXC 111.
* **G6 — Navigazione mobile zen a 375 px**: Homepage a 7 card, assenza di top tabs, breadcrumb a 4 livelli e stile Typewriter minimalista con palette carta/inchiostro via `extra.css`.
* **G7 — Hosting su dominio personalizzato**: Deploy automatico su GitHub Pages (`wiki.andypanix.com`) tramite FG-PAT.

### Out-of-Scope Non-Goals
* **NG1 — Nessuna pubblicazione del testo integrale delle 64 raw**: I file raw servono solo localmente come fonte e per estrarre `source_url`.
* **NG2 — Nessun runtime Node.js né container locale**: La compilazione avviene esclusivamente sui runner cloud effimeri di GitHub Actions.
* **NG3 — Nessun grafo interattivo globale né viste complesse**: La navigazione privilegia la lettura lineare e l'albero gerarchico senza grafi pesanti.
* **NG4 — Nessuna autenticazione o gestione accessi**: Il sito compilato finale è interamente pubblico e ad accesso libero.

## Key Decisions & Staging for Next Steps

1. **Airgap di Staging Whitelist**: Implementare `tools/build_public_site.py` nel repository privato `personal-llm-wiki` per eseguire copia controllata, trasformazione wikilink, risoluzione citazioni e validazione dell'integrità dei link.
2. **SSG Engine**: Adozione di **Zensical** configurato tramite `staged_site/mkdocs.yml`, con layout privo di schede orizzontali (`navigation.tabs: false`) e breadcrumb nativo (`navigation.path`).
3. **Estetica Typewriter**: Definizione di `docs/extra.css` con tipografia mista serif/monospace, larghezza colonna controllata (~700 px) e palette morbida carta/inchiostro con dark mode inclusa.
4. **Deploy Cross-Repo**: Creazione del repository pubblico `andypanix/wiki-site`, configurazione del record DNS CNAME `wiki.andypanix.com` verso `andypanix.github.io` e generazione del Fine-Grained PAT per il workflow `.github/workflows/deploy.yml`.
5. **Dismissione Host Locale**: Chiusura e rimozione delle configurazioni Docker/systemd/Nginx abbozzate nel repository `wiki-site` locale.
