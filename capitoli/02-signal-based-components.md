---
capitolo: 2
titolo: "Signal-Based Components"
pagine: "32-72"
tags: [tipo/capitolo, components, signals, templates, http, angular-22]
---
# 02 · Signal-Based Components
> 📖 cap.2 · pp.32-72 — *Modern Angular* v2.0.0

Si costruisce una feature di **ricerca voli** (`flight-search`) e si introducono i mattoni dei componenti signal-based: data model, logica con [[signal]], template syntax, accesso ai dati (HttpClient vs [[resource|httpResource]]) e sotto-componenti con input/output.

Struttura cartelle a **subdomain** (approfondita nel [[08-sustainable-architectures|cap.8]]): `src/app/domains/ticketing/feature-booking/flight-search`.

## Scaffolding & Style Guide
> 📖 pp.32-34

```bash
ng g c domains/ticketing/feature-booking/flight-search
```

Genera `flight-search.ts` + `flight-search.html` (niente stylesheet/test per via della config del [[01-getting-started|cap.1]]).

> [!tip] Take-away
> Lo Style Guide aggiornato **non usa più i suffissi** `Component`/`.component.ts`: la classe è `FlightSearch`, il file `flight-search.ts`. Si possono usare suffissi semantici propri (`search`, `edit`).

## Data model
> 📖 pp.34-35

Interfacce in `domains/ticketing/data/`. Le date sono **stringhe ISO**. Le costanti `initial*` evitano `null`/`undefined`.

```ts
// flight.ts
export interface Flight {
  id: number; from: string; to: string; date: string;
  delayed: boolean; delay: number; aircraft: Aircraft; prices: Price[];
}
export const initialFlight: Flight = { id: 0, from: '', to: '', date: '', delayed: false, delay: 0, aircraft: initialAircraft, prices: [] };
```

## Component logic con i signal
> 📖 pp.35-38

```ts
@Component({
  selector: 'app-flight-search',
  imports: [],
  templateUrl: './flight-search.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class FlightSearch {
  protected readonly filter = signal({ from: 'Hamburg', to: 'Graz' });
  protected readonly filterForm = form(this.filter);            // Signal Forms (cap.6)
  protected readonly flights = signal<Flight[]>([]);            // tipo esplicito: array vuoto non lo inferisce
  protected readonly selectedFlight = signal<Flight | null>(null);

  protected search(): void { /* ... */ }
  protected select(f: Flight): void { this.selectedFlight.set(f); }
}
```

- Stato = [[signal]]. Si legge chiamandolo (`this.filter().from`), si scrive con `.set()` o `.update(prev => next)`.
- Un signal **ha sempre un valore**; il tipo si inferisce dal default (esplicitalo quando non basta, es. `signal<Flight[]>([])`).
- `signal()` → `WritableSignal<T>` (getter+setter); `Signal<T>` è sola lettura. Esporre read-only: `counter.asReadonly()`.

```ts
const counter = signal(0);
counter.update((c) => c + 1);
const ro = counter.asReadonly();
```

Collegamenti: [[signal]] · [[06-signal-forms]] (la `form()`).

## Template & data binding
> 📖 pp.40-46

```html
<input [formField]="filterForm.from" id="from" />
<button type="button" (click)="search()" [disabled]="!filter().from || !filter().to">Search</button>

@if (flights().length > 0) {
  <table>
    @for (flight of flights(); track flight.id) {
      <tr [class.selected]="selectedFlight() === flight"
          [style.color]="flight.delayed ? 'darkred' : 'black'"
          (click)="select(flight)" class="selectable">
        <td>{{ flight.id }}</td>
        <td>{{ flight.date | date }}</td>
      </tr>
    } @empty {
      <tr><td colspan="5">No flights found.</td></tr>
    }
  </table>
}
<pre>{{ selectedFlight() | json }}</pre>
```

Tipi di binding:
- **Event** `(click)="..."` → DOM event o evento di sotto-componente.
- **Property** `[disabled]="..."` → proprietà del DOM/sotto-componente.
- **Interpolation** `{{ expr }}` + **pipe** (`| date`, `| json`, `| number:'1.2'` → serve `DecimalPipe`).
- **Conditional styling**: `[class.x]="cond"`, `[style.color]="..."`.

