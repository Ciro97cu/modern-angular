---
capitolo: 17
titolo: "Defer, SSR & Hydration"
pagine: "422-434"
tags: [tipo/capitolo, ssr, performance, angular-22]
---
# 17 · Defer, SSR & Hydration
> 📖 cap.17 · pp.422-434 — *Modern Angular* v2.0.0

Le SPA hanno ottime performance a runtime, ma il **primo caricamento** è più lento di un sito classico server-rendered: il browser deve scaricare ed eseguire molto JavaScript prima di poter dipingere la pagina, quindi il *First Meaningful Paint* (FMP) arriva tardi. Per app interne può andare bene; per shop e landing pubbliche è un problema (più secondi = più bounce).

Il capitolo presenta tre meccanismi complementari:
- **Deferrable views** (`@defer`): rimanda le regioni non critiche.
- **SSR** (server-side rendering): invia HTML già pronto da dipingere.
- **Hydration**: rende interattivo quell'HTML sul client, eventualmente a step incrementali.

## Deferrable Views — `@defer`
> 📖 pp.422-423

Non tutte le aree di una pagina contano allo stesso modo. Sulla ricerca voli, form e lista risultati sono primari; pannelli opzionali (auto, hotel, tenda) sono secondari. Caricarne il codice subito appesantisce il bundle iniziale inutilmente. Con `@defer` si avvolge la regione: Angular ne **splitta il codice in un bundle separato** e lo carica solo allo scattare di un trigger. Nel frattempo `@placeholder` fornisce contenuto alternativo, così il layout non "salta".

```html
<!-- .../flight-search/flight-search.html -->
@defer (on hover) {
  <app-car-pane />
} @placeholder {
  <app-placeholder />
}
```

Oltre a `@placeholder`, `@defer` supporta `@loading` (mostrato mentre si scarica il bundle) e `@error` (se il caricamento fallisce). Per evitare flicker:
- `minimum` su `@placeholder`/`@loading` → durata minima di visualizzazione.
- `after` su `@loading` → ritarda lo spinner finché il caricamento non supera la soglia.

```html
@defer (on viewport) {
  <app-recommendations />
} @placeholder (minimum 300ms) {
  <app-skeleton />
} @loading (after 150ms; minimum 300ms) {
  <app-loading-spinner />
} @error {
  <p>Could not load recommendations.</p>
}
```

> [!tip] Take-away
> `@defer` non è per nulla che serva alla prima interazione utile. Differire navigazione core o azioni primarie costringe l'utente a uno stato di loading proprio quando ne ha bisogno: danneggia la *perceived performance*.

### Triggers
> 📖 pp.423-424

Insieme fisso di trigger su `@defer`:

| Trigger | Quando scatta |
|---|---|
| `on idle` | il browser non ha task critici pendenti (**default**) |
| `on viewport` | il placeholder (o un elemento referenziato) entra nell'area visibile |
| `on interaction` | l'utente inizia a interagire col placeholder/elemento |
| `on hover` | il mouse passa sopra il placeholder/elemento |
| `on immediate` | appena possibile dopo il load della pagina |
| `on timer(duration)` | dopo una durata, es. `on timer(5s)` |
| `when (condition)` | quando la condizione è vera, es. `when (userName() !== null)` |

`on viewport`, `on interaction` e `on hover` di default **richiedono un `@placeholder`**. In alternativa possono riferirsi a un altro elemento via template variable: `@defer (on viewport(recommendations)) { … }` con un `#recommendations` su qualche elemento.

**Prefetch**: si può anticipare il download del bundle senza inserire subito il contenuto.

```html
@defer (on viewport; prefetch on immediate) {
  <app-recommendations />
} @placeholder {
  <app-skeleton />
}
<!-- scarica appena possibile, ma scambia il contenuto solo quando il placeholder entra nel viewport -->
```

Collegamenti: il code-splitting/lazy loading è lo stesso tema del [[04-router-navigation-lazy-loading|cap.4]] applicato a livello di template.

## SSR & Hydration
> 📖 pp.424-425

