---
titolo: "signal()"
tags: [tipo/concetto, signals, reactivity]
aliases: [WritableSignal, segnale]
---
# signal()

Primitiva di stato reattivo scrivibile. `signal(initial)` crea un `WritableSignal`: lo **leggi chiamandolo** come funzione (`count()`), e lo aggiorni con `.set(v)` o `.update(fn)`. Ogni lettura dentro un contesto reattivo ([[reactive-context]]) registra una dipendenza, così computed/effect/template si ricalcolano da soli al cambiamento.

```ts
const count = signal(0);
count();             // lettura → 0
count.set(5);        // scrittura
count.update(n => n + 1); // 6
```

> [!warning] Gotcha
> Notifica i dipendenti solo se il **valore cambia** secondo l'[[equality-immutability|equality]] (default `Object.is`). Mutare un oggetto in place (`obj.x = 1`) NON notifica: usa `.set`/`.update` con un nuovo riferimento.

**Usato in:** [[02-signal-based-components]], [[03-reactive-design-with-signals]], [[05-state-management-services-signals]]
