---
titolo: "Reactive context & auto-tracking"
tags: [tipo/concetto, signals, reactivity]
aliases: [auto-tracking, contesto reattivo, signal graph]
---
# Reactive context & auto-tracking

Il **contesto reattivo** è l'esecuzione di una funzione tracciante ([[computed]], [[effect]], template) in cui ogni signal **letto** viene registrato come dipendenza. Questo "auto-tracking" costruisce il **signal graph**: alla modifica di un signal, solo i nodi che ne dipendono si ricalcolano.

- Le dipendenze sono **dinamiche**: dipende solo da ciò che è stato letto in quell'esecuzione (un ramo non preso non crea dipendenza).
- Angular è **glitch-free**: niente stati intermedi incoerenti propagati ai consumatori (vedi [[equality-immutability]]).
- Per leggere senza dipendere → [[untracked]].

> [!tip] Take-away
> Pensare in termini di **grafo dei signal** (sorgenti → derivati → effetti/UI) è il modello mentale chiave del reactive design.

**Usato in:** [[03-reactive-design-with-signals]]