Per migliorare il primo paint di soluzioni pubbliche si rende la SPA **sul server**, che consegna una pagina HTML completa: il chiamante vede il contenuto prima. Una volta caricati i bundle JS, la pagina diventa interattiva — questo processo si chiama **hydration**.

### Adding Server-Side Rendering
> 📖 p.424

```bash
ng add @angular/ssr
# oppure, in fase di creazione:
ng new my-app --ssr
```

La configurazione vive in `app.config.ts`:

```ts
// src/app/app.config.ts (Angular 22)
import { provideHttpClient } from '@angular/common/http';
import {
  provideClientHydration,
  withEventReplay,
} from '@angular/platform-browser';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(),            // FetchBackend di default (v22): niente withFetch()
    provideClientHydration(
      withEventReplay(),            // incremental hydration è già di default (v22)
    ),
  ],
};
```

> [!info] Angular 22+ · FetchBackend di default
> Da **Angular 22** `HttpClient` usa il **`FetchBackend`** di default → `withFetch()` non serve più (è **deprecato**). La Fetch API è ovunque, è Promise-based e funziona uguale in browser e SSR. Unica cosa che non offre: l'**upload progress** → se ti serve `reportUploadProgress`, torna a XHR con `provideHttpClient(withXhr())`.
> **Pre-22:** il default era `XhrBackend`; la fetch andava attivata con `provideHttpClient(withFetch())`, cruciale in SSR/hydration. `ng update` inserisce `withXhr()` dove l'app dipendeva dal vecchio comportamento.

> [!tip] Take-away
> Con la config standard di Angular 22 basta `provideHttpClient()` + `provideClientHydration(withEventReplay())`: fetch backend e hydration incrementale sono **default**.

Collegamenti: [[providers]] · le provider function (`provide*`/`with*`).

### Incremental Hydration
> 📖 pp.425-426

L'hydration **non deve avvenire tutta in una volta**. I bundle si richiedono solo quando servono — il più tardi possibile, idealmente mai per le regioni che l'utente non tocca. Le parti più importanti diventano interattive prima; le meno importanti dopo. Si abbina naturalmente a `@defer`: le regioni da idratare on-demand si marcano con `@defer` e la clausola **`hydrate`** dice *quando* idratare.

> [!info] Angular 22+ · Default invertito
> Da **Angular 22** `provideClientHydration()` abilita l'incremental hydration **in automatico**: `withIncrementalHydration()` non serve più (resta solo per casi di opt-out). Per disattivarla usa la nuova feature **`withNoIncrementalHydration()`**.
> **Pre-22 (Angular 19–21):** era opt-in, da attivare con `provideClientHydration(withIncrementalHydration())`. Al bump `ng update` aggiunge `withNoIncrementalHydration()` dove la feature non era richiesta, preservando il comportamento.

```html
<!-- .../flight-search/flight-search.html -->
@defer (on hover; hydrate on hover) {
  <app-car-pane />
} @placeholder {
  <app-placeholder />
}
```

Fino all'hydration, **il markup server-rendered fa da placeholder** per quella regione. La clausola `hydrate` riguarda solo il render iniziale: quando l'utente naviga a una nuova route sul client, Angular renderizza quella route interamente sul client e l'incremental hydration non si applica.

Trigger di hydration disponibili:

| Trigger | Quando idrata |
|---|---|
| `hydrate on idle` | quando il browser è idle |
| `hydrate on viewport` | quando il contenuto entra nell'area visibile |
| `hydrate on interaction` | quando l'utente interagisce con l'elemento |
| `hydrate on hover` | quando il mouse è sopra l'area |
| `hydrate on immediate` | appena possibile (le parti non-deferred hanno precedenza) |
| `hydrate on timer(duration)` | dopo un ritardo, es. `hydrate on timer(500ms)` |
| `hydrate when (condition)` | quando la condizione è vera, es. `hydrate when (userName !== null)` |
| `hydrate never` | nessuna hydration; anche i `@defer` annidati restano statici |

