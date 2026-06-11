---
titolo: "Equality & immutability"
tags: [tipo/concetto, signals, reactivity]
aliases: [equality, immutability, glitch-free]
---
# Equality & immutability

Un [[signal]] notifica i dipendenti **solo se il nuovo valore è diverso** dal precedente. Il confronto default è `Object.is`, personalizzabile con l'opzione `equal`. Per questo lo stato va trattato in modo **immutabile**: sostituire il riferimento, non mutarlo in place.

```ts
// ❌ non notifica: stesso riferimento
list().push(x);
// ✅ notifica: nuovo array
list.update(l => [...l, x]);
```

> [!warning]
> Mutazioni in place (`obj.prop = ...`, `arr.push(...)`) lasciano i computed/effect/UI **non aggiornati**. È anche alla base della proprietà **glitch-free**: i consumatori vedono solo valori coerenti e stabilizzati.

**Usato in:** [[03-reactive-design-with-signals]]
