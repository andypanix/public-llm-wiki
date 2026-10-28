# Personal LLM Wiki

> Curated personal knowledge base spanning 7 technical domains, built with a Docs-as-Code philosophy.

🌐 **Live Website**: [https://wiki.andypanix.com/](https://wiki.andypanix.com/)

---

## Overview

This repository hosts the public compiled static website for **Personal LLM Wiki**, an Obsidian-based personal knowledge base organized into 7 specialized technical wikis:

1. **Software Engineering** — Modern software development lifecycle (SDLC), agentic workflows, architecture patterns, and engineering practices.
2. **Finanza Personale & Previdenza** — Italian personal finance, ETF indexing, retirement frameworks, and wealth planning.
3. **Hardware & Homelab** — Proxmox VE virtualization, networking, storage topologies, and self-hosted infrastructure.
4. **Intelligenza Artificiale** — Foundation models, prompt engineering, agentic swarms, and local inference runtimes.
5. **Produttività & Sistemi Operativi** — Linux CLI tooling, knowledge management workflows, automation, and system administration.
6. **Longevità & Salute** — Health protocols, biometric tracking, biomarker analysis, and longevity research.
7. **Filosofia & Crescita Personale** — Mental models, decision frameworks, and cognitive hygiene.

---

## Architecture & Pipeline

The content is authored privately in Obsidian and transformed through an automated, privacy-first CI/CD pipeline:

```
Private Vault (Obsidian)
        │
        ├── Linting & Validation (Quality Gate)
        ├── Airgap Filter (Excludes raw sources & agent artifacts)
        ├── Link Transformer (Resolves relative links & external provenance)
        └── Static Site Generation (Zensical / Material)
                        │
                        ▼
Public Repository (andypanix/public-llm-wiki)
        │
        └── GitHub Pages CDN ──► https://wiki.andypanix.com/
```

- **Source**: Curated notes from [`andypanix/personal-llm-wiki`](https://github.com/andypanix/personal-llm-wiki) (private).
- **Airgap & Privacy**: Deterministic whitelist filtering ensures raw transcripts and internal agent instructions remain completely private.
- **SSG Engine**: Built with [Zensical](https://github.com/zensical/zensical) using a custom Typewriter typography palette (`Newsreader` serif + `JetBrains Mono`).
- **Hosting**: Served via GitHub Pages on a custom domain with automated Let's Encrypt TLS.

---

## Branches

- `main`: Repository documentation, architectural design records, and issue tracking.
- `gh-pages`: The compiled static distribution (HTML, CSS, assets, and pre-computed client-side search index) deployed automatically on every vault update.

---

## License

The written content and documentation are published for personal reference and educational purposes.