**Control flow built-in** (`@if`, `@for`, `@switch`, prefisso `@`):

```html
@switch (passenger().passengerStatus) {
  @case ('SEN') @case ('A') { <p>Senator</p> }   <!-- multi-value da Angular 21.1 -->
  @default { <p>Regular</p> }
}
```

> [!info] Angular 22+ · Exhaustive `@switch`
> Quando l'espressione è una **literal union**, il ramo `@default never;` chiede al compilatore un **exhaustiveness check**: se in futuro aggiungi un valore alla union senza il relativo `@case`, il template **non compila**. (Forma base da **Angular 21.2**.)
> ```html
> @switch (passenger().passengerStatus) {  <!-- 'A' | 'B' | 'C' -->
>   @case ('A') { <p>Senator</p> }
>   @case ('B') { <p>Frequent Traveller</p> }
>   @case ('C') { <p>Regular</p> }
>   @default never;
> }
> ```
> Se invece fai switch su una **proprietà** di una discriminated union (non sulla union intera), usa `never(<expression>)` (**Angular 22**) per dichiarare l'esaustività.

> [!warning] Gotcha
> Nel `@for` il **`track` è obbligatorio** (es. `track flight.id`; in mancanza, `track $index` o `track item`): permette ad Angular di spostare i nodi DOM invece di ri-renderizzare tutto.

> [!warning] Gotcha
> Usa `<button type="button">`: il default è `submit` e farebbe il reload della pagina. Il `formRoot` del [[06-signal-forms|cap.6]] disabilita questo comportamento.

## Chiamare il componente
> 📖 pp.46-47

Importa `FlightSearch` nell'array `imports` di `App` e usalo in `app.html` con `<app-flight-search />`. Il prefisso del selector (`app-`) si cambia in `angular.json` (chiave `prefix`, idem `eslint.config.js`).

## Debugging
> 📖 pp.47-50

- Errori in **console** del browser: Angular stampa link cliccabili al file HTML/TS.
- **Source Maps** generate da `ng serve` → debugger del browser (Chrome → Sources, `Cmd+Shift+P` per aprire file).
- **VS Code**: breakpoint nel `.ts` + F5; config già in `.vscode/launch.json` generata dalla CLI.

## Data access: HttpClient
> 📖 pp.50-54

```ts
private readonly http = inject(HttpClient);

protected search(): void {
  const url = 'https://demo.angulararchitects.io/api/flight';
  const params = { from: this.filter().from, to: this.filter().to };
  this.http.get<Flight[]>(url, { params }).subscribe({
    next: (flights) => this.flights.set(flights),
    error: (err) => console.error('Error', err),
  });
}
```

- `get<T>` ritorna un **Observable** → `subscribe` con `next`/`error`. `T` è il tipo di risposta.
- Metodi: `get` / `post(body)` / `put(body)` / `patch(body)` / `delete` / `request(method,...)`.
- `options` per `params`, `headers`, ecc.

Collegamenti: [[inject]].

## Data access: httpResource (signal-based)
> 📖 pp.54-59

```ts
protected readonly flightsResource = httpResource<Flight[]>(
  () => ({
    url: 'https://demo.angulararchitects.io/api/flight',
    params: { from: this.filter().from, to: this.filter().to },
  }),
  { defaultValue: [] },
);
protected readonly flights = this.flightsResource.value;
protected readonly error = this.flightsResource.error;
protected readonly isLoading = this.flightsResource.isLoading;

protected search(): void { this.flightsResource.reload(); }
```

- La lambda è **reattiva**: cambia un signal letto al suo interno → la resource ri-carica. Parte subito alla creazione.
- Per **disattivarla/condizionarla** ritorna `undefined` dalla lambda (usa il `return` esplicito).
- `{ defaultValue: [] }` evita `undefined` nel template. `value` è **scrivibile** (copia di lavoro locale; per persistere serve comunque `HttpClient`).
- Stati via `status()`: `idle · loading · reloading · error · resolved · local`.

