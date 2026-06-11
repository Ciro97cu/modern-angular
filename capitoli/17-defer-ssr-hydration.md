---
capitolo: 17
titolo: "Defer, SSR & Hydration"
pagine: "422-434"
tags: [tipo/capitolo, ssr, performance, angular-22]
---
# 17 · Defer, SSR & Hydration
> 📖 cap.17 · pp.422-434 — *Modern Angular* v2.0.0

Le SPA hanno ottime performance a runtime, ma il **primo caricamento** è spesso più lento di qualche secondo rispetto a un sito classico server-rendered: il browser non riceve solo l'HTML, ma deve anche scaricare ed eseguire una quantità significativa di JavaScript prima di poter renderizzare l'applicazione. Il *First Meaningful Paint* (FMP) arriva quindi tardi. Per app aziendali interne il ritardo può essere accettabile; per soluzioni pubbliche come web shop e landing page diventa un problema (più secondi di attesa = più bounce).

Il capitolo presenta tre meccanismi complementari:
- **Deferrable views** (`@defer`): rimandano le regioni non critiche.
- **SSR** (server-side rendering): invia HTML già pronto da dipingere.
- **Hydration**: rende interattivo quell'HTML sul client, eventualmente a step incrementali.

## Deferrable Views — `@defer`
> 📖 pp.422-423

Non tutte le aree di una pagina contano allo stesso modo. Sulla pagina di ricerca voli, form e lista risultati sono primari; i pannelli opzionali (noleggio auto, hotel, tenda) sono secondari. Caricarne il codice subito appesantisce inutilmente il bundle iniziale. Con `@defer` si avvolge la regione: Angular ne **splitta il codice in un bundle separato** e lo carica solo allo scattare di un trigger. Nel frattempo `@placeholder` fornisce contenuto alternativo, così il layout non "salta".

```html
<!-- .../ticketing/feature-booking/flight-search/flight-search.html -->
@defer (on hover) {
  <app-car-pane />
} @placeholder {
  <app-placeholder />
}
```

Nella demo i pannelli opzionali mostrano prima dei ghost element; appena il bundle è caricato, `@defer` scambia il placeholder col contenuto reale.

Oltre a `@placeholder`, `@defer` supporta `@loading` (mostrato mentre si scarica il bundle) e `@error` (se il caricamento fallisce). Per evitare flicker:
- `minimum` su `@placeholder`/`@loading` → durata minima di visualizzazione.
- `after` su `@loading` → ritarda lo spinner finché il caricamento non supera la soglia indicata.

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

> [!tip]
> Usa `@defer` per UI che chiaramente **non** serve alla prima interazione utile. Differire la navigazione core o le azioni primarie costringe l'utente a uno stato di loading proprio quando ha bisogno di quella funzionalità: danneggia la *perceived performance*.

### Triggers
> 📖 pp.423-424

Insieme fisso di trigger su `@defer`:

| Trigger | Quando scatta |
|---|---|
| `on idle` | il browser segnala che non ha task critici pendenti (**default**) |
| `on viewport` | il placeholder (o un elemento referenziato) entra nell'area visibile |
| `on interaction` | l'utente inizia a interagire col placeholder o con un elemento referenziato |
| `on hover` | il mouse passa sopra il placeholder o un elemento referenziato |
| `on immediate` | appena possibile dopo il load della pagina |
| `on timer(duration)` | dopo una durata, es. `on timer(5s)` |
| `when (condition)` | quando la condizione è vera, es. `when (userName() !== null)` |

Di default `on viewport`, `on interaction` e `on hover` **richiedono un `@placeholder`**. In alternativa possono riferirsi a un altro elemento via template variable: `@defer (on viewport(recommendations)) { … }` con un corrispondente `#recommendations` su qualche elemento.

**Prefetch**: si può anticipare il download del bundle senza inserire subito il contenuto. `@defer (on viewport; prefetch on immediate)` inizia a caricare appena possibile, ma scambia il contenuto solo quando il placeholder entra nel viewport.