> [!warning] Gotcha
> Per l'hydration *non distruttiva* il markup server-rendered deve **combaciare** con quello che Angular renderizzerebbe sul client. Componenti/librerie di terze parti che manipolano il DOM direttamente possono violare questo vincolo. Per escluderli usa l'attributo `ngSkipHydration` sull'host element: Angular **non** ammette data binding su questo attributo (deve essere assente o `'true'`). Per default a livello componente: `host: { 'ngSkipHydration': 'true' }`. Se più app Angular convivono sulla stessa pagina, vanno distinte via token `APP_ID` perché l'hydration colpisca l'app giusta.

### Event Replay
> 📖 p.428

Il periodo tra FMP e *Time to Interactive* — la **Uncanny Valley** — è quando l'utente già vede la pagina ma il JS che gestisce le interazioni non è ancora caricato: i click andrebbero persi. L'**Event Replay** lo evita: uno script nell'HTML server-rendered si carica con la risposta iniziale e **registra** le interazioni (click, input, ecc.) avvenute prima del termine dell'hydration. Quando l'app è interattiva, Angular **rigioca** gli eventi registrati, così nulla va perso.

Si abilita passando `withEventReplay()` a `provideClientHydration`.

> [!tip] Take-away
> Con il setup SSR integrato di Angular l'**Event Replay è attivo di default**: usando la config standard non serve aggiungerlo esplicitamente. La tecnica viene da Wiz, il framework interno di Google; Angular ha adottato l'Event Replay da Wiz mentre Wiz ha adottato i signal di Angular.

### Different Implementations for Server and Client
> 📖 pp.428-430

SSR e hydration fanno girare lo **stesso codice in due ambienti diversi**: sul server al render iniziale e nel browser dopo il boot. Alcune API (`window`, `document`, `navigator`) esistono solo nel browser; il codice server spesso ha bisogno di dati specifici della richiesta.

**Via DI** — quando un servizio ha responsabilità diverse su server e client, si forniscono **implementazioni diverse**. Es. un `LanguageDetector`: sul server legge l'header `Accept-Language` dalla richiesta HTTP, sul client legge `navigator.language`.

```ts
// src/app/domains/shared/util-common/language.ts
import { inject, Injectable, REQUEST } from '@angular/core';

export abstract class LanguageDetector {
  abstract getUserLang(): string;
}

@Injectable()
export class ServerLanguageDetector implements LanguageDetector {
  private request = inject(REQUEST);                 // token solo lato server
  getUserLang(): string {
    return this.request?.headers.get('accept-language') ?? 'en';
  }
}

@Injectable()
export class ClientLanguageDetector implements LanguageDetector {
  getUserLang(): string {
    return navigator.language;                        // API solo browser
  }
}
```

L'implementazione server si registra in `app.config.server.ts` (usato solo nell'esecuzione lato server); quella client in `app.config.ts`.

```ts
// src/app/app.config.server.ts — usato SOLO lato server
import { ApplicationConfig, mergeApplicationConfig } from '@angular/core';
import { provideServerRendering } from '@angular/ssr';
import { appConfig } from './app.config';
import { LanguageDetector, ServerLanguageDetector } from './domains/shared/util-common/language';

const serverConfig: ApplicationConfig = {
  providers: [
    provideServerRendering(),
    { provide: LanguageDetector, useClass: ServerLanguageDetector },
  ],
};
// merge: la config server estende quella applicativa
export const config = mergeApplicationConfig(appConfig, serverConfig);
```

```ts
// src/app/app.config.ts — lato client
export const appConfig: ApplicationConfig = {
  providers: [
    { provide: LanguageDetector, useClass: ClientLanguageDetector },
  ],
};
```

**Controllo della piattaforma a runtime** — se solo una piccola parte è platform-specific, conviene ramificare a runtime con gli helper `isPlatformBrowser` / `isPlatformServer` sul token `PLATFORM_ID`.

