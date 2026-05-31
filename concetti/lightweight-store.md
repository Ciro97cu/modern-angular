---
titolo: "Lightweight store (signal-based)"
tags: [tipo/concetto, state-management, signals, architecture]
aliases: [store, signal store pattern, lightweight store]
---
# Lightweight store (signal-based)

Pattern di **state management** senza librerie: un service `@Injectable` che incapsula lo stato in [[signal]] privati ed espone signal/[[computed]] di sola lettura + metodi per le modifiche. Garantisce **unidirectional data flow** (la UI legge i derivati, scrive solo via metodi).

```ts
@Injectable()
export class FlightStore {
  private _flights = signal<Flight[]>([]);
  readonly flights = this._flights.asReadonly();
  readonly count = computed(() => this._flights().length);
  load(criteria: string) { /* set _flights */ }
}
```

> [!tip] Take-away
> Granularità e posizione contano: store per-feature forniti a livello [[providers|route/componente]] per scope e auto-cleanup; evita cicli/ridondanze tra store. Per esigenze più ricche → [[09-ngrx-signal-store|NgRx Signal Store]].

**Usato in:** [[05-state-management-services-signals]], [[08-sustainable-architectures]]
