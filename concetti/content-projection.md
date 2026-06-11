---
titolo: "Content projection (ng-content)"
tags: [tipo/concetto, components, templates]
aliases: [ng-content, content projection, proiezione del contenuto]
---
# Content projection (ng-content)

Permette a un componente di **renderizzare contenuto fornito dal padre** tramite `<ng-content>`. Con l'attributo `select` si creano **slot multipli** in base a selettori CSS.

```html
<!-- card.html -->
<header><ng-content select="[card-title]" /></header>
<section><ng-content /></section>   <!-- slot di default -->
```
```html
<app-card>
  <h2 card-title>Titolo</h2>
  <p>corpo proiettato</p>
</app-card>
```

> [!tip]
> Il contenuto proiettato resta nel contesto del **padre** (binding e lifecycle del padre). Per interagirci da codice → [[signal-queries]] (`contentChild`). Versione programmatica con `ng-template`/`ViewContainerRef` nel cap. direttive.

**Usato in:** [[02-signal-based-components]], [[10-signal-queries-component-communication]], [[11-directives-templates-containers]]
