---
titolo: "resource / httpResource / rxResource"
tags: [tipo/concetto, signals, reactivity, http]
aliases: [resource, httpResource, rxResource]
---
# resource()

Famiglia di primitive per **dati asincroni reattivi**. Una resource lega una richiesta a dei signal sorgente: quando questi cambiano, **rilancia** automaticamente e espone `value()`, `status()`, `error()` e `isLoading()` come signal.

- **`httpResource`** — dichiarativa, basata su `HttpClient`; passi una URL/request reattiva.
- **`rxResource`** — basata su un `loader` che ritorna un `Observable`.
- **`resource`** — generica, `loader` Promise-based (con `abortSignal`).

```ts
const id = signal(1);
const flight = httpResource<Flight>(() => `/api/flight/${id()}`);
// flight.value(), flight.isLoading(), flight.error(), flight.reload()
```

> [!warning] Gotcha
> La richiesta riparte ad ogni cambio delle dipendenze lette nella funzione sorgente. `httpResource` è pensata per **GET/read**; per le mutazioni usa `HttpClient` o le mutations dello store.

**Usato in:** [[02-signal-based-components]], [[03-reactive-design-with-signals]], [[09-ngrx-signal-store]]