```ts
// src/app/app.ts
import { isPlatformBrowser } from '@angular/common';
import { Component, inject, PLATFORM_ID } from '@angular/core';

@Component({ selector: 'app-root', templateUrl: './app.html' })
export class App {
  private readonly platformId = inject(PLATFORM_ID);
  constructor() {
    if (isPlatformBrowser(this.platformId)) {
      // codice solo-browser
    }
  }
}
```

> [!warning] Gotcha
> Toccare `window`, `document` o `navigator` senza guardia fa crashare il render **lato server**. Isola sempre quel codice dietro DI (implementazioni server/client distinte) o dietro `isPlatformBrowser` / `isPlatformServer`.

Collegamenti: [[inject]] · [[providers]] · l'[[injection-context]] e i provider con `useClass`.

## Hybrid Routing
> 📖 pp.430-432

Non tutte le route traggono lo stesso beneficio dall'SSR: una landing/lista prodotti pubblica sì, una back-office dietro login forse no, una pagina "About" quasi statica è ottima per il **prerendering** a build time. L'**hybrid routing** lo permette: una config di route **lato server** sceglie il *render mode* per ogni route. Oltre alla normale routing config, l'app definisce le `serverRoutes` e le passa alla config server via `withRoutes`.

```ts
// src/app/app.config.server.ts
import { ApplicationConfig, mergeApplicationConfig } from '@angular/core';
import { provideServerRendering, withRoutes } from '@angular/ssr';
import { appConfig } from './app.config';
import { serverRoutes } from './app.routes.server';
import { LanguageDetector, ServerLanguageDetector } from './domains/shared/util-common/language';

const serverConfig: ApplicationConfig = {
  providers: [
    provideServerRendering(withRoutes(serverRoutes)),
    { provide: LanguageDetector, useClass: ServerLanguageDetector },
  ],
};
export const config = mergeApplicationConfig(appConfig, serverConfig);
```

Le decisioni per-route vivono in `app.routes.server.ts`. La catch-all `path: '**'` copre tutto ciò che non è elencato esplicitamente.

```ts
// src/app/app.routes.server.ts
import { PrerenderFallback, RenderMode, ServerRoute } from '@angular/ssr';

export const serverRoutes: ServerRoute[] = [
  { path: 'ticketing/reporting',               renderMode: RenderMode.Client },
  { path: 'ticketing/booking/flight-edit/:id', renderMode: RenderMode.Prerender },
  { path: 'about',                             renderMode: RenderMode.Prerender },
  { path: '**',                                renderMode: RenderMode.Server }, // catch-all
];
```

I tre render mode:
- **`RenderMode.Client`** — niente SSR sulla richiesta iniziale; SPA classica.
- **`RenderMode.Prerender`** — HTML statico generato a **build time**.
- **`RenderMode.Server`** — renderizzato sul server a **ogni richiesta**.

Server e prerender valgono solo per il render **iniziale**; dopo, Angular gira sul client.

### Prerendering Routes With Parameters
> 📖 pp.432-432

Il prerendering produce HTML statico a build time, quindi i valori possibili dei parametri devono essere **noti allora**. Invece di mantenere una lista separata, si definisce una funzione async **`getPrerenderParams`** sulla route: ritorna un array di record, uno per ogni combinazione di parametri per cui la CLI genera una pagina statica.

```ts
// src/app/app.routes.server.ts
import { PrerenderFallback, RenderMode, ServerRoute } from '@angular/ssr';

export const serverRoutes: ServerRoute[] = [
  {
    path: 'ticketing/booking/flight-edit/:id',
    renderMode: RenderMode.Prerender,
    fallback: PrerenderFallback.Server,        // valore non prerenderizzato → render server
    async getPrerenderParams() {
      return [{ id: '1' }, { id: '2' }, { id: '3' }];
    },
  },
];
```

`fallback` si applica quando si richiede un valore non prerenderizzato (es. link a `flight-edit/17` con solo 1, 2, 3 prerenderizzati). Default: `PrerenderFallback.Server`.

> [!warning] Gotcha
> Solo le combinazioni restituite da `getPrerenderParams` esistono come HTML statico. Senza un `fallback` adatto, gli `:id` non previsti non avrebbero pagina. `PrerenderFallback.Server` li fa renderizzare on-demand sul server.

