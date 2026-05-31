---
titolo: "Two-way binding [(x)]"
tags: [tipo/concetto, components, templates]
aliases: [two-way, banana in a box]
---
# Two-way binding [(x)]

La sintassi **"banana in a box"** `[(prop)]="expr"` è zucchero per un property binding `[prop]` + un event binding `(propChange)`. Funziona su una proprietà esposta come [[model-signal]] (o sul classico `prop`/`propChange`).

```html
<my-input [(value)]="text" />
<!-- equivale a -->
<my-input [value]="text" (valueChange)="text.set($event)" />
```

> [!warning] Gotcha
> Richiede la convenzione di naming `prop` + `propChange`. Con i signal, `model()` la fornisce out-of-the-box.

**Usato in:** [[02-signal-based-components]]
