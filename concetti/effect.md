---
titolo: "effect()"
tags: [tipo/concetto, signals, reactivity]
aliases: [effetto, side effect]
---
# effect()

Esegue **side-effect** in reazione ai signal letti al suo interno. Gira una prima volta e poi ad ogni cambiamento delle dipendenze auto-tracciate ([[reactive-context]]). Va creato in un [[injection-context]] (o passando un `Injector`); si pulisce da solo alla distruzione.

```ts
effect((onCleanup) => {
  console.log('count =', count());
  const id = setInterval(...);
  onCleanup(() => clearInterval(id)); // cleanup opzionale
});
```

> [!warning]
> Non è il posto per **derivare stato** (usa [[computed]]) né per sincronizzare due signal a vicenda (rischio loop). Scrivere su un signal dentro un effect è sconsigliato; se serve, valuta [[linked-signal]]. Per leggere un signal senza dipenderne usa [[untracked]].

**Usato in:** [[03-reactive-design-with-signals]]