Collegamenti: route parametrizzate e configurazione in [[04-router-navigation-lazy-loading|cap.4]].

### Working with the HTTP request and response
> 📖 pp.432-433

Per le route server-rendered l'app può **leggere la richiesta HTTP in entrata** e **influenzare la risposta**. Angular fornisce i token:
- **`REQUEST`** — l'oggetto request standard di Node/fetch.
- **`REQUEST_CONTEXT`** — oggetto opzionale fornito dal server con contesto (es. sessione utente corrente).
- **`RESPONSE_INIT`** — per impostare status code e header della risposta.

`REQUEST` e `REQUEST_CONTEXT` sono disponibili **solo lato server**: controlla `null` prima di usarli. Nell'esempio, `ServerLanguageDetector` inietta `REQUEST` e legge l'header `Accept-Language`; il client usa l'implementazione che si appoggia a `navigator.language`.

```ts
// src/app/domains/shared/util-common/language.ts
import { inject, Injectable, REQUEST } from '@angular/core';

@Injectable()
export class ServerLanguageDetector implements LanguageDetector {
  private request = inject(REQUEST);
  getUserLang(): string {
    return this.request?.headers.get('accept-language') ?? 'en'; // null-check sul request
  }
}
```

Per la **response** si possono impostare header e status **dichiarativamente** nella server route config, oppure iniettare **`RESPONSE_INIT`** in un componente/servizio e settare `status`, `statusText` e `headers` **programmaticamente** — utile quando status/header dipendono dal risultato della logica server.

> [!tip] Take-away
> L'hybrid routing dà il meglio combinato con `@defer` + incremental hydration: anche su route server-rendered le regioni non critiche restano fuori dal bundle iniziale e diventano interattive solo quando l'utente le richiede.

Collegamenti: [[inject]] · [[12-initialization-route-changes|cap.12]] (resolver/interceptor lato HTTP).

## 🔁 Ripasso lampo
1. Cosa fanno `@placeholder`, `@loading` e `@error` in un blocco `@defer`? A cosa servono `minimum` e `after`?
2. Elenca i trigger di `@defer`. Quali richiedono un `@placeholder` e come si può aggirare? Cosa fa `prefetch on immediate`?
3. Qual è la differenza tra `@defer (on …)` e la clausola `hydrate on …`? Cosa fa da placeholder fino all'hydration?
4. Cosa risolve l'Event Replay e qual è la "Uncanny Valley"? È attivo di default?
5. Due modi per avere implementazioni diverse server/client: quando usare la DI (`useClass`) e quando `isPlatformBrowser`/`isPlatformServer`?
6. Differenza tra `RenderMode.Client`, `Prerender` e `Server`? A cosa servono `getPrerenderParams` e `fallback`?
7. Cosa fanno i token `REQUEST`, `REQUEST_CONTEXT` e `RESPONSE_INIT`? Perché serve un null-check?

**Take-away del capitolo:**
- Le **deferrable views** (`@defer`) riducono il bundle iniziale caricando le regioni UI non critiche solo allo scattare di un trigger (`on idle/viewport/interaction/hover/immediate/timer/when`, con `prefetch`).
- L'**SSR** (`@angular/ssr`, `provideClientHydration`) migliora il primo paint inviando HTML pronto; l'**hydration** lo rende interattivo, e quella **incrementale** (`withIncrementalHydration` + `hydrate on …`) idrata on-demand abbinandosi a `@defer`.
- L'**Event Replay** (`withEventReplay`) non perde le interazioni pre-hydration; per codice platform-specific si usano DI (impl. server/client) o `isPlatformBrowser`/`isPlatformServer`.
- L'**hybrid routing** sceglie un `RenderMode` per route (`Client`/`Prerender`/`Server`), prerenderizza route parametrizzate con `getPrerenderParams`/`fallback` e legge/scrive HTTP via `REQUEST`/`REQUEST_CONTEXT`/`RESPONSE_INIT`.