> [!warning] Gotcha
> In stato di errore **non leggere `value()`** (lancia): controlla prima `error()`. Nel template: `@if (!error() && flights().length > 0)`.

> [!tip] Take-away
> `httpResource` gestisce da solo le **race condition** (tiene solo l'ultima richiesta, come `switchMap`); `reload()` invece ignora le chiamate sovrapposte (come `exhaustMap`). È pensata per il **fetch (GET)**: per save/delete usa `HttpClient`.

Collegamenti: [[resource]] · approfondimento in [[03-reactive-design-with-signals]].

## Sotto-componenti: input, output, model
> 📖 pp.59-71

Si estrae un `FlightCard`. Interfaccia pubblica vista dal padre:

```html
<app-flight-card [item]="flight" [selected]="basket()[flight.id]"
                 (selectedChange)="updateBasket(flight.id, $event)" />
```

Aggiornamento **immutabile** del basket (nuovo riferimento → change detection):

```ts
protected readonly basket = signal<Record<number, boolean>>({ 3: true, 5: true });
protected updateBasket(id: number, selected: boolean): void {
  this.basket.update((b) => ({ ...b, [id]: selected }));
}
```

Componente con [[signal-input|input()]] e [[signal-output|output()]]:

```ts
export class FlightCard {
  readonly item = input.required<Flight>();   // obbligatorio, niente default
  readonly selected = input(false);           // opzionale, default
  readonly selectedChange = output<boolean>(); // OutputEmitterRef<boolean>
  protected select()   { this.selectedChange.emit(true); }
  protected deselect() { this.selectedChange.emit(false); }
}
```

- Gli input sono **signal di sola lettura** (`InputSignal<T>`): nel template chiamali (`item()`), o usa `@let v = item()`.
- Nell'event binding il valore emesso arriva in **`$event`**.

**ModelSignal** (input scrivibile → two-way):

```ts
readonly selected = model(false);  // crea selected + selectedChange
// uso nel padre: <app-flight-card [(selected)]="..."/>  oppure  [selected]+(selectedChange)
```

**Two-way binding** "banana in a box" — zucchero per property + event con naming `prop`/`propChange`:

```html
<app-simple-delay-stepper [(value)]="maxDelay" />
<!-- equivale a -->
<app-simple-delay-stepper [value]="maxDelay()" (valueChange)="maxDelay.set($event)" />
```

Funziona sia con `model()` sia con `input.required` + `output` (purché l'event sia `valueChange` ed emetta il nuovo valore). Nel `[(value)]="maxDelay"` si passa il signal **senza** chiamarne il getter.

**Content projection** con `<ng-content>` (default content + slot multipli via `select=".classe"`):

```html
<div class="card-body">
  <ng-content select=".detail" />
  <ng-content select=".button" />
  <ng-content select=".footer">All rights reserved.</ng-content>  <!-- default -->
</div>
```

Collegamenti: [[signal-input]] · [[signal-output]] · [[model-signal]] · [[two-way-binding]] · [[content-projection]] · approfondimento comunicazione in [[10-signal-queries-component-communication]].

## 🔁 Ripasso lampo
1. Perché `signal<Flight[]>([])` ha il tipo esplicito mentre `signal({from,to})` no?
2. Quando usare `HttpClient` e quando `httpResource`? Cosa garantisce `httpResource` sulle race condition?
3. Perché il `track` nel `@for` è obbligatorio e cosa puoi passargli?
4. Differenza tra `input()`, `output()` e `model()`? Cos'è `$event` in un event binding?
5. Cosa fa `[(value)]="x"` sotto il cofano? Quali condizioni di naming richiede?
6. Perché aggiorni il basket con lo spread invece di mutarlo?

**Take-away del capitolo:**
- Lo stato dei componenti si modella con **signal**; template syntax = interpolation/property/event/two-way + control flow `@if/@for/@switch` + pipe.
- Due vie ai dati: `HttpClient` (Observable, tutti i verbi) e `httpResource` (signal-based, fetch reattivo con stati e gestione race condition).
- Comunicazione padre-figlio con `input`/`output`/`model` (tutti basati su signal) + content projection.
