---
titolo: "output() — OutputEmitterRef"
tags: [tipo/concetto, components, signals]
aliases: [output, OutputEmitterRef, signal output]
---
# output() — OutputEmitterRef

Dichiara un **evento emesso dal componente** verso il padre. Sostituisce `@Output() EventEmitter`. Ritorna un `OutputEmitterRef`; emetti con `.emit(value)`.

```ts
flightChange = output<Flight>();
// ...
this.flightChange.emit(updated);
```
```html
<flight-card (flightChange)="onChange($event)" />
```

> [!tip]
> Coppia naturale di [[signal-input]] per il pattern "input giù, event su". Per il caso bidirezionale combinato (`[(x)]`) c'è [[model-signal]] / [[two-way-binding]].

**Usato in:** [[02-signal-based-components]]
