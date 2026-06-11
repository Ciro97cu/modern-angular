---
capitolo: 14
titolo: "Monorepos & Reusable Libraries"
pagine: "384-399"
tags: [tipo/capitolo, monorepo, architecture, angular-22]
---
# 14 · Monorepos & Reusable Libraries
> 📖 cap.14 · pp.384-399 — *Modern Angular* v2.0.0

Un singolo progetto Angular basta finché il codice è piccolo: appena cresce, i team raggiungono i propri limiti. La risposta sono i **monorepo**, un'unica repo che raggruppa più applicazioni e **librerie riutilizzabili**. Le librerie servono a sotto-strutturare un sistema grande (vedi i moduliths del [[08-sustainable-architectures|cap.8]]) e, quando servono in altri progetti, si **pubblicano via npm**.

Il capitolo parte dalla **Angular CLI** (monorepo manuale, build e publish) e poi passa a **Nx**, più potente: build incrementali, module boundaries, cache distribuita e parallelizzazione.

> [!info] Monorepo ≠ solo pubblicazione
> I monorepo non servono solo a creare librerie da pubblicare. Nei progetti molto grandi suddividono la soluzione complessiva in app e librerie più piccole, più facili da governare e mantenere. In questo scenario le librerie **non vanno pubblicate su npm**: sono consumate localmente dalle app del monorepo stesso. Termine alternativo a monorepo: **workspace**.

## Angular CLI-based Repos — Creating a Monorepo
> 📖 pp.384-385

Dal punto di vista della CLI un monorepo è un normale progetto Angular. La differenza è che **non vogliamo la cartella `src` centrale**: di norma `ng new` la genera per ospitare tutto il codice dell'app, ma qui vogliamo dividere il progetto in app e librerie separate, quindi la disattiviamo con `--create-application false`. App e librerie si aggiungono poi dentro `projects/`.

```bash
# Crea il monorepo senza la cartella src centrale
ng new logger --create-application false
cd logger

# Aggiunge librerie e applicazioni (usa i default suggeriti dalla CLI)
ng g lib util-logger
ng g app playground-app
```

Struttura risultante (Listing 14-1): le app finiscono in `projects/<app>/src/app`, le lib in `projects/<lib>/src/lib` con un `public-api.ts`.

```mermaid
graph TD
  R[logger/ - monorepo root] --> P[projects/]
  P --> A[playground-app/src/app]
  P --> L[util-logger/src/lib + public-api.ts]
  R --> D[dist/ - output build]
  R --> T[tsconfig.json - path mappings]
```

> [!warning]
> In un monorepo ogni comando CLI deve indicare **a quale subproject si riferisce** con `--project` (o come argomento posizionale per serve/build):
> ```bash
> ng generate component MyComponent --project playground-app
> ng generate component OtherComponent --project util-logger
> ng serve playground-app -o
> ng build playground-app
> ng build util-logger
> ```

## Structure of Libraries
> 📖 pp.386-388

