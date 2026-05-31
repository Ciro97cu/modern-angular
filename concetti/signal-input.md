---
titolo: "input() — InputSignal"
tags: [tipo/concetto, components, signals]
aliases: [input, InputSignal, signal input]
---
# input() — InputSignal

Dichiara un **input di componente come signal di sola lettura**. Sostituisce il decoratore `@Input()`. Si legge come signal (`this.flight()`), è reattivo e usabile in [[computed]]/[[effect]]/template.

```ts
flight = input<Flight>();                 // InputSignal<Flight | undefined>
id = input.required<number>();            // obbligatorio
label = input('', { alias: 'caption' });  // alias + default
count = input(0, { transform: numberAttribute }); // transform
```

> [!tip] Take-away
> `input.required` non ha default e fallisce a compile-time se non passato. Per un input **scrivibile** (two-way) usa invece [[model-signal]].

**Usato in:** [[02-signal-based-components]]