```html
@defer (on viewport; prefetch on immediate) {
  <app-recommendations />
} @placeholder {
  <app-skeleton />
}
```

Collegamenti: il code-splitting/lazy loading è lo stesso tema del [[04-router-navigation-lazy-loading|cap.4]] applicato a livello di template.

## SSR & Hydration
> 📖 pp.424-425

Per migliorare il primo paint delle soluzioni pubbliche si rende la SPA **sul server**, che consegna una pagina HTML completa: il chiamante vede il contenuto prima. Una volta caricati i bundle JS, la pagina diventa interattiva — questo processo si chiama **hydration**.

### Adding Server-Side Rendering
> 📖 p.424

Per aggiungere il supporto SSR basta aggiungere il pacchetto `@angular/ssr`, oppure usare lo switch `--ssr` alla creazione del progetto:

```bash
ng add @angular/ssr
# oppure, in fase di creazione:
ng new my-app --ssr
```

La configurazione dell'hydration vive in `app.config.ts`:

```ts
// src/app/app.config.ts
import { provideHttpClient } from '@angular/common/http';
// [...]
import {
  provideClientHydration,
  withEventReplay,
} from '@angular/platform-browser';

export const appConfig: ApplicationConfig = {
  providers: [
    // [...]
    provideHttpClient(),            // FetchBackend di default (v22): niente withFetch()
    provideClientHydration(
      withEventReplay(),            // incremental hydration è già di default (v22)
    ),
  ],
};
```

> [!info] Angular 22+ · FetchBackend di default
> Da **Angular 22** `HttpClient` usa il **`FetchBackend`** di default: il vecchio opt-in `withFetch()` non serve più ed è deprecato — basta `provideHttpClient()`. La fetch API è ora disponibile in tutti i browser e runtime rilevanti, offre un'API Promise-based più moderna e funziona ugualmente bene nel browser e durante l'SSR.
> L'unica feature che la fetch **non** offre è l'**upload progress**: se dipendi da `reportUploadProgress`, torna a XHR con `provideHttpClient(withXhr())`. Preferisci comunque le opzioni dedicate `reportDownloadProgress` e `reportUploadProgress` al vecchio `reportProgress`, ora deprecato.
> **Prima della 22** (fino ad Angular 21) il default era `XhrBackend` e il comportamento fetch si attivava con `provideHttpClient(withFetch())` (importante in SSR e hydration). Al bump, `ng update` inserisce automaticamente `withXhr()` dove serviva il vecchio comportamento.

> [!tip]
> Con la config standard di Angular 22 bastano `provideHttpClient()` + `provideClientHydration(withEventReplay())`: fetch backend e hydration incrementale sono **default**.

Collegamenti: [[providers]] · le provider function (`provide*`/`with*`).

### Incremental Hydration
> 📖 pp.425-426

L'hydration **non deve avvenire tutta in una volta**. I bundle si richiedono solo quando servono — il più tardi possibile, idealmente mai per le regioni che l'utente non tocca. Le parti più importanti diventano interattive prima; le meno importanti dopo. Si abbina naturalmente a `@defer`: le regioni da idratare on-demand si marcano con `@defer` e la clausola **`hydrate`** dice *quando* idratare.

> [!info] Angular 22+ · Hydration incrementale di default
> Da **Angular 22** `provideClientHydration()` attiva l'hydration incrementale da sola: la feature esplicita `withIncrementalHydration()` non serve più ed è tenuta solo per gli scenari di opt-out. Per disattivarla usa la nuova feature **`withNoIncrementalHydration()`**.
> **Prima della 22** (Angular 19–21) era opt-in, da attivare con `provideClientHydration(withIncrementalHydration())`. Al bump, `ng update` aggiunge `withNoIncrementalHydration()` dove la feature non era richiesta, così il comportamento non cambia.

