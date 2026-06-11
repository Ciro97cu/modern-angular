---
titolo: "untracked()"
tags: [tipo/concetto, signals, reactivity]
aliases: [untracked]
---
# untracked()

Legge un signal **senza creare una dipendenza** nel contesto reattivo corrente ([[reactive-context]]). Serve dentro [[computed]]/[[effect]] quando vuoi usare il valore attuale di un signal ma non vuoi che le sue variazioni facciano rieseguire il calcolo/effetto.

```ts
effect(() => {
  const c = count();              // dipendenza → ri-esegue al cambio
  untracked(() => log(user()));   // legge user senza dipenderne
});
```

> [!tip]
> Tipico per loggare/accedere a contesto ausiliario senza accoppiare l'effetto a quel signal.

**Usato in:** [[03-reactive-design-with-signals]]
