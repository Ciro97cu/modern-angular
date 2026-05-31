---
titolo: "linkedSignal()"
tags: [tipo/concetto, signals, reactivity]
aliases: [linkedSignal, signal collegato]
---
# linkedSignal()

Signal **scrivibile ma derivato**: ha un valore calcolato da una sorgente, ma puoi anche sovrascriverlo con `.set`/`.update`. Quando la sorgente cambia, **si reinizializza** al valore derivato (perdendo l'override). Utile per stato locale che dipende da un input ma deve restare editabile (es. selezione che si resetta al cambio lista).

```ts
const options = signal(['A', 'B']);
const selected = linkedSignal(() => options()[0]); // derivato, ma settabile
selected.set('B');        // override locale
options.set(['X', 'Y']);  // selected torna a 'X'
```

> [!tip] Take-away
> Sta a metà tra [[signal]] (scrivibile) e [[computed]] (derivato): scrivibile **e** reattivo alla sorgente.

**Usato in:** [[03-reactive-design-with-signals]]
