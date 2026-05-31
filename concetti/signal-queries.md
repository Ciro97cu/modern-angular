---
titolo: "Signal queries (viewChild / contentChild)"
tags: [tipo/concetto, components, signals]
aliases: [viewChild, viewChildren, contentChild, contentChildren]
---
# Signal queries

Recuperano riferimenti a elementi/componenti come **signal**, sostituendo i decoratori `@ViewChild`/`@ContentChild`.

- **`viewChild` / `viewChildren`** → elementi del **proprio template** (la *view*).
- **`contentChild` / `contentChildren`** → contenuto proiettato via [[content-projection]] (il *content*).

```ts
title = viewChild<ElementRef>('title');     // Signal<ElementRef | undefined>
items = contentChildren(ItemComponent);     // Signal<readonly ItemComponent[]>
first = viewChild.required(ChildCmp);        // versione required
```

> [!warning] Gotcha
> Disponibili dopo il rendering della rispettiva fase; leggili in [[effect]] o nei hook appropriati, non nel costruttore. Il libro invita a **mettere in discussione** l'abuso di `viewChild` (preferire input/output e servizi).

**Usato in:** [[10-signal-queries-component-communication]], [[11-directives-templates-containers]]
