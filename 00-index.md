---
titolo: "Modern Angular — Appunti (IT)"
tags: [tipo/indice, moc]
---
# 🅰️ Modern Angular — Appunti

Vault di studio in italiano sul libro **_Modern Angular_ (v1.0.4)**. Appunti per capire e ripassare: prosa in italiano, termini/API/codice in inglese. Ogni nota-capitolo è un **hub** con riferimenti alle pagine del PDF e una sezione **🔁 Ripasso lampo** in fondo; i concetti cardine ricorrenti sono note atomiche cross-linkate.

> [!tip] Come usare questo vault
> - Apri la cartella `ma/` come **vault in Obsidian** per navigare i link `[[...]]` e il *graph view*.
> - Parti dai **Fondamenti** in ordine; gli altri blocchi sono abbastanza autonomi.
> - Per ripassare: leggi i **Take-away** e rispondi al **🔁 Ripasso lampo** di ogni capitolo.
> - I `> 📖 pp.x-y` rimandano alle pagine del PDF per approfondire.

## 📚 Capitoli

### Fondamenti
- [[01-getting-started|01 · Getting Started with Angular]] — pp.18-30
- [[02-signal-based-components|02 · Signal-Based Components]] — pp.31-70
- [[03-reactive-design-with-signals|03 · Reactive Design with Signals]] — pp.71-93

### Routing, servizi e stato
- [[04-router-navigation-lazy-loading|04 · Navigation & Lazy Loading with the Router]] — pp.94-125
- [[05-state-management-services-signals|05 · State Management with Services & Signals]] — pp.126-148
- [[09-ngrx-signal-store|09 · State Management with NgRx Signal Store]] — pp.243-286
- [[12-initialization-route-changes|12 · Initialization & Route Changes]] — pp.331-345

### Form e testing
- [[06-signal-forms|06 · Signal Forms]] — pp.149-192
- [[07-testing-with-vitest|07 · Modern Testing with Vitest]] — pp.193-219

### Componenti avanzati
- [[10-signal-queries-component-communication|10 · Signal Queries & Component Communication]] — pp.287-300
- [[11-directives-templates-containers|11 · Directives, Templates, and Containers]] — pp.301-330

### Architettura e scaling
- [[08-sustainable-architectures|08 · Sustainable Architectures for Modern Angular]] — pp.220-242
- [[14-monorepos-libraries|14 · Monorepos & Reusable Libraries]] — pp.373-388
- [[18-micro-frontends|18 · Micro Frontends: Scaling Across Multiple Teams]] — pp.423-443
- [[19-forensic-architecture-analysis|19 · Analyzing Your Architecture with Forensic Techniques]] — pp.444-453

### Argomenti trasversali e avanzati
- [[13-hashbrown-agentic-ui|13 · Agentic UI & AI Assistants with Hashbrown]] — pp.346-372
- [[15-internationalization|15 · Internationalization]] — pp.389-400
- [[16-authentication-authorization|16 · Authentication & Authorization]] — pp.401-410
- [[17-defer-ssr-hydration|17 · Defer, SSR & Hydration]] — pp.411-422

## 🧩 Concetti cardine

**Reattività**
[[signal]] · [[computed]] · [[effect]] · [[linked-signal]] · [[resource]] · [[untracked]] · [[reactive-context]] · [[equality-immutability]]

**Componenti & comunicazione**
[[signal-input]] · [[signal-output]] · [[model-signal]] · [[two-way-binding]] · [[content-projection]] · [[signal-queries]]

**Dependency Injection & stato**
[[inject]] · [[injection-context]] · [[providers]] · [[lightweight-store]]

## 🏷️ Tag utili
`tipo/capitolo` · `tipo/concetto` · `signals` · `reactivity` · `components` · `routing` · `di` · `services` · `state-management` · `ngrx` · `forms` · `testing` · `architecture` · `directives` · `templates` · `ssr` · `security` · `micro-frontends` · `i18n` · `ai` · `monorepo`

---
*Fonte: `modern-angular_v1_0_4.pdf`. Convenzioni del vault in [[conventions]] (cartella `_meta/`).*
