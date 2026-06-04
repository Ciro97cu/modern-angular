---
titolo: "@Service decorator"
tags: [tipo/concetto, di, services, angular-22]
aliases: [Service, autoProvided, Injectable vs Service]
---
# @Service()

Decoratore (**Angular 22**) per marcare una classe come servizio iniettabile. Di default **registra la classe nel root injector** → singleton applicativo, senza dover scrivere `providedIn: 'root'`. È la forma usata in tutto il libro dalla 2ª edizione.

```ts
import { Service } from '@angular/core';

@Service()                       // singleton in root, default
export class FlightClient {}

@Service({ autoProvided: false }) // NON registrato in root: va fornito via providers
export class BrowserLanguageService implements LanguageService {}
```

> [!info] Angular 22+
> `@Service()` sostituisce `@Injectable({ providedIn: 'root' })`. **Semantica identica**, solo più conciso. Mappa pre-22:
> - `@Service()` ≡ `@Injectable({ providedIn: 'root' })`
> - `@Service({ autoProvided: false })` ≡ `@Injectable()` (senza `providedIn`, da fornire a mano)
>
> Per servizi scambiabili tramite [[providers]] (es. dietro un abstract class come token) si usa `autoProvided: false`. `ng update` riscrive automaticamente i decoratori durante il bump.

> [!tip] Take-away
> `@Service()` di default = root singleton. Per la lazy injection con `injectAsync` il servizio **deve** essere auto-provided (`@Service()` semplice). Vedi [[inject]] e [[providers]].

**Usato in:** [[05-state-management-services-signals]], [[09-ngrx-signal-store]], [[12-initialization-route-changes]], [[08-sustainable-architectures]]