Come le app, le librerie sono file TypeScript (componenti, servizi, ...) e vivono sotto `src/lib`. Una lib contiene anche `ng-package.json`, `package.json` e i `tsconfig.lib*.json` (Listing 14-3). Generiamo un servizio `Logger` (da non confondere con l'`UtilLoggerService` che la CLI crea quando genera la libreria omonima):

```bash
ng g service logger --project util-logger
```

> [!info] Angular 22+
> Il servizio usa il decoratore `@Service()` introdotto in Angular 22. `@Service()` senza opzioni equivale al vecchio `@Injectable({ providedIn: 'root' })`: il servizio è iniettabile e registrato automaticamente nello scope root. Dettagli in [[service]].

```ts
// projects/util-logger/src/lib/logger.ts
import { Service } from '@angular/core';

@Service()
export class Logger {
  log(message: string): void {
    const date = new Date().toISOString().substr(0, 10);
    console.log(`[${date}] ${message}`);
  }
}
```

Perché il servizio sia usabile da fuori va **esportato dal `public-api.ts`**, l'entry point della libreria:

```ts
// projects/util-logger/src/public-api.ts
[...]
export * from './lib/logger';
```

Il nome del file di entry point si configura in `ng-package.json` (Listing 14-6), insieme a `dest`, la cartella in cui finisce il pacchetto durante la build:

```json
{
  "$schema": "../../node_modules/ng-packagr/ng-package.schema.json",
  "dest": "../../dist/util-logger",
  "lib": {
    "entryFile": "src/public-api.ts"
  }
}
```

Prima di pubblicare si cura il `package.json` della libreria (Listing 14-7):

```json
{
  "name": "@my/util-logger",
  "version": "0.0.1",
  "peerDependencies": {
    "@angular/common": ">= 11.0.0",
    "@angular/core": ">= 11.0.0"
  },
  "dependencies": {
    "tslib": "^2.0.0"
  }
}
```

- **`name`** — nome con cui il pacchetto è pubblicato e installato. Può avere uno **scope** introdotto con `@` (es. `@my`): mette ordine indicando da quale area arriva il pacchetto (spesso nome azienda o progetto) e fa sì che solo chi ha i diritti possa pubblicare sotto quello scope.
- **`version`** — a ogni pubblicazione serve un numero **ancora disponibile** (non già usato).
- **`peerDependencies`** — dipendenze che il consumer deve installare a parte per usare il pacchetto; supportano range con espressioni booleane. Di regola **da preferire** alle `dependencies` perché non impongono una versione precisa al consumer.
- **`dependencies`** — installate insieme al pacchetto.

> [!warning]
> A parte `tslib` (richiesto da TypeScript), l'uso di una `dependency` "convenzionale" va **autorizzato esplicitamente** in `ng-package.json` con `allowedNonPeerDependencies`, altrimenti la CLI non la consente (protezione da uso accidentale):
> ```json
> {
>   "allowedNonPeerDependencies": ["sha-256-js"]
> }
> ```

## Trying Out the Library in the Monorepo
> 📖 pp.389-390

I building block delle librerie si testano come tutti gli altri (vedi [[07-testing-with-vitest|cap.7]]), indicando il nome della lib:

```bash
ng test util-logger
```

Per dare agli altri subproject del monorepo accesso alle librerie, la CLI imposta dei **path mapping** in `tsconfig.json` (root del monorepo). Di default puntano alla cartella `dist/` (Listing 14-10), il che obbligherebbe a **ricompilare la lib dopo ogni modifica** — tedioso ed error-prone. Meglio farli puntare al **sorgente** (al `public-api`), riusando se possibile lo scope di `package.json` (Listing 14-11):

```jsonc
// tsconfig.json (root) — meglio puntare al sorgente, non a dist/
"paths": {
  "@my/util-logger": [
    "projects/util-logger/src/public-api"
  ]
}
```

Tutto ciò che il `public-api` esporta diventa importabile dagli altri subproject tramite il nome mappato — niente più path relativi lunghi e illeggibili:

```ts
import { Logger } from '@my/util-logger';
```

> [!warning]
> **Dentro la libreria stessa** non usare il proprio path mapping, né importare dal proprio `public-api`: crea **riferimenti circolari**. Dopo aver cambiato i path mapping, **riavvia l'IDE**.

Prova: si inietta `Logger` nell'`App` di `playground-app` (qui in stile costruttore classico):

```ts
// projects/playground-app/src/app/app.component.ts
import { Component } from '@angular/core';
import { Logger } from '@my/util-logger';   // Add

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css'],
})
export class AppComponent {
  title = 'playground-app';
  constructor(private logger: Logger) {      // Add
    logger.log('Manfred was here!');
  }
}
```

Lanciando `ng serve playground-app -o`, il messaggio di test compare nella console JavaScript.

## Building and Publishing the npm Package
> 📖 pp.390-392

Prima di pubblicare, assicurati che il `package.json` della lib (es. `projects/util-logger/package.json`) abbia un **numero di versione univoco**, altrimenti la pubblicazione fallisce. Poi build e publish:

```bash
# Build di produzione → il pacchetto finisce in dist/util-logger
ng build util-logger --prod

# Pubblica sul registry npm pubblico
npm publish dist/util-logger
```

In alternativa puoi usare un **registry npm privato** (in-house). Se non ne hai uno, **Verdaccio** è un server npm gratuito e leggerissimo, eseguibile in locale per i test oppure come servizio Windows / daemon Linux su un server:

```bash
npm i -g verdaccio
verdaccio
# Verdaccio stampa il suo indirizzo, di default http://localhost:4873

# Al primo avvio, crea un utente npm
npm adduser --registry http://localhost:4873

# Pubblica sul registry privato
npm publish dist/util-logger --registry http://localhost:4873
```

Aprendo Verdaccio nel browser dovresti vedere il pacchetto pubblicato (Figure 14-1).

> [!tip]
> Per non ripetere `--registry` a ogni `npm publish`, crea un `.npmrc` (formato .ini) nella root del progetto. Lo puoi mettere **sotto version control**, così vale per tutto il team:
> ```ini
> registry=http://localhost:4873
> # oppure un registry per singolo scope:
> @my:registry=http://localhost:4873
> ```

## Consuming the npm Package
> 📖 p.392

Il consumer installa il pacchetto con `npm install` nella propria app Angular e lo usa come visto sopra con `playground-app`:

```bash
npm install @my/util-logger --registry http://localhost:4873
# Se ha già configurato il default registry, --registry si può omettere
```

> [!info] Fallback al registry pubblico
> Se si richiede un pacchetto che Verdaccio non conosce, di default delega al **registry npm pubblico** e lo recupera da lì.

## Faster Builds and More Convenience with Nx
> 📖 pp.393-394

Il limite della soluzione CLI: gli sviluppatori devono **sapere quali app sono cambiate** e lanciare a mano la build giusta; e il build server, per sicurezza, finisce comunque per ricostruire e testare tutto. Meglio lasciare che sia il **tooling** a capire cosa è cambiato — p.es. calcolando un **hash** dei file sorgente che confluiscono in ogni app: se l'hash cambia, quell'app va ricostruita o ritestata.

**Nx** implementa questa idea e aggiunge molto altro. Oltre ad Angular supporta React e backend Node.js, e integra senza setup manuale i tool comuni dello sviluppo web: i testing tool Jest, Cypress e Playwright, il server npm Verdaccio e Storybook per la documentazione interattiva dei componenti. Per gli sviluppatori Angular è naturale: la **Nx CLI** si usa come la Angular CLI, basta sostituire `ng` con `nx` e gli argomenti restano in gran parte gli stessi (`nx build`, `nx serve`, `nx g app`, `nx g lib`, ...).

```bash
npm i -g nx

# Crea un nuovo workspace Nx (rispondi Enter a tutte le domande per i default)
npx create-nx-workspace@latest my-project
```

> [!info] Migrare o ricreare?
> Esistono script per **migrare** workspace CLI a Nx, ma potrebbero non attivare tutte le feature di Nx. L'autore consiglia di **creare un nuovo workspace Nx** e, se serve, copiarci dentro il sorgente esistente.

Convenzione Nx: app in `apps/`, lib in `libs/`. Al prompt del generator scegli `@nx/angular:application` per le app e `@nx/angular:library` per le lib:

```bash
nx g app apps/appName
nx g lib libs/libName
```

```mermaid
graph TD
  W[Nx workspace] --> APPS[apps/]
  W --> LIBS[libs/]
  LIBS --> F[type:feature]
  LIBS --> U[type:ui]
  LIBS --> D[type:data]
  LIBS --> UT[type:util]
  APPS --> F
```

> [!warning]
> Di default Nx crea librerie **internal**: non pubblicabili, usate solo nel monorepo. Per pubblicarne una servono `--publishable` e `--importPath` (quest'ultimo definisce il nome del pacchetto npm usato negli import):
> ```bash
> nx g lib libs/libName --publishable --importPath @my-company/lib-name
> ```

Il comando `nx graph` mostra il **grafo delle dipendenze** tra app e librerie (Figure 14-2).

## Module Boundaries
> 📖 pp.395-396

Come Sheriff (vedi [[08-sustainable-architectures]]), anche Nx permette di definire regole su **chi può dipendere da cosa** — i cosiddetti **module boundaries** — ma in Nx sono sempre definiti **a livello di libreria e applicazione** (non per cartella). Si configura la regola ESLint `@nx/enforce-module-boundaries` in `eslint.config.js`, dichiarando i `depConstraints` per `sourceTag` (Listing 14-13):

```js
// eslint.config.js
[...]
'@nx/enforce-module-boundaries': [
  'error',
  {
    depConstraints: [
      {
        sourceTag: 'domain:miles',
        onlyDependOnLibsWithTags: ['domain:miles', 'domain:shared'],
      },
      {
        sourceTag: 'domain:shared',
        onlyDependOnLibsWithTags: ['domain:shared'],
      },
      {
        sourceTag: 'type:app',
        onlyDependOnLibsWithTags: ['type:feature', 'type:ui', 'type:data', 'type:util'],
      },
      {
        sourceTag: 'type:feature',
        onlyDependOnLibsWithTags: ['type:ui', 'type:data', 'type:util'],
      },
      {
        sourceTag: 'type:ui',
        onlyDependOnLibsWithTags: ['type:data', 'type:util'],
      },
      {
        sourceTag: 'type:data',
        onlyDependOnLibsWithTags: ['type:util'],
      },
      {
        sourceTag: 'type:util',
        onlyDependOnLibsWithTags: [],
      },
    ],
  },
],
```

Sono le stesse restrizioni per **layer e domini** introdotte con Sheriff nel [[08-sustainable-architectures|cap.8]]. I tag si dichiarano nel `project.json` della lib/app (Nx lo crea con un array `tags` vuoto da estendere, Listing 14-14):

```jsonc
// libs/miles/feature-manage/project.json
[...]
"tags": ["domain:miles", "type:feature"],
[...]
```

Violare le regole (es. una UI lib che accede a una feature) genera un **errore di linting**, eseguibile da IDE o da riga di comando:

```bash
nx lint <project-name>

# Per più progetti o per tutti:
nx run-many -t lint -p flights,miles
nx run-many -t lint --all
```

## Nx with Sheriff and Detective
> 📖 p.397

I module boundaries di Nx lavorano **per libreria e applicazione**. Se servono regole **più granulari, per-cartella**, si combina **Sheriff** con Nx; **Detective**, sempre con Nx, aiuta a **visualizzare** i setup folder-based. Entrambi i tool sono trattati nel [[08-sustainable-architectures|cap.8]] e nel [[19-forensic-architecture-analysis|cap.19]].

Collegamenti: [[08-sustainable-architectures]] (Sheriff, Detective, moduliths) · [[19-forensic-architecture-analysis]].

## Incremental Builds with Nx
> 📖 p.397

Gli stessi dati del grafo delle dipendenze alimentano le **build incrementali** che Nx offre out of the box. Con `nx build`, se i sorgenti che confluiscono nell'app non sono cambiati, il risultato arriva **subito dalla cache locale** (cartella `.nx`, ignorata dal `.gitignore` del progetto):

```bash
nx build miles

# Ricostruire più progetti o tutti (la cache scatta comunque se nulla è cambiato):
npx nx run-many -t build -p flights,miles
npx nx run-many -t build --all
```

Anche unit test, E2E e linting sono incrementali allo stesso modo; Nx va oltre e li **cacha a livello di libreria**: dividere l'app in più librerie migliora le performance.

> [!warning]
> Lo stesso sarebbe possibile per `nx build` rendendo le singole lib **buildable** (`nx g lib myLib --buildable`), ma in pratica **raramente porta vantaggi**: i rebuild incrementali a livello di applicazione sono preferibili.

## Distributed Cache with Nx Cloud
> 📖 p.397

Di default la cache è **locale**. Per andare oltre, una **cache distribuita** condivisa da tutto il team e dal build server permette di beneficiare anche delle build già fatte da altri. La offre **Nx Cloud**, add-on commerciale del Nx gratuito (self-hostabile se non si possono usare provider cloud). Per connettere il workspace basta un comando:

```bash
npx nx connect-to-nx-cloud
```

## Even Faster: Parallelization with Nx Cloud
> 📖 pp.398-399

Per accelerare ulteriormente, Nx Cloud **parallelizza** i singoli task di build. Anche qui il grafo delle dipendenze è il vantaggio: stabilisce l'ordine in cui i task devono essere eseguiti e quali si possono parallelizzare. Si usano nodi distinti nel cloud: un **main node** coordina, più **worker node** eseguono i singoli task in parallelo. Nx può perfino generare gli **script CI** che avviano i nodi e gli assegnano i task:

```bash
# Genera un workflow per GitHub (supporta anche --ci circleci e --ci azure)
nx generate @nx/workspace:ci-workflow --ci github
```

Gli script specificano il numero di worker node e di processi paralleli per worker, e dividono i comandi in tre gruppi: comandi sequenziali di **inizializzazione**, comandi paralleli sul **main node**, comandi paralleli sugli **agent**. Si attivano quando cambia il branch `main` della repo, per push diretto o per merge di una PR (Figure 14-3).

> [!tip]
> Lo stesso **dependency graph** che disegna `nx graph` è il motore di tutto Nx: module boundaries, build/test/lint **incrementali** con cache, cache **distribuita** e **parallelizzazione**. Più spezzi in librerie ben confinate, più ne guadagni.

## 🔁 Ripasso lampo

**1.** Perché si crea il monorepo con `ng new logger --create-application false`? Come si generano poi app e librerie?
> [!success]- Risposta
> Senza `--create-application false` la CLI genererebbe la cartella `src` centrale con un'app singola; in un monorepo non serve perché il codice va diviso in subproject separati. App e librerie si aggiungono poi con `ng g app <nome>` e `ng g lib <nome>`, e finiscono in `projects/`.

**2.** Cos'è il `public-api.ts` e dove si configura l'entry point di una libreria?
> [!success]- Risposta
> È l'**entry point** della libreria: solo ciò che vi viene esportato (es. `export * from './lib/logger'`) è visibile dai consumer. Il nome del file di entry point si configura in `ng-package.json`, nel campo `lib.entryFile` (insieme a `dest`, la cartella di output della build).

**3.** Perché far puntare i path mapping al **sorgente** anziché a `dist/`? E perché non usare il proprio path mapping **dentro** la libreria?
> [!success]- Risposta
> Puntando a `dist/` dovresti **ricompilare la lib dopo ogni modifica** (tedioso ed error-prone); puntando al `public-api` sorgente i cambiamenti sono visibili subito. Dentro la libreria stessa non si usa il proprio path mapping né si importa dal proprio `public-api` perché crea **riferimenti circolari**.

**4.** Differenza tra `peerDependencies` e `dependencies` in una lib, e a cosa serve `allowedNonPeerDependencies`?
> [!success]- Risposta
> Le `peerDependencies` vanno installate **separatamente dal consumer** e supportano range di versioni, quindi non impongono una versione precisa (da preferire). Le `dependencies` sono installate **insieme** al pacchetto. A parte `tslib`, ogni `dependency` "convenzionale" va elencata esplicitamente in `allowedNonPeerDependencies` (in `ng-package.json`), altrimenti la CLI non la consente.

**5.** Come si pubblica su un registry privato (Verdaccio) e come si evita di ripetere `--registry`?
> [!success]- Risposta
> Si avvia Verdaccio (`verdaccio`, di default su `http://localhost:4873`), si crea l'utente con `npm adduser --registry ...` e si pubblica con `npm publish dist/util-logger --registry ...`. Per non ripetere `--registry` si crea un `.npmrc` (formato .ini) nella root con `registry=...` (o `@my:registry=...` per singolo scope), versionabile per tutto il team.

**6.** Su quale livello operano i module boundaries di Nx e quando si combina con Sheriff?
> [!success]- Risposta
> Operano **per libreria e applicazione** (regola ESLint `@nx/enforce-module-boundaries` + tag `sourceTag`/`onlyDependOnLibsWithTags` nei `depConstraints`). Quando servono regole **più granulari, per-cartella**, si combina **Sheriff** con Nx; **Detective** aiuta a visualizzare i setup folder-based.

**7.** Cosa rende possibile, in Nx, build incrementali, cache distribuita e parallelizzazione?
> [!success]- Risposta
> Il **dependency graph** (lo stesso che disegna `nx graph`). Permette di capire quali progetti sono cambiati (build/test/lint incrementali con cache, locale o distribuita via **Nx Cloud**) e quali task possono girare in parallelo (parallelizzazione su main node + worker node).

**In sintesi:**
- Un progetto Angular può essere un **monorepo/workspace** che raggruppa più app eseguibili e **librerie riutilizzabili**; le lib si pubblicano come pacchetti npm oppure si consumano solo localmente.
- Con la **Angular CLI**: `ng new --create-application false`, `ng g lib`/`ng g app`, export dal `public-api.ts`, path mapping al sorgente, `ng build --prod` + `npm publish` (anche su Verdaccio).
- **Nx** dà più comodità e build più veloci: stessa ergonomia della CLI (`nx` al posto di `ng`), **module boundaries** via ESLint + tag, integrazione con i tool comuni, e build/test/lint **incrementali** basati sul dependency graph.
- **Nx Cloud** aggiunge **cache distribuita** e **parallelizzazione** dei task su main/worker node, con script CI generati.
