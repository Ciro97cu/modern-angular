---
titolo: "Injection context"
tags: [tipo/concetto, di]
aliases: [injection context, runInInjectionContext]
---
# Injection context

È l'ambito in cui [[inject]] (ed `effect`, `toSignal`, ecc.) sono utilizzabili perché esiste un `Injector` attivo: durante la **costruzione** di componenti/servizi/direttive, nelle **factory** dei provider, e dentro `runInInjectionContext(injector, fn)`.

```ts
constructor() {
  effect(() => ...);      // ok: injection context
}
ngOnInit() {
  // inject() qui fallisce → fuori dal context
  runInInjectionContext(this.injector, () => inject(X));
}
```

> [!warning] Gotcha
> Chiamare `inject()` fuori contesto (es. in un callback async o in un metodo di lifecycle) lancia un errore. Cattura l'`Injector` con `inject(Injector)` se ti serve più tardi.

**Usato in:** [[05-state-management-services-signals]]
