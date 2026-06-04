# Convenzioni del vault — Modern Angular (appunti IT)

Vault Obsidian. Scopo: **studio personale / ripasso**. Fonte: `modern-angular_v2_0_0.pdf` (libro "Modern Angular", **2ª edizione** v2.0.0, aggiornata ad **Angular 22**). I numeri di pagina dei riferimenti `📖` puntano a questo PDF.

> [!info] Versioning del vault
> Gli appunti seguono la **v2.0.0** (Angular 22). Le feature introdotte con Angular 21.1/21.2/22 sono marcate con un callout `> [!info] Angular 22+` e il tag `angular-22` nel frontmatter → filtrabili in search/graph. Dove un vecchio snippet mostra ancora `@Injectable({ providedIn: 'root' })`, leggilo come [[service|@Service()]].

## Lingua
- **Prosa in italiano**, diretta, senza fronzoli.
- **Filename, titoli di sezione e nomi delle API in inglese** (es. `signal()`, `computed`, `effect`, `inject`, `httpResource`, `linkedSignal`, `withComponentInputBinding`). Non tradurre i termini tecnici Angular.
- Codice in inglese, **re-indentato** fedelmente alle convenzioni Angular/TS (l'estrazione dal PDF appiattisce l'indentazione).

## Struttura
```
00-index.md          home/MOC: mappa dei 19 capitoli + concetti cardine
capitoli/            1 nota-hub per capitolo (filename inglesi numerati a 2 cifre)
concetti/            note atomiche sui concetti cardine ricorrenti
assets/              eventuali immagini
_meta/               book-outline.txt + queste convenzioni (non sono appunti)
```
Modello **ibrido**: la nota-capitolo è l'hub e contiene il grosso; le note atomiche (`concetti/`) esistono solo per i concetti davvero centrali e ricorrenti, e si linkano da più capitoli.

## Naming file capitoli
`NN-kebab-title-inglese.md` con NN = numero capitolo a 2 cifre. Mappa:
- 01-getting-started
- 02-signal-based-components
- 03-reactive-design-with-signals
- 04-router-navigation-lazy-loading
- 05-state-management-services-signals
- 06-signal-forms
- 07-testing-with-vitest
- 08-sustainable-architectures
- 09-ngrx-signal-store
- 10-signal-queries-component-communication
- 11-directives-templates-containers
- 12-initialization-route-changes
- 13-hashbrown-agentic-ui
- 14-monorepos-libraries
- 15-internationalization
- 16-authentication-authorization
- 17-defer-ssr-hydration
- 18-micro-frontends
- 19-forensic-architecture-analysis

## Concetti atomici disponibili (in `concetti/`) — linkare con `[[nome]]`
signal · computed · effect · linked-signal · resource · untracked · reactive-context · equality-immutability · signal-input · signal-output · model-signal · two-way-binding · content-projection · signal-queries · inject · injection-context · providers · lightweight-store · service

## Template nota-capitolo
```markdown
---
capitolo: N
titolo: "<Titolo inglese>"
pagine: "<start>-<end>"
tags: [tipo/capitolo, <tematici>]
---
# NN · <Titolo inglese>
> 📖 cap.N · pp.<start>-<end> — *Modern Angular* v2.0.0

<Intro/contesto breve: cosa copre il capitolo e perché conta.>

## <Sezione>
> 📖 pp.<x>-<y>

<Prosa breve in italiano.>

```ts
// snippet commentato, re-indentato
```

> [!warning] Gotcha
> <punto insidioso>

> [!tip] Take-away
> <cosa ricordare>

Collegamenti: [[concetto]], [[NN-altro-capitolo]]

## 🔁 Ripasso lampo
1. <domanda>
2. <domanda>
3. <domanda>
(3-5 domande di autovalutazione)

**Take-away del capitolo:** <2-4 bullet con i punti chiave.>
```

## Template nota atomica (concetti/)
```markdown
---
titolo: "<nome API/concetto>"
tags: [tipo/concetto, <tematico>]
aliases: [<sinonimi/varianti>]
---
# <nome>

<Definizione concisa in 2-4 righe.>

```ts
// snippet minimo
```

> [!warning] Gotcha
> <insidia tipica>

**Usato in:** [[NN-capitolo]], [[NN-capitolo]]
```

## Tag controllati
- tipo: `tipo/capitolo`, `tipo/concetto`
- tematici: `signals`, `components`, `reactivity`, `routing`, `di`, `services`, `state-management`, `forms`, `testing`, `architecture`, `ngrx`, `directives`, `templates`, `lifecycle`, `ai`, `monorepo`, `i18n`, `security`, `ssr`, `micro-frontends`, `http`, `performance`
- versione: `angular-22` (note che trattano feature gated ad Angular 21.1/21.2/22)

## Diagrammi
Mermaid dove rende davvero: reactive flow / signal graph, gerarchia DI, child routes, flussi OAuth/OIDC, architecture matrix, unidirectional data flow. Non ovunque.

## Note operative sull'estrazione PDF
- Testo via `pypdf` (`python3` con `from pypdf import PdfReader`). Pagina i-esima = `r.pages[i-1].extract_text()` (0-based).
- Il codice estratto **perde indentazione** → ricostruirla.
- Ogni pagina ha un watermark `ciro.cu97@gmail.com <data>` → **ignorarlo**.
- I numeri pagina nel testo estratto sono +1 rispetto al numero stampato sulla pagina (offset frontmatter), ma `r.pages[N-1]` usa il numero del PDF reader = quello dell'outline. Usare l'outline in `book-outline.txt`.