```html
<!-- .../ticketing/feature-booking/flight-search/flight-search.html -->
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
| `hydrate on immediate` | appena possibile (le parti non-deferred hanno comunque precedenza) |
| `hydrate on timer(duration)` | dopo un ritardo, es. `hydrate on timer(500ms)` |
| `hydrate when (condition)` | quando la condizione è vera, es. `hydrate when (userName !== null)` |
| `hydrate never` | nessuna hydration; anche i `@defer` annidati restano statici |

> [!warning]
> Per l'hydration *non distruttiva* il markup server-rendered deve **combaciare** con quello che Angular renderizzerebbe sul client. Componenti o librerie di terze parti che manipolano il DOM direttamente possono violare questo vincolo. Per escluderli usa l'attributo `ngSkipHydration` sull'host element: Angular **non** ammette data binding su questo attributo (deve essere assente o impostato a `'true'`). Per escludere un componente per default, mettilo nei `host` metadata: `host: { 'ngSkipHydration': 'true' }`. Se più app Angular convivono sulla stessa pagina, vanno distinte via token `APP_ID` perché l'hydration colpisca l'app giusta.

### Event Replay
> 📖 p.428

Il periodo tra FMP e *Time to Interactive* — la **Uncanny Valley** — è quando l'utente già vede la pagina ma il JavaScript che gestisce le interazioni non è ancora caricato: i click e gli altri eventi andrebbero persi. L'**Event Replay** lo evita: un piccolo script incluso nell'HTML server-rendered si carica con la risposta iniziale e **registra** le interazioni (click, input nei form, ecc.) avvenute prima del termine dell'hydration. Quando l'app è interattiva, Angular **rigioca** gli eventi registrati, così nulla va perso.

Si abilita passando `withEventReplay()` a `provideClientHydration`, come nel listato di `app.config.ts` sopra.

> [!tip]
> Con il setup SSR integrato di Angular l'**Event Replay è attivo di default**: usando la config standard non serve aggiungerlo esplicitamente. La tecnica è usata da tempo in **Wiz**, il framework interno di Google noto per le sue capacità di SSR e hydration: Angular ha adottato l'Event Replay da Wiz, mentre Wiz ha adottato i signal di Angular.

### Different Implementations for Server and Client
> 📖 pp.428-430

SSR e hydration fanno girare lo **stesso codice Angular in due ambienti molto diversi**: sul server al render iniziale e nel browser dopo il boot. Alcune API (`window`, `document`, `navigator`) esistono solo nel browser; il codice server spesso ha bisogno di dati specifici della richiesta.

**Via DI** — quando un servizio ha responsabilità diverse su server e client, si forniscono **implementazioni diverse**. Es. un `LanguageDetector` che rileva la lingua dell'utente: sul server legge l'header `Accept-Language` dalla richiesta HTTP, sul client legge `navigator.language`.

```ts
// src/app/domains/shared/util-common/language.ts
import { inject, REQUEST, Service } from '@angular/core';

export abstract class LanguageDetector {
  abstract getUserLang(): string;
}

@Service({ autoProvided: false })
export class ServerLanguageDetector implements LanguageDetector {
  private request = inject(REQUEST);                  // token solo lato server
  getUserLang(): string {
    return this.request?.headers.get('accept-language') ?? 'en';
  }
}

