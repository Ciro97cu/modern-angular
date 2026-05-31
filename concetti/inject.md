---
titolo: "inject()"
tags: [tipo/concetto, di, services]
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
> Più componibile dei parametri del costruttore: abilita funzioni helper riusabili (es. guardie, resolver, "injectable functions"). Cosa è disponibile dipende dalla gerarchia dei [[providers]].

**Usato in:** [[05-state-management-services-signals]], [[12-initialization-route-changes]]
