---
titolo: "resource / httpResource / rxResource"
tags: [tipo/concetto, signals, reactivity, http, angular-22]
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

> [!info] Angular 22+ · Snapshots
> Ogni resource espone un signal **`snapshot()`** che impacchetta `status` + `value` in un unico oggetto. Lo si può trasformare con un [[linked-signal]] e ri-convertire in resource con **`resourceFromSnapshots`** → resource derivate da altre resource (prima era possibile solo per singola proprietà).
> ```ts
> // mantieni l'ultimo valore valido durante un reload
> const derived = linkedSignal<ResourceSnapshot<T>, ResourceSnapshot<T>>({
>   source: input.snapshot,
>   computation: (snap, previous) =>
>     snap.status === 'loading' && previous?.value?.status === 'resolved'
>       ? { ...snap, value: previous.value.value }
>       : snap,
> });
> return resourceFromSnapshots(derived);
> ```

> [!info] Angular 22+ · debounced
> I signal non hanno nozione di tempo. **`debounced(sig, ms)`** prende un signal e ritorna una *resource* che insegue il signal con ritardo configurabile (`status() === 'loading'` mentre il valore si assesta). Per i form esiste l'helper dedicato `debounce()` di `@angular/forms/signals` (vedi [[06-signal-forms]]).
> ```ts
> const debouncedFilter = debounced(filter, 300); // 300ms
> ```

**Usato in:** [[02-signal-based-components]], [[03-reactive-design-with-signals]], [[09-ngrx-signal-store]]
