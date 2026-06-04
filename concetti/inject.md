---
titolo: "inject()"
tags: [tipo/concetto, di, services, angular-22]
aliases: [inject, dependency injection]
---
# inject()

Funzione per ottenere una dipendenza dal DI **senza injection via costruttore**. Va chiamata in un [[injection-context]] (campi di classe, costruttore, factory di provider, `runInInjectionContext`).

```ts
export class FlightService {
  private http = inject(HttpClient);
  private cfg = inject(APP_CONFIG);
}
```

> [!tip] Take-away
> Più componibile dei parametri del costruttore: abilita funzioni helper riusabili (es. guardie, resolver, "injectable functions"). Cosa è disponibile dipende dalla gerarchia dei [[providers]]. I servizi si dichiarano con [[service|@Service()]].

> [!info] Angular 22+ · injectAsync
> **`injectAsync(() => import(...).then(m => m.Svc))`** inietta un servizio in modo **lazy**: il bundle viene caricato solo alla prima chiamata della funzione ritornata (che dà una `Promise`). Opzione `prefetch` (es. `onIdle()`) per pre-caricare a browser idle. Il servizio target deve essere auto-provided ([[service|@Service()]]).
> ```ts
> private readonly upgradeService = injectAsync(
>   () => import('./upgrade-service').then((m) => m.UpgradeService),
>   { prefetch: () => onIdle() },
> );
> // ...
> const svc = await this.upgradeService();
> ```

**Usato in:** [[05-state-management-services-signals]], [[12-initialization-route-changes]]