@Service({ autoProvided: false })
export class ClientLanguageDetector implements LanguageDetector {
  getUserLang(): string {
    return navigator.language;                         // API solo browser
  }
}
```

Entrambe le implementazioni sono decorate con `@Service({ autoProvided: false })`: sono **iniettabili ma non registrate** globalmente. È la configurazione applicativa platform-specific a decidere quale usare. L'implementazione server si registra in `app.config.server.ts` (eseguito solo lato server); quella client in `app.config.ts`.

> [!info] Angular 22+
> `@Service({ autoProvided: false })` dichiara un service iniettabile ma **non** auto-registrato nello scope root: lo fornisci tu dove serve. Equivale al vecchio `@Injectable()` *senza* `providedIn: 'root'`. Vedi [[service]].

```ts
// src/app/app.config.server.ts — usato SOLO lato server
import { ApplicationConfig, mergeApplicationConfig } from '@angular/core';
import { provideServerRendering } from '@angular/ssr';
import { appConfig } from './app.config';
import {
  LanguageDetector,
  ServerLanguageDetector,
} from './domains/shared/util-common/language';

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
    // [...]
    { provide: LanguageDetector, useClass: ClientLanguageDetector },
  ],
};
```

**Controllo della piattaforma a runtime** — se solo una piccola parte è platform-specific, conviene ramificare a runtime con gli helper `isPlatformBrowser` / `isPlatformServer` sul token `PLATFORM_ID`.

```ts
// src/app/app.ts
import { isPlatformBrowser } from '@angular/common';
import { Component, inject, PLATFORM_ID } from '@angular/core';

@Component({
  selector: 'app-root',
  templateUrl: './app.html',
})
export class App {
  private readonly platformId = inject(PLATFORM_ID);
  constructor() {
    if (isPlatformBrowser(this.platformId)) {
      // codice solo-browser
    }
  }
}
```

> [!warning]
> Toccare `window`, `document` o `navigator` senza guardia fa crashare il render **lato server**. Isola sempre quel codice dietro DI (implementazioni server/client distinte) o dietro `isPlatformBrowser` / `isPlatformServer`.

Collegamenti: [[inject]] · [[providers]] · l'[[injection-context]] e i provider con `useClass`.

## Hybrid Routing
> 📖 pp.430-432

Non tutte le route traggono lo stesso beneficio dall'SSR: una landing o lista prodotti pubblica sì, una route di back-office dietro login forse non giustifica il costo, una pagina "About" quasi statica è ottima per il **prerendering** a build time. L'**hybrid routing** lo permette: una config di route **lato server** sceglie il *render mode* per ogni route. Oltre alla normale routing config, l'app definisce le `serverRoutes` e le passa alla config server.

```ts
// src/app/app.config.server.ts
import { ApplicationConfig, mergeApplicationConfig } from '@angular/core';
import { provideServerRendering, withRoutes } from '@angular/ssr';
import { appConfig } from './app.config';
import { serverRoutes } from './app.routes.server';
import {
  LanguageDetector,
  ServerLanguageDetector,
} from './domains/shared/util-common/language';

const serverConfig: ApplicationConfig = {
  providers: [
    provideServerRendering(withRoutes(serverRoutes)),
    { provide: LanguageDetector, useClass: ServerLanguageDetector },
  ],
};
export const config = mergeApplicationConfig(appConfig, serverConfig);
```

Le decisioni per-route vivono in `app.routes.server.ts`: ogni entry assegna un `renderMode` a un path; la catch-all `path: '**'` copre tutto ciò che non è elencato esplicitamente.

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
> 📖 p.432

Il prerendering produce HTML statico a build time, quindi i valori possibili dei parametri di route devono essere **noti allora**. Invece di mantenere una lista separata di route, si definisce una funzione async **`getPrerenderParams`** sulla route config: ritorna un array di record, uno per ogni combinazione di parametri per cui la CLI genera una pagina statica.

```ts
// src/app/app.routes.server.ts
import { PrerenderFallback, RenderMode, ServerRoute } from '@angular/ssr';

