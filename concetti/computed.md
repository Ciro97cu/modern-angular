---
titolo: "computed()"
tags: [tipo/concetto, signals, reactivity]
aliases: [computed signal, segnale derivato]
---
# computed()

Signal **di sola lettura derivato** da altri signal. La funzione passata viene rieseguita solo quando una delle dipendenze lette cambia; il risultato è **memoizzato** e il calcolo è **lazy** (avviene alla prima lettura, non alla creazione).

```ts
const price = signal(100);
const qty = signal(2);
const total = computed(() => price() * qty()); // 200, ricalcolato on-demand
```

> [!tip]
> Le dipendenze sono **auto-tracciate** ([[reactive-context]]): vengono raccolte solo i signal effettivamente letti durante l'esecuzione. Un ramo non eseguito (dentro un `if`) non crea dipendenza.

> [!warning]
> Deve essere **puro**: niente side-effect, niente `.set()` su altri signal. Per gli effetti collaterali usa [[effect]].

**Usato in:** [[03-reactive-design-with-signals]], [[02-signal-based-components]]
