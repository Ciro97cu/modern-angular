---
titolo: "model() — ModelSignal"
tags: [tipo/concetto, components, signals]
aliases: [model, ModelSignal, writable input]
---
# model() — ModelSignal

Input **scrivibile**: combina un [[signal-input]] e un [[signal-output]] (`<nome>Change`) per abilitare il **two-way binding** ([[two-way-binding]]). Lo modifichi con `.set`/`.update` dal componente e la modifica risale al padre.

```ts
value = model<string>('');     // ModelSignal<string>
value.set('ciao');             // emette automaticamente valueChange
```
```html
<my-input [(value)]="text" />  <!-- bind bidirezionale -->
```

> [!tip]
> `model` = input + output sincronizzati. Usalo per componenti tipo form-control/custom input.

**Usato in:** [[02-signal-based-components]], [[06-signal-forms]]
