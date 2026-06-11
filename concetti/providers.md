---
titolo: "Providers"
tags: [tipo/concetto, di, services, angular-22]
aliases: [provider, useClass, useValue, useFactory, providedIn]
---
# Providers

Configurano **cosa** restituisce il DI per un token. Definibili a livello root (via [[service|@Service()]] / pre-22 `providedIn: 'root'`), di applicazione, di route o di componente — la **gerarchia** determina visibilità e ciclo di vita.

```ts
providers: [
  FlightService,                                  // short-hand (useClass implicito)
  { provide: API, useClass: RealApi },
  { provide: TOKEN, useValue: 42 },
  { provide: X, useFactory: () => new X(inject(Dep)) },
  { provide: Y, useExisting: Z },
]
```

> [!tip]
> Provider a livello **componente/route** creano istanze locali (utile per stato per-feature, vedi [[lightweight-store]]); root crea singleton applicativi. Esistono **provider functions** (`provideHttpClient()`, `provideRouter()`) come API idiomatica.

> [!info] Angular 22+
> Lo short-hand `providers: [FlightClient]` (token = implementazione) è equivalente a [[service|@Service()]] sulla classe — pre-22 era `@Injectable({ providedIn: 'root' })`. Per servizi scambiabili dietro un base type si usa `@Service({ autoProvided: false })` + provider esplicito.

**Usato in:** [[05-state-management-services-signals]], [[12-initialization-route-changes]]