export const serverRoutes: ServerRoute[] = [
  // [...]
  {
    path: 'ticketing/booking/flight-edit/:id',
    renderMode: RenderMode.Prerender,
    fallback: PrerenderFallback.Server,        // valore non prerenderizzato → render server
    async getPrerenderParams() {
      return [{ id: '1' }, { id: '2' }, { id: '3' }];
    },
  },
  // [...]
];
```

`fallback` si applica quando si richiede un valore di parametro non prerenderizzato (es. un link a `flight-edit/17` con solo 1, 2, 3 prerenderizzati). Default: `PrerenderFallback.Server`.

> [!warning]
> Solo le combinazioni restituite da `getPrerenderParams` esistono come HTML statico. Senza un `fallback` adatto, gli `:id` non previsti non avrebbero pagina. `PrerenderFallback.Server` li fa renderizzare on-demand sul server.

Collegamenti: route parametrizzate e configurazione in [[04-router-navigation-lazy-loading|cap.4]].

### Working with the HTTP request and response
> 📖 pp.432-433

Per le route server-rendered l'app può **leggere la richiesta HTTP in entrata** e **influenzare la risposta**. Angular fornisce i token:
- **`REQUEST`** — l'oggetto request standard di Node/fetch.
- **`REQUEST_CONTEXT`** — oggetto opzionale fornito dal server con contesto (es. la sessione utente corrente).
- **`RESPONSE_INIT`** — per impostare status code e header della risposta.

`REQUEST` e `REQUEST_CONTEXT` sono disponibili **solo lato server**: controlla `null` prima di usarli. Nell'esempio, `ServerLanguageDetector` inietta `REQUEST` e legge l'header `Accept-Language`; il client usa l'implementazione che si appoggia a `navigator.language`.

```ts
// src/app/domains/shared/util-common/language.ts
import { inject, REQUEST, Service } from '@angular/core';

export abstract class LanguageDetector {
  abstract getUserLang(): string;
}

@Service({ autoProvided: false })
export class ServerLanguageDetector implements LanguageDetector {
  private request = inject(REQUEST);
  getUserLang(): string {
    return this.request?.headers.get('accept-language') ?? 'en'; // null-check sul request
  }
}

@Service({ autoProvided: false })
export class ClientLanguageDetector implements LanguageDetector {
  getUserLang(): string {
    return navigator.language;
  }
}
```

Per la **response** si possono impostare header e status **dichiarativamente** nella server route config, oppure iniettare **`RESPONSE_INIT`** in un componente/servizio e settare `status`, `statusText` e `headers` **programmaticamente** — utile quando status o header dipendono dal risultato della logica server.

> [!tip]
> L'hybrid routing dà il meglio combinato con `@defer` + incremental hydration: anche su route server-rendered le regioni non critiche restano fuori dal bundle iniziale e diventano interattive solo quando l'utente le richiede.

Collegamenti: [[inject]] · [[12-initialization-route-changes|cap.12]] (resolver/interceptor lato HTTP).

## 🔁 Ripasso lampo

**1.** Cosa fanno `@placeholder`, `@loading` e `@error` in un blocco `@defer`? A cosa servono `minimum` e `after`?
> [!success]- Risposta
> `@placeholder` mostra contenuto alternativo finché il trigger non scatta (così il layout non salta); `@loading` è mostrato **mentre** si scarica il bundle; `@error` se il caricamento fallisce. `minimum` (su `@placeholder`/`@loading`) impone una durata minima di visualizzazione per evitare flicker; `after` (su `@loading`) ritarda lo spinner finché il caricamento non supera la soglia indicata.

**2.** Elenca i trigger di `@defer`. Quali richiedono un `@placeholder` e come si può aggirare? Cosa fa `prefetch on immediate`?
> [!success]- Risposta
> `on idle` (default), `on viewport`, `on interaction`, `on hover`, `on immediate`, `on timer(duration)`, `when (condition)`. Di default `on viewport`, `on interaction` e `on hover` richiedono un `@placeholder`; si aggira riferendo un altro elemento via template variable, es. `on viewport(recommendations)` con `#recommendations`. `prefetch on immediate` anticipa il download del bundle appena possibile, ma scambia il contenuto solo allo scattare del trigger principale.

