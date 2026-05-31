---
titolo: "Providers"
tags: [tipo/concetto, di, services]
aliases: [provider, useClass, useValue, useFactory, providedIn]
---
# Providers

Configurano **cosa** restituisce il DI per un token. Definibili a livello root (`providedIn: 'root'`), di applicazione, di route o di componente — la **gerarchia** determina visibilità e ciclo di vita.

```ts
providers: [
  FlightService,                                  // short-hand (useClass implicito)
  { provide: API, useClass: RealApi },
  { provide: TOKEN, useValue: 42 },
  { provide: X, useFactory: () => new X(inject(Dep)) },
  { provide: Y, useExisting: Z },
]
```

> [!tip] Take-away
> Provider a livello **componente/route** creano istanze locali (utile per stato per-feature, vedi [[lightweight-store]]); root crea singleton applicativi. Esistono **provider functions** (`provideHttpClient()`, `provideRouter()`) come API idiomatica.

**Usato in:** [[05-state-management-services-signals]], [[12-initialization-route-changes]]
