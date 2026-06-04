---
capitolo: 12
titolo: "Initialization & Route Changes"
pagine: "342-356"
tags: [tipo/capitolo, routing, lifecycle, di, angular-22]
---
# 12 · Initialization & Route Changes
> 📖 cap.12 · pp.342-356 — *Modern Angular* v2.0.0

Prima di poter usare una feature spesso serve **inizializzarla**: caricare dati, registrare error handler, avviare servizi. Questo avviene tipicamente allo **startup** e al **cambio rotta**. Il capitolo raccoglie i meccanismi tecnici per agganciarsi a questi momenti: **initializers** (app/environment/platform), **guards** (consentire/negare activation e deactivation), **router events**, **resolver** (caricare dati *prima* dell'attivazione della rotta) e **HttpInterceptors** (ispezionare/modificare richieste e risposte HTTP).

Tutti questi hook girano in un [[injection-context]], quindi usano [[inject]]`(...)` direttamente invece della constructor injection.

## Initializers
> 📖 pp.342-345

Gli initializer servono per compiti tecnici da eseguire *prima* che la UI parta davvero: caricare la configurazione a runtime, registrare error handler globali, avviare servizi di background. Angular offre tre livelli di hook.

### Application Initializers
> 📖 pp.342-344

Un application initializer gira durante `bootstrapApplication`. Se la funzione passata ritorna una **Promise** o un **Observable**, Angular **attende** che si completi prima di renderizzare i componenti: ideale per caricare config o dati da cui dipende il resto dell'app.

```ts
// src/app/app.config.ts
import {
  ApplicationConfig,
  inject,
  provideAppInitializer,
  provideBrowserGlobalErrorListeners,
} from '@angular/core';
import { routes } from './app.routes';
import { ConfigService } from './domains/shared/util-common/config-service';

export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    provideAppInitializer(() => inject(ConfigService).load()),  // attende la Promise di load()
    // ...
  ],
};
```

L'initializer gira in un [[injection-context]] → può usare [[inject]] direttamente. La logica asincrona vera vive nel servizio:

> [!info] Angular 22+
> Negli snippet `@Injectable({ providedIn: 'root' })` ≡ `@Service()` (Angular 22). Dettagli in [[service]].

```ts
// src/app/domains/shared/util-common/config-service.ts
import { HttpClient } from '@angular/common/http';
import { inject, Injectable } from '@angular/core';
import { firstValueFrom } from 'rxjs';

export interface Config {
  readonly baseUrl: string;
  readonly model: string;
}

@Injectable({ providedIn: 'root' })
export class ConfigService {
  private readonly http = inject(HttpClient);
  private _baseUrl = 'https://demo.angulararchitects.io/api';
  private _model = 'gpt-5-chat-latest';

  get baseUrl() { return this._baseUrl; }
  get model() { return this._model; }

  async load(configPath = '/config.json'): Promise<void> {
    const config = await firstValueFrom(this.http.get<Config>(configPath));
    this._model = config.model;
    this._baseUrl = config.baseUrl;
  }
}
```

> [!warning] Gotcha
> Un application initializer **ritarda il primo render** e può far sembrare lo startup lento. Se ti servono dati solo per una feature specifica, caricali in modo **lazy** (es. all'attivazione della rotta, via [[#Resolver]]) invece di bloccare l'intera app.

### Environment Initializers
> 📖 p.344

Mentre `provideAppInitializer` è **globale** (gira col root injector), un environment initializer gira quando viene creato un **environment injector** — utile per setup *feature-* o *route-scoped* quando usi provider a livello di rotta (vedi [[04-router-navigation-lazy-loading]]).

```ts
// src/app/domains/ticketing/ticketing.routes.ts
export const bookingRoutes: Routes = [
  {
    path: 'booking',
    component: BookingNavigation,
    providers: [
      provideEnvironmentInitializer(() => {
        console.log('init bookingRoutes');
      }),
    ],
    // ...
  },
];
```

L'initializer è agganciato all'injector della rotta. Usi tipici: avviare servizi specifici della feature, configurare logging/telemetria di un'area, registrare listener che devono esistere solo finché vive quell'injector.

### Platform Initializers
> 📖 p.345

Registrato con `providePlatformInitializer`, gira quando viene creata la **platform** Angular, *prima* del bootstrap dell'applicazione. In pratica è usato soprattutto da Angular stesso e da librerie infrastrutturali di basso livello: per il codice applicativo quotidiano sono preferibili `provideAppInitializer` e gli environment initializer route/feature-scoped.

Collegamenti: [[inject]] · [[providers]] · [[injection-context]] · [[04-router-navigation-lazy-loading]].

## Guards
> 📖 pp.345-350

I guard informano l'app sui cambi di rotta: sono **funzioni** che il router chiama in certi momenti, e il cui **valore di ritorno** decide se la navigazione può procedere.

- Decisione **immediata** → `boolean` (oppure `UrlTree` per i redirect).
- Decisione **differita** (serve consultare una web API o chiedere all'utente) → `Observable<boolean>` o `Promise<boolean>`.

Funzioni guard tipizzate (nomi funzionali moderni):

| Tipo | Decide se… |
|---|---|
| `CanActivateFn` | la rotta richiesta può essere **attivata** |
| `CanActivateChildFn` | e quali **child route** possono essere attivate |
| `CanMatchFn` | la rotta può fare **match** |
| `CanDeactivateFn` | la rotta può essere **disattivata** (si può lasciare) |

### Preventing Route Activation (CanActivateFn)
> 📖 pp.345-347

Esempio: un auth guard che redirige al login gli utenti non autenticati. Nota: nelle SPA browser-based la **sicurezza va sempre imposta sul backend** — il guard qui è una questione di *usability* (vedi [[16-authentication-authorization]]).

```ts
// src/app/domains/shared/util-auth/auth.guard.ts
import { inject } from '@angular/core';
import {
  ActivatedRouteSnapshot,
  CanActivateFn,
  Router,
  RouterStateSnapshot,
} from '@angular/router';
import { AuthService } from './auth.service';

export const authGuard: CanActivateFn = (
  _route: ActivatedRouteSnapshot,
  _state: RouterStateSnapshot,
) => {
  const authService = inject(AuthService);   // injection context → inject()
  const router = inject(Router);

  if (authService.isLoggedIn()) {
    return true;
  }
  return router.createUrlTree(['/home']);    // UrlTree → redirect
};
```

Si registra nell'array `canActivate` della rotta:

```ts
// src/app/domains/ticketing/ticketing.routes.ts
import { authGuard } from '../shared/util-auth/auth.guard';

export const bookingRoutes: Routes = [
  {
    path: 'booking',
    component: BookingNavigation,
    children: [
      {
        path: 'flight-edit/:id',
        component: FlightEdit,
        canActivate: [authGuard],
      },
    ],
  },
];
```

> [!warning] Gotcha
> `canActivate` è un **array**: la navigazione procede solo se **ogni** guard ritorna `true` (o un `UrlTree` per redirect). Basta un `false` per bloccarla.

### Preventing Route Deactivation (CanDeactivateFn)
> 📖 pp.347-350

Un `CanDeactivateFn` può chiedere all'utente se vuole davvero lasciare la rotta, evitando di perdere dati modificati ma non salvati. Riceve come **primo parametro l'istanza del componente** corrente. Qui i componenti proteggibili implementano un'interfaccia `FormComponent` con `isDirty()`; il guard usa il CDK `Dialog` per chiedere conferma.

```ts
// src/app/domains/shared/util-common/exit.guard.ts
import { Dialog } from '@angular/cdk/dialog';
import { inject } from '@angular/core';
import {
  ActivatedRouteSnapshot,
  CanDeactivateFn,
  RouterStateSnapshot,
} from '@angular/router';
import { map } from 'rxjs';
import { ConfirmComponent } from './confirm';

export interface FormComponent {
  isDirty(): boolean;
}

export const exitGuard: CanDeactivateFn<FormComponent> = (
  component: FormComponent,       // ← istanza del componente da lasciare
  _currentRoute: ActivatedRouteSnapshot,
  _currentState: RouterStateSnapshot,
  _nextState: RouterStateSnapshot,
) => {
  const dialog = inject(Dialog);

  if (!component.isDirty()) {
    return true;                  // niente modifiche → si può uscire
  }

  const dialogRef = dialog.open<boolean>(ConfirmComponent, {
    data: 'Do you really want to leave without saving?',
  });
  return dialogRef.closed.pipe(map((result) => (result ? true : false)));
};
```

Il contenuto del dialog è un piccolo `ConfirmComponent` che chiude con la scelta dell'utente:

```ts
// src/app/domains/shared/util-common/confirm.ts
import { DIALOG_DATA, DialogRef } from '@angular/cdk/dialog';
import { Component, inject } from '@angular/core';

@Component({
  selector: 'app-confirm',
  template: `
    <div class="card">
      <div class="card-body">
        <p>{{ message }}</p>
        <button (click)="close(true)" class="btn btn-default">Yes</button>
        <button (click)="close(false)" class="btn btn-default">No</button>
      </div>
    </div>
  `,
})
export class ConfirmComponent {
  protected readonly message = inject(DIALOG_DATA) as string;
  private readonly dialogRef = inject(DialogRef) as DialogRef<boolean>;

  close(decision: boolean): void {
    this.dialogRef.close(decision);
  }
}
```

Il componente protetto implementa `FormComponent` esponendo `isDirty()`:

```ts
// src/app/domains/ticketing/feature-booking/flight-edit/flight-edit.ts
import { FormComponent } from '../../../shared/util-common/exit.guard';

@Component({ selector: 'app-flight-edit' /* ... */ })
export class FlightEdit implements FormComponent {
  isDirty(): boolean {
    return this.flightForm().dirty();
  }
}
```

Si registra con la property `canDeactivate` della rotta (qui insieme a `canActivate`):

```ts
// src/app/domains/ticketing/ticketing.routes.ts
{
  path: 'flight-edit/:id',
  component: FlightEdit,
  canActivate: [authGuard],
  canDeactivate: [exitGuard],
}
```

> [!tip] Take-away
> I guard moderni sono **funzioni** (`CanActivateFn`, `CanDeactivateFn`, …) che girano in injection context. `CanActivate` protegge l'ingresso; `CanDeactivate` riceve l'**istanza del componente** e protegge l'uscita (es. form dirty).

Collegamenti: [[inject]] · [[injection-context]] · [[16-authentication-authorization]] (auth guard) · [[04-router-navigation-lazy-loading]].

## Router Events
> 📖 pp.350-352

Per reagire all'attività di routing, il router pubblica **eventi** su `router.events` (un Observable). Una selezione:

| Evento | Significato |
|---|---|
| `NavigationStart` | è iniziato il cambio verso una nuova rotta |
| `RoutesRecognized` | il router ha derivato la rotta richiesta dall'URL |
| `NavigationEnd` | il cambio rotta si è **completato** |
| `NavigationCancel` | il cambio è stato **annullato** da un guard |
| `NavigationError` | si è verificato un **errore** durante il cambio |

Sequenza tipica di una navigazione che va a buon fine (o che viene interrotta):

```mermaid
sequenceDiagram
    participant U as Utente
    participant R as Router
    participant G as Guards / Resolver
    participant C as Componente
    U->>R: navigazione (click / navigate())
    R-->>R: NavigationStart
    R-->>R: RoutesRecognized
    R->>G: CanActivate / Resolve
    alt guard nega (false / UrlTree)
        G-->>R: blocco
        R-->>R: NavigationCancel
    else guard ok + resolver completati
        G-->>R: ok + dati risolti
        R->>C: attiva rotta
        R-->>R: NavigationEnd
    end
    Note over R: in caso di eccezione → NavigationError
```

Uso classico: mostrare un **loading indicator** durante i cambi rotta. Il root `App` inietta il `Router` e si sottoscrive:

```ts
// src/app/app.ts
import {
  NavigationCancel,
  NavigationEnd,
  NavigationError,
  NavigationStart,
  Router,
  RouterOutlet,
} from '@angular/router';

@Component({ /* ... */ })
export class App {
  private readonly router = inject(Router);
  protected readonly isLoading = signal(false);

  constructor() {
    this.router.events.subscribe((events) => {
      if (events instanceof NavigationStart) {
        this.isLoading.set(true);
      } else if (
        events instanceof NavigationEnd ||
        events instanceof NavigationError ||
        events instanceof NavigationCancel
      ) {
        this.isLoading.set(false);
      }
    });
  }
}
```

```html
<!-- src/app/app.html -->
@if (isLoading()) {
  <div class="loading-backdrop" aria-busy="true" aria-live="polite">
    <div class="spinner" role="status" aria-label="Loading"></div>
  </div>
}
```

> [!warning] Gotcha
> Spegnere lo spinner solo su `NavigationEnd` **non basta**: vanno gestiti anche `NavigationError` e `NavigationCancel`, altrimenti l'overlay resta appeso quando un guard blocca o c'è un errore.

## Resolver
> 📖 pp.352-354

Problema: se il componente legge l'id dalla rotta e *poi* inizia a caricare, il caricamento parte **dopo** che la navigazione è completata. Il router emette `NavigationEnd` subito, lo spinner basato sugli eventi si spegne troppo presto, e il template deve gestire un valore `null`/`undefined`.

I **resolver** risolvono il problema: sono funzioni (`ResolveFn<T>`) che girano **prima** dell'attivazione della rotta. Il router **attende** la Promise/Observable e solo dopo attiva la rotta. Il valore risolto si legge da `route.data` *oppure* il resolver può **pilotare uno store** che il componente legge.

```ts
// src/app/domains/ticketing/feature-booking/passenger-edit/passenger-resolver.ts
import { inject } from '@angular/core';
import { toObservable } from '@angular/core/rxjs-interop';
import {
  ActivatedRouteSnapshot,
  ResolveFn,
  RouterStateSnapshot,
} from '@angular/router';
import { filter, take } from 'rxjs';
import { PassengerDetailStore } from './passenger-detail-store';

export const passengerResolver: ResolveFn<unknown> = (
  route: ActivatedRouteSnapshot,
  _state: RouterStateSnapshot,
) => {
  const passengerStore = inject(PassengerDetailStore);   // injection context
  const id = route.paramMap.get('id') ?? '0';
  passengerStore.setPassengerId(+id);

  // observable che completa quando il caricamento finisce (o fallisce)
  return toObservable(passengerStore.passengerStatus).pipe(
    filter((status) => status !== 'loading'),
    take(1),
  );
};
```

Si assegna a una chiave nell'oggetto `resolve` della rotta:

```ts
// src/app/domains/ticketing/ticketing.routes.ts
import { passengerResolver } from './feature-booking/passenger-edit/passenger-resolver';

{
  path: 'passenger-edit/:id',
  component: PassengerEdit,
  resolve: {
    passenger: passengerResolver,   // chiave → funzione resolver
  },
}
```

Il router esegue ogni resolver e ne attende il risultato. Due modi di consumarlo:
1. **Via `route.data`**: è un Observable i cui valori contengono i dati risolti sotto le stesse chiavi (`data['passenger']`, …).
2. **Via store** (come qui): il resolver alimenta lo store, il componente fa `inject(PassengerDetailStore)` e si lega ai suoi signal; il resolver garantisce che i dati siano in caricamento/pronti *prima* dell'attivazione.

> [!tip] Take-away
> Resolver = carica **prima** di attivare la rotta. Così il componente trova i dati già disponibili (niente `null` transitorio) e lo spinner basato sui router events resta coerente, perché `NavigationEnd` arriva solo dopo che il resolver ha completato.

## HttpInterceptors
> 📖 pp.354-356

Gli interceptor sono **funzioni** che ispezionano e modificano le richieste HTTP in uscita e le risposte in entrata. Casi tipici: aggiungere header di autenticazione, error handling globale, supportare formati oltre il JSON.

Seguono il pattern **Chain of Responsibility**: ogni interceptor può passare la richiesta alla funzione successiva (`next`); in fondo alla catena parte la richiesta vera.

```ts
// src/app/domains/shared/util-auth/auth.interceptor.ts
import { HttpInterceptorFn, HttpStatusCode } from '@angular/common/http';
import { inject } from '@angular/core';
import { catchError, throwError } from 'rxjs';
import { AuthService } from './auth.service';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);       // injection context

  // req e headers sono immutabili → clona e modifica
  const clonedReq = req.clone({
    headers: req.headers.set(
      'Authorization',
      `Bearer ${authService.authToken()}`,
    ),
  });

  return next(clonedReq).pipe(                    // passa al prossimo della catena
    catchError((error) => {
      if (
        error.status === HttpStatusCode.Unauthorized ||  // 401
        error.status === HttpStatusCode.Forbidden        // 403
      ) {
        console.log('you need to login!');
      }
      return throwError(() => error);
    }),
  );
};
```

Registrazione con `provideHttpClient` + `withInterceptors`:

```ts
// src/app/app.config.ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { authInterceptor } from './domains/shared/util-auth/auth.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    // ...
    provideHttpClient(withInterceptors([authInterceptor])),
    // ...
  ],
};
```

> [!warning] Gotcha
> `HttpRequest` e `HttpHeaders` sono **immutabili**: per modificarli devi `req.clone({ ... })`, non mutarli in place. E l'**ordine** dell'array in `withInterceptors([...])` determina l'ordine della catena: ogni interceptor riceve la richiesta eventualmente già modificata dai precedenti.

Collegamenti: [[inject]] · [[providers]] · [[16-authentication-authorization]] (Bearer token / gestione 401-403).

## 🔁 Ripasso lampo
1. Quando un application initializer **blocca** il bootstrap? Cosa deve ritornare la funzione per farlo attendere ad Angular?
2. Differenza tra `provideAppInitializer`, `provideEnvironmentInitializer` e `providePlatformInitializer`: scope e momento di esecuzione?
3. Cosa ritorna un `CanActivateFn` per redirigere invece di bloccare/consentire? E perché `canActivate` è un array?
4. Quale parametro speciale riceve un `CanDeactivateFn` che un `CanActivateFn` non ha, e a cosa serve?
5. Quali router events vanno gestiti, oltre a `NavigationStart`/`NavigationEnd`, per spegnere correttamente uno spinner?
6. Perché un resolver risolve il problema del "loading indicator spento troppo presto"? Due modi per consumare il valore risolto?
7. Perché in un interceptor devi clonare la richiesta? Cosa stabilisce l'ordine in `withInterceptors([...])`?

**Take-away del capitolo:**
- **Initializers**: `provideAppInitializer` blocca il bootstrap finché il lavoro async (Promise/Observable) finisce; gli environment initializer scopano il setup a rotta/feature; i platform initializer sono per infrastruttura di basso livello.
- **Guards**: funzioni che il router chiama per consentire/negare activation (`CanActivateFn`, ritorna `boolean`/`UrlTree`) e deactivation (`CanDeactivateFn`, riceve il componente, es. form dirty).
- **Router events** (`NavigationStart/End/Cancel/Error`) permettono di reagire alla navigazione (loading overlay).
- **Resolver** (`ResolveFn<T>`) caricano i dati *prima* dell'attivazione della rotta: il router attende, niente valori `null` transitori.
- **HttpInterceptors** (`HttpInterceptorFn`) formano una catena (Chain of Responsibility): clonano/modificano richieste e risposte; registrati con `provideHttpClient(withInterceptors([...]))`.
- Tutti gli hook girano in [[injection-context]] → usano [[inject]] direttamente.