**3.** Qual è la differenza tra `@defer (on …)` e la clausola `hydrate on …`? Cosa fa da placeholder fino all'hydration?
> [!success]- Risposta
> `on …` decide **quando caricare** il bundle e renderizzare la regione (anche in una SPA client-only); `hydrate on …` decide **quando rendere interattiva** (idratare) una regione già renderizzata dal server. Fino all'hydration, è **il markup server-rendered** a fare da placeholder. `hydrate` agisce solo sul render iniziale: navigando a una nuova route sul client, Angular renderizza tutto sul client e l'incremental hydration non si applica.

**4.** Cosa risolve l'Event Replay e qual è la "Uncanny Valley"? È attivo di default?
> [!success]- Risposta
> La **Uncanny Valley** è il periodo tra *First Meaningful Paint* e *Time to Interactive*: l'utente vede la pagina ma il JS delle interazioni non è ancora caricato, quindi click e input andrebbero persi. L'**Event Replay** registra quelle interazioni con un piccolo script nell'HTML server-rendered e le **rigioca** una volta che l'app è interattiva. Con il setup SSR integrato di Angular è **attivo di default** (`withEventReplay()` con la config standard).

**5.** Due modi per avere implementazioni diverse server/client: quando usare la DI (`useClass`) e quando `isPlatformBrowser`/`isPlatformServer`?
> [!success]- Risposta
> **DI con `useClass`**: quando un intero servizio ha responsabilità diverse sui due ambienti — definisci una classe astratta come token e fornisci la classe server in `app.config.server.ts` e quella client in `app.config.ts`. **`isPlatformBrowser`/`isPlatformServer`** (sul token `PLATFORM_ID`): quando solo una piccola parte di un componente/servizio è platform-specific, conviene ramificare a runtime senza creare due classi.

**6.** Differenza tra `RenderMode.Client`, `Prerender` e `Server`? A cosa servono `getPrerenderParams` e `fallback`?
> [!success]- Risposta
> `Client` = niente SSR sulla richiesta iniziale, SPA classica; `Prerender` = HTML statico generato a **build time**; `Server` = renderizzato sul server a **ogni richiesta** (server e prerender valgono solo per il render iniziale). `getPrerenderParams` è una funzione async che ritorna le combinazioni di parametri per cui prerenderizzare le pagine; `fallback` (default `PrerenderFallback.Server`) decide cosa fare quando si richiede un parametro non prerenderizzato.

**7.** Cosa fanno i token `REQUEST`, `REQUEST_CONTEXT` e `RESPONSE_INIT`? Perché serve un null-check?
> [!success]- Risposta
> `REQUEST` è l'oggetto request standard di Node/fetch; `REQUEST_CONTEXT` è contesto opzionale fornito dal server (es. sessione utente); `RESPONSE_INIT` imposta status code e header della risposta. Serve un null-check perché `REQUEST` e `REQUEST_CONTEXT` esistono **solo lato server** e sono `null` quando il codice gira nel browser.

**In sintesi:**
- Le **deferrable views** (`@defer`) riducono il bundle iniziale caricando le regioni UI non critiche solo allo scattare di un trigger (`on idle/viewport/interaction/hover/immediate/timer/when`, con `prefetch`).
- L'**SSR** (`@angular/ssr`, `provideClientHydration`) migliora il primo paint inviando HTML pronto; l'**hydration** lo rende interattivo, e quella **incrementale** (di default in v22, `hydrate on …`) idrata on-demand abbinandosi a `@defer`.
- L'**Event Replay** (`withEventReplay`, default in v22) non perde le interazioni pre-hydration; per codice platform-specific si usano DI (impl. server/client con `@Service({ autoProvided: false })` + `useClass`) o `isPlatformBrowser`/`isPlatformServer`.
- L'**hybrid routing** sceglie un `RenderMode` per route (`Client`/`Prerender`/`Server`), prerenderizza route parametrizzate con `getPrerenderParams`/`fallback` e legge/scrive HTTP via `REQUEST`/`REQUEST_CONTEXT`/`RESPONSE_INIT`.
