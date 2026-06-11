---
capitolo: 2
titolo: "Signal-Based Components"
pagine: "32-72"
tags: [tipo/capitolo, components, signals, templates, http, angular-22]
---
# 02 · Signal-Based Components
> 📖 cap.2 · pp.32-72 — *Modern Angular* v2.0.0

Si estende l'app del [[01-getting-started|cap.1]] con una feature di **ricerca voli** (`flight-search`): l'utente cerca collegamenti e seleziona un volo. Lungo il percorso si incontrano i mattoni di un componente signal-based: data model, logica con [[signal]], template syntax e control flow, accesso ai dati (`HttpClient` vs [[resource|httpResource]]) e sotto-componenti con `input`/`output`/`model`.

La struttura cartelle è a **subdomain** (approfondita nel [[08-sustainable-architectures|cap.8]]): l'app si suddivide in sottodomini che mappano aree del mondo reale. Qui si parte dal dominio `ticketing` con la feature `feature-booking`, sotto `src/app/domains/ticketing/feature-booking/flight-search`.

## Scaffolding & Style Guide
> 📖 pp.32-34

```bash
ng generate component domains/ticketing/feature-booking/flight-search
# in forma abbreviata:
ng g c domains/ticketing/feature-booking/flight-search
```

Per via della config del [[01-getting-started|cap.1]] (in `angular.json`) la CLI genera solo `flight-search.ts` (la classe del componente) + `flight-search.html` (il template): niente stylesheet né file di test, così l'esempio resta pulito.

> [!tip]
> Lo Style Guide aggiornato **non usa più i suffissi** `Component`/`.component.ts`: la classe è `FlightSearch`, il file `flight-search.ts` (non più `FlightSearchComponent` / `flight-search.component.ts`). Puoi comunque usare suffissi semantici tuoi più informativi di "Component", es. `search` o `edit`.

## Data model
> 📖 pp.34-35

Le interfacce del modello (`Flight`, `Aircraft`, `Price`) stanno in `domains/ticketing/data/`. Le date sono **stringhe ISO** (es. `2030-12-24T17:00+01:00`). Le costanti `initial*` danno un valore di partenza comodo per evitare `null`/`undefined` quando si creano nuovi oggetti.

```ts
// src/app/domains/ticketing/data/flight.ts
import { Aircraft, initialAircraft } from './aircraft';
import { Price } from './price';

export interface Flight {
  id: number;
  from: string;
  to: string;
  date: string;
  delayed: boolean;
  delay: number;
  aircraft: Aircraft;
  prices: Price[];
}

export const initialFlight: Flight = {
  id: 0,
  from: '',
  to: '',
  date: '',
  delayed: false,
  delay: 0,
  aircraft: initialAircraft,
  prices: [],
};
```

```ts
// src/app/domains/ticketing/data/aircraft.ts
export interface Aircraft {
  type: string;
  registration: string;
}

export const initialAircraft: Aircraft = {
  registration: '',
  type: '',
};
```

```ts
// src/app/domains/ticketing/data/price.ts
export interface Price {
  flightClass: string;
  amount: number;
}
```

## Component logic con i signal
> 📖 pp.35-38

Lo scaffold di partenza è una classe vuota; la si espande per gestire la ricerca con i [[signal]] e si usa subito **Signal Forms** (cap.6) per il form dei criteri.

```ts
// src/app/domains/ticketing/feature-booking/flight-search/flight-search.ts
import { Component, ChangeDetectionStrategy, signal } from '@angular/core';
import { form } from '@angular/forms/signals';
import { Flight } from '../../data/flight';

@Component({
  selector: 'app-flight-search',
  imports: [],
  templateUrl: './flight-search.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class FlightSearch {
  protected readonly filter = signal({
    from: 'Hamburg',
    to: 'Graz',
  });
  protected readonly filterForm = form(this.filter);       // Signal Forms (cap.6)
  protected readonly flights = signal<Flight[]>([]);       // tipo esplicito: l'array vuoto non lo inferisce
  protected readonly selectedFlight = signal<Flight | null>(null);

  protected search(): void {
    // implementazione più avanti
  }
  protected select(f: Flight): void {
    this.selectedFlight.set(f);
  }
}
```

- Tutto ciò che il template usa è esposto come **proprietà** o **metodo**. Lo stato si modella con i [[signal]]: legati al template, avvisano Angular dei cambi di valore così che il framework aggiorni i binding.
- Un signal si crea con `signal()` (da `@angular/core`) passando un valore iniziale: **ha sempre un valore**. Il tipo si inferisce dal default; esplicitalo quando il default non basta — es. `signal<Flight[]>([])`, perché un array vuoto non porta informazione sul tipo degli elementi.
- Il `filter` rappresenta i criteri di ricerca; `form(this.filter)` ne costruisce il `filterForm` (Signal Forms) a cui si legano gli input: a ogni modifica dei controlli, sia `filterForm` sia `filter` si aggiornano.
- Si **legge** un signal chiamandone il getter (il signal usato come funzione, es. `this.filter().from`); si **scrive** con `.set(value)` o con `.update(prev => next)` quando il nuovo valore dipende dal precedente.

```ts
const counter = signal(0);
counter.update((current) => current + 1);
```

`signal()` ritorna un `WritableSignal<T>` (getter **e** setter); deriva da `Signal<T>`, che ha solo il getter (niente `set`/`update`). Per esporre un signal in sola lettura usa `asReadonly()`:

```ts
const counter = signal(0);
const readonlyCounter = counter.asReadonly();
```

Per far girare il componente con dati statici prima di introdurre l'HTTP, `search()` riempie `flights` a mano (ogni volo riceve una copia dell'`initialAircraft` via spread, `{ ...initialAircraft }`):

```ts
protected search(): void {
  const date = new Date().toISOString();
  this.flights.set([
    {
      id: 1,
      from: this.filter().from,
      to: this.filter().to,
      date,
      delayed: false,
      delay: 0,
      aircraft: { ...initialAircraft },
      prices: [],
    },
    // ...altri voli
  ]);
}
```

Collegamenti: [[signal]] · [[06-signal-forms]] (la `form()`).

## Template & data binding
> 📖 pp.40-46

Prima del template vanno dichiarate le dipendenze nell'array `imports` del decoratore `@Component`: la direttiva `FormField` (parte di Signal Forms, lega `filterForm` ai singoli `<input>`) e le pipe `JsonPipe` e `DatePipe` (per formattare l'output).

```ts
import { JsonPipe, DatePipe } from '@angular/common';
import { FormField, form } from '@angular/forms/signals';
// ...
@Component({
  selector: 'app-flight-search',
  imports: [FormField, JsonPipe, DatePipe],
  templateUrl: './flight-search.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
```

```html
<!-- .../flight-search/flight-search.html -->
<form>
  <div>
    <label for="from">From:</label>
    <input [formField]="filterForm.from" id="from" />
  </div>
  <div>
    <label for="to">To:</label>
    <input [formField]="filterForm.to" id="to" />
  </div>
  <div>
    <button
      type="button"
      (click)="search()"
      [disabled]="!filter().from || !filter().to"
    >
      Search
    </button>
  </div>
</form>

@if (flights().length > 0) {
  <table>
    <tr>
      <th>Id</th><th>From</th><th>To</th><th>Date</th><th>Delayed</th>
    </tr>
    @for (flight of flights(); track flight.id) {
      <tr
        [class.selected]="selectedFlight() === flight"
        [style.color]="flight.delayed ? 'darkred' : 'black'"
        (click)="select(flight)"
        class="selectable"
      >
        <td>{{ flight.id }}</td>
        <td>{{ flight.from }}</td>
        <td>{{ flight.to }}</td>
        <td>{{ flight.date | date }}</td>
        <td>{{ flight.delayed }}</td>
      </tr>
    }
  </table>
}
<div>
  <pre>{{ selectedFlight() | json }}</pre>
</div>
```

Tipi di binding:
- **Event binding** `(click)="search()"` → un evento DOM o di un sotto-componente, tra parentesi tonde. Idem `(change)`, `(keydown)`, `(mouseover)`, ecc.
- **Property binding** `[disabled]="..."` → una proprietà del DOM o di un sotto-componente, tra parentesi quadre. Idem `[value]`, `[checked]`, `[src]`.
- **Interpolation** `{{ expr }}` con le **pipe** per trasformare/formattare: `| date` (accetta `Date` o stringhe ISO; parametrizzabile con `| date:'dd.MM.yyyy HH:mm'`), `| json`, `| number:'1.2'` (almeno 1 cifra intera e 2 decimali — richiede l'import di `DecimalPipe` da `@angular/common`).
- **Conditional styling**: `[class.selected]="cond"` aggiunge la classe quando la condizione è vera; `[style.color]="..."` imposta uno stile inline dinamico.

**Control flow built-in** (`@if`, `@for`, `@switch`, prefisso `@`, sintassi vicina a quella di JavaScript). Il `@for` ha anche una clausola `@empty`:

```html
@for (flight of flights(); track flight.id) {
  <!-- ... -->
} @empty {
  <tr><td colspan="5">No flights found.</td></tr>
}
```

```html
@switch (passenger().passengerStatus) {
  @case ('A') { <p>Senator</p> }
  @case ('B') { <p>Frequent Traveller</p> }
  @default { <p>Regular Passenger</p> }
}
```

> [!info] Angular 22+
> Da **Angular 21.1** un singolo blocco può seguire più `@case` consecutivi (multi-value), così più valori condividono lo stesso markup:
> ```html
> @switch (passenger().passengerStatus) {
>   @case ('SEN') @case ('A') { <p>Senator</p> }
>   @case ('FQT') @case ('B') { <p>Frequent Traveller</p> }
>   @default { <p>Regular Passenger</p> }
> }
> ```

### Exhaustive @switch
> 📖 pp.43-44 (Listing 2-12/2-14)

Un'insidia tipica dello `@switch` è dimenticare di gestire un valore aggiunto al tipo in seguito. Da **Angular 21.2** il ramo `@default` può chiedere al compilatore un **exhaustiveness check**. Restringendo `passengerStatus` da `string` a una **literal union** `'A' | 'B' | 'C'`, quando l'espressione dello `@switch` **è la union** basta la forma breve `@default never;`: TypeScript restringe a `never` una volta gestiti tutti i casi.

> [!info] Angular 22+
> ```html
> @switch (passenger().passengerStatus) {  <!-- 'A' | 'B' | 'C' -->
>   @case ('A') { <p>Senator</p> }
>   @case ('B') { <p>Frequent Traveller</p> }
>   @case ('C') { <p>Regular Passenger</p> }
>   @default never;
> }
> ```
> Se aggiungi `'D'` alla union senza il relativo `@case`, il template **non compila**.
> Quando invece fai switch su una **proprietà** di una discriminated union (il discriminatore) e non sulla union intera, TypeScript sa restringere la proprietà ma non sa dire se l'intera union è coperta: serve la forma `never(<expression>)` (**Angular 22**), che indica esplicitamente al compilatore quale espressione controllare per la copertura completa.
> ```html
> @switch (loyalty().kind) {
>   @case ('senator')  { <p>Senator (Lounge access: {{ loyalty().loungeAccess }})</p> }
>   @case ('frequent') { <p>Frequent Traveller ({{ loyalty().bonusMiles }} Miles)</p> }
>   @case ('regular')  { <p>Regular Passenger</p> }
>   @default never(loyalty());
> }
> ```
> Aggiungere una variante come `{ kind: 'staff' }` alla union `Loyalty` senza il corrispondente `@case` rompe la build del template.

> [!warning]
> Nel `@for` il `track` è **obbligatorio**: punta a un identificatore univoco (es. `track flight.id`; in mancanza, `track $index` o l'oggetto stesso `track flight`). Permette ad Angular di **spostare** i nodi DOM esistenti quando cambia l'ordine, invece di ri-renderizzare tutta la lista.

> [!warning]
> Usa `<button type="button">`: il default è `submit`, che farebbe il submit del form e un reload completo della pagina. La direttiva `formRoot` del [[06-signal-forms|cap.6]] disabilita questo comportamento di default del browser.

## Chiamare il componente
> 📖 pp.46-47

Si importa `FlightSearch` nell'array `imports` di `App` (`app.ts`) e lo si usa in `app.html`:

```ts
// src/app/app.ts
import { FlightSearch }
  from './domains/ticketing/feature-booking/flight-search/flight-search';

@Component({
  selector: 'app-root',
  imports: [Navbar, Sidebar, FlightSearch],
  templateUrl: './app.html',
  styleUrl: './app.css',
})
export class App {
  protected readonly title = signal('flights42');
}
```

```html
<!-- src/app/app.html -->
<app-flight-search></app-flight-search>
<!-- oppure il tag self-closing: <app-flight-search /> -->
```

Il prefisso del selector (`app-`, inserito dalla CLI per evitare collisioni con librerie e tag HTML nativi) si cambia in `angular.json` (chiave `prefix`); con il linting attivo va allineato anche in `eslint.config.js`.

## Debugging
> 📖 pp.47-50

- **Console del browser**: gli errori a runtime compaiono nei dev tools; Angular stampa **link cliccabili** alle righe dei file HTML/TS coinvolti.
- **Source Maps**: `ng serve` le genera di default → puoi usare il debugger JavaScript del browser (in Chrome: tab **Sources**; `Ctrl+Shift+P` / `Cmd+Shift+P` per aprire un file e mettere un breakpoint sul numero di riga).
- **VS Code**: breakpoint direttamente nel `.ts` e avvio con `Run | Start Debugging` (F5); funziona senza config aggiuntiva perché la CLI genera già `.vscode/launch.json` allo scaffolding del progetto.

## Data access: HttpClient
> 📖 pp.50-54

Per la comunicazione HTTP diretta si inietta il service `HttpClient` con [[inject]] e si chiamano i suoi metodi dentro `search()`.

```ts
import { HttpClient } from '@angular/common/http';
import { Component, ChangeDetectionStrategy, signal, inject } from '@angular/core';

export class FlightSearch {
  private readonly http = inject(HttpClient);
  // ...
  protected search(): void {
    const url = 'https://demo.angulararchitects.io/api/flight';
    const filter = this.filter();
    const params = {
      from: filter.from,
      to: filter.to,
    };
    this.http.get<Flight[]>(url, { params }).subscribe({
      next: (flights) => {
        this.flights.set(flights);
      },
      error: (err) => {
        console.error('Error', err);
      },
    });
  }
}
```

- `get<T>` emette una GET e ritorna un **Observable** (il recupero è asincrono): ci si iscrive con `subscribe({ next, error })`. `T` è il tipo della risposta (qui `Flight[]`): l'`HttpClient` converte il JSON ricevuto in oggetto.
- Altri metodi con firma analoga, uno per verbo HTTP: `get<T>(url, options)`, `post<T>(url, body, options)` (aggiunge una risorsa o avvia un'elaborazione), `put<T>` (aggiunge/aggiorna), `patch<T>` (aggiorna solo le proprietà cambiate), `delete<T>(url, options)`, `request<T>(method, url, options)` (generico, il verbo è un parametro). I metodi che inviano dati hanno un parametro `body`; l'`options` configura `params`, `headers`, ecc.
- Attenzione: non ogni Web API supporta tutti i verbi e la semantica implementata può discostarsi da quella HTTP (es. una POST che in realtà aggiorna) → fai riferimento alla doc dell'API.

> [!warning]
> Il termine *resource* qui è quello di HTTP (l'oggetto da recuperare o inviare) e **non** va confuso con il concetto Angular di `resource`/`httpResource` della sezione successiva.

Collegamenti: [[inject]].

## Data access: httpResource (signal-based)
> 📖 pp.54-59

`httpResource` è il modo **signal-based** di caricare dati via HTTP: prende signal come input, e ne espone stato e risultato come signal.

```ts
import { httpResource } from '@angular/common/http';

export class FlightSearch {
  // ...
  protected readonly flightsResource = httpResource<Flight[]>(
    () => ({
      url: 'https://demo.angulararchitects.io/api/flight',
      params: {
        from: this.filter().from,
        to: this.filter().to,
      },
    }),
    { defaultValue: [] },
  );

  // risultato e stato della resource:
  protected readonly flights = this.flightsResource.value;
  protected readonly error = this.flightsResource.error;
  protected readonly isLoading = this.flightsResource.isLoading;

  protected search(): void {
    this.flightsResource.reload();
  }
}
```

- La lambda ritorna l'oggetto che descrive la richiesta ed è **reattiva**: quando cambia un qualsiasi signal letto al suo interno, la resource si ri-triggera e ricarica. Parte **subito** alla creazione, per il primo fetch.
- `Flight[]` è il tipo atteso della risposta: la resource fa il parse del JSON assumendo questo tipo.
- `reload()` ri-triggera manualmente la richiesta senza cambiare i parametri.
- Per **impedire** il fetch iniziale o disattivarla a runtime, ritorna `undefined` dalla lambda (conviene passare a una lambda con `return` esplicito):

```ts
protected readonly flightsResource = httpResource<Flight[]>(
  () => {
    const filter = this.filter();
    if (!filter.from || !filter.to) {
      return undefined;
    }
    return {
      url: 'https://demo.angulararchitects.io/api/flight',
      params: { from: filter.from, to: filter.to },
    };
  },
  { defaultValue: [] },
);
```

- Di default fa una **GET**; puoi specificare un altro metodo e un `body`, ma è pensata per il **fetch (read)**: per save/delete usa `HttpClient`.
- `{ defaultValue: [] }` (secondo argomento) dà un valore iniziale prima che la richiesta completi → niente `undefined` nel template.
- Stato esposto via signal: `value` (i dati), `error` (info sull'errore), `isLoading` (richiesta in corso).

> [!info] Angular 22+
> Il `value` di una resource è **scrivibile**: puoi modificare i dati caricati localmente (utile per editarli con un form in two-way binding sul `value`). Resta però una **copia di lavoro locale** — per persistere sul server serve comunque `HttpClient`.

```html
<!-- .../flight-search/flight-search.html -->
@if (!error() && flights().length > 0) {
  <table> <!-- ...righe come prima... --> </table>
}
@if (error()) {
  <div>Error: {{ error() | json }}</div>
}
@if (isLoading()) {
  <div>Loading...</div>
}
```

> [!warning]
> In stato di errore **non leggere `value()`**: lancia un'eccezione. Controlla prima `error()` e leggi `value` solo se è `undefined` — nel template `@if (!error() && flights().length > 0)`.

In alternativa a `error()` puoi guardare il signal `status()`, che vale uno tra: `idle` (niente caricato) · `loading` · `reloading` · `error` · `resolved` (caricamento riuscito) · `local` (valore cambiato localmente via `set`/`update`).

> [!tip]
> `httpResource` gestisce da sola le **race condition**: se più richieste partono in rapida successione tiene **solo l'ultima** e scarta le risposte precedenti che arrivano dopo (come `switchMap` in RxJS — la richiesta in corso viene abortita o ignorata). `reload()` invece, se chiamato mentre una richiesta è già in corso, **viene ignorato** (come `exhaustMap`). Queste semantiche sono ideali per il fetch, ma non per update/delete → un altro motivo per cui `HttpClient` resta rilevante.

Collegamenti: [[resource]] · approfondimento in [[03-reactive-design-with-signals]].

## Sotto-componenti: input, output, model
> 📖 pp.59-71

Crescendo l'app conviene spezzare i componenti complessi in pezzi piccoli e riutilizzabili. Si estrae un `FlightCard` che mostra il singolo volo e può essere selezionato, sostituendo le righe della tabella. Essendo un componente general-purpose non legato a una specifica feature, sta nella cartella `ui` del dominio:

```bash
ng g c domains/ticketing/ui/flight-card
```

Prima di estrarlo si sostituisce `selectedFlight` con un `basket`: un `Record` che mappa id-volo → booleano (i voli 3 e 5 sono nel carrello fin dall'inizio, a scopo dimostrativo). L'aggiornamento è **immutabile** (nuovo riferimento d'oggetto → Angular rileva il cambio):

```ts
protected readonly basket = signal<Record<number, boolean>>({
  3: true,
  5: true,
});

protected updateBasket(flightId: number, selected: boolean): void {
  this.basket.update((basket) => ({
    ...basket,
    [flightId]: selected,
  }));
}
```

Lo spread crea un **nuovo** oggetto con tutte le voci correnti più quella aggiornata: il nuovo riferimento aiuta Angular a rilevare la modifica (gli *immutables* sono approfonditi nel [[03-reactive-design-with-signals|cap.3]]).

L'interfaccia pubblica vista dal padre: due **property binding** in ingresso (volo e stato di selezione) e un **event binding** in uscita; il `$event` dell'evento porta il valore emesso.

```html
<app-flight-card
  [item]="flight"
  [selected]="basket()[flight.id]"
  (selectedChange)="updateBasket(flight.id, $event)"
/>
```

Dentro `FlightCard` ogni proprietà è un [[signal-input|input()]] e ogni evento un [[signal-output|output()]]:

```ts
// src/app/domains/ticketing/ui/flight-card/flight-card.ts
import { DatePipe } from '@angular/common';
import { ChangeDetectionStrategy, Component, input, output } from '@angular/core';
import { Flight } from '../../data/flight';

@Component({
  selector: 'app-flight-card',
  imports: [DatePipe],
  templateUrl: './flight-card.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class FlightCard {
  readonly item = input.required<Flight>();    // obbligatorio, niente default
  readonly selected = input(false);            // opzionale, default false
  readonly selectedChange = output<boolean>(); // OutputEmitterRef<boolean>

  protected select() {
    this.selectedChange.emit(true);
  }
  protected deselect() {
    this.selectedChange.emit(false);
  }
}
```

- Gli input sono **signal di sola lettura**: `InputSignal<T>` (o `InputSignal<T | undefined>` se opzionale). `input.required<Flight>()` non ha default e, se il padre lo dimentica, Angular lancia un **errore a compile-time**; gli input obbligatori non possono avere default.
- Essendo read-only, per notificare il padre si emette un evento: `output<boolean>()` crea un `OutputEmitterRef<boolean>`, su cui si chiama `.emit(value)`. Nel template li si legge come signal: `item()` (o `@let v = item()` per una variabile locale più leggibile).
- Nell'event binding il valore emesso arriva nella variabile speciale **`$event`** del template padre.

**ModelSignal** — input **scrivibile** (two-way). `model()` crea automaticamente la proprietà **e** l'evento `<nome>Change`:

```ts
import { ChangeDetectionStrategy, Component, input, model } from '@angular/core';

export class FlightCard {
  readonly item = input.required<Flight>();
  readonly selected = model(false);   // crea selected + selectedChange

  protected select() {
    this.selected.set(true);
  }
  protected deselect() {
    this.selected.set(false);
  }
}
```

Dal padre il binding **non cambia**: si lega ancora a `[selected]` e `(selectedChange)`. La differenza è di intento: un `model()` serve quando il figlio deve **riscrivere subito** la modifica al padre; con `input` + `output` il figlio controlla con precisione *quando* notificare.

**Two-way binding** "banana in a box" `[(prop)]` — zucchero per un property + un event binding con naming `prop`/`propChange`. Esempio: uno `SimpleDelayStepper` con `value = model(0)` che incrementa/decrementa di 15 minuti.

```ts
// src/app/domains/shared/ui-common/simple-delay-stepper/simple-delay-stepper.ts
import { Component, ChangeDetectionStrategy, model } from '@angular/core';

@Component({
  selector: 'app-simple-delay-stepper',
  imports: [],
  templateUrl: './simple-delay-stepper.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class SimpleDelayStepper {
  readonly value = model(0);

  protected increase(): void {
    this.value.update((v) => v + 15);
  }
  protected decrease(): void {
    this.value.update((v) => Math.max(v - 15, 0));
  }
}
```

```html
<app-simple-delay-stepper [(value)]="maxDelay" />
<!-- equivale a: -->
<app-simple-delay-stepper [value]="maxDelay()" (valueChange)="maxDelay.set($event)" />
```

Nel `[(value)]="maxDelay"` si passa il signal **senza** chiamarne il getter: Angular invoca getter e setter da solo. Funziona sia con `model()` sia con `input.required` + `output`, purché l'evento si chiami `valueChange` ed emetta direttamente il nuovo valore (così compare in `$event`):

```ts
export class SimpleDelayStepper {
  readonly value = input.required<number>();
  readonly valueChange = output<number>();

  protected increase(): void {
    this.valueChange.emit(this.value() + 15);
  }
  protected decrease(): void {
    this.valueChange.emit(Math.max(this.value() - 15, 0));
  }
}
```

**Content projection** con `<ng-content>` — il padre inietta markup (HTML o componenti) nel figlio senza toccarne l'implementazione. Un `<ng-content>` può avere un **contenuto di default** (mostrato se il padre non proietta nulla) e si possono avere più slot, ciascuno con `select="<selettore-CSS>"`:

```html
<!-- luggage-card.html (tre slot, simile a flight-card) -->
<div class="card-body">
  <ng-content select=".detail" />
  <p>
    <!-- ...bottoni Select/Remove... -->
    <ng-content select=".button" />
  </p>
  <ng-content select=".footer">All rights reserved.</ng-content>  <!-- default -->
</div>
```

```html
<!-- il chiamante riempie gli slot per nome di classe -->
<app-luggage-card [item]="item" [selected]="selected()[item.id]"
                  (selectedChange)="updateSelected(item.id, $event)">
  <p class="detail">Full Id: LUGGAGE-0815-4711-{{ item.id }}</p>
  <span class="button"><button>Add Insurance</button></span>
  <div class="footer">Your luggage is in the best hands!</div>
</app-luggage-card>
```

Collegamenti: [[signal-input]] · [[signal-output]] · [[model-signal]] · [[two-way-binding]] · [[content-projection]] · approfondimento comunicazione padre-figlio in [[10-signal-queries-component-communication]].

## 🔁 Ripasso lampo

**1.** Perché `signal<Flight[]>([])` ha il tipo esplicito mentre `signal({ from, to })` no?
> [!success]- Risposta
> Il tipo di un signal si inferisce dal valore iniziale. `{ from: 'Hamburg', to: 'Graz' }` porta abbastanza informazione per inferire la forma dell'oggetto. Un **array vuoto** `[]` invece non dice nulla sul tipo degli elementi, quindi va dato esplicitamente con il type parameter: `signal<Flight[]>([])`.

**2.** Quando usare `HttpClient` e quando `httpResource`? Cosa garantisce `httpResource` sulle race condition?
> [!success]- Risposta
> `HttpClient` per la comunicazione diretta e per **tutti** i verbi (save/delete inclusi): ritorna Observable a cui ti iscrivi. `httpResource` per il **fetch (GET) reattivo** signal-based: la lambda reattiva ricarica al cambio dei signal letti, ed espone `value`/`error`/`isLoading`/`status`. Sulle race condition tiene **solo l'ultima** richiesta scartando le risposte precedenti (come `switchMap`); `reload()` invece ignora le chiamate sovrapposte (come `exhaustMap`).

**3.** Perché il `track` nel `@for` è obbligatorio e cosa puoi passargli?
> [!success]- Risposta
> Identifica univocamente ogni elemento così Angular, al cambio d'ordine, **sposta** i nodi DOM esistenti invece di ri-renderizzare tutta la lista. Idealmente una proprietà univoca come `track flight.id`; in mancanza, `track $index` o l'oggetto stesso `track flight`.

**4.** Differenza tra `input()`, `output()` e `model()`? Cos'è `$event` in un event binding?
> [!success]- Risposta
> `input()` crea un **InputSignal** di sola lettura (dato dal padre verso il figlio). `output()` crea un **OutputEmitterRef** su cui chiamare `.emit()` (evento dal figlio verso il padre). `model()` crea un input **scrivibile** (ModelSignal): genera proprietà **e** evento `<nome>Change` per il two-way binding. `$event` è la variabile speciale del template che, in un event binding, contiene il valore emesso (es. il `boolean` di `selectedChange.emit`).

**5.** Cosa fa `[(value)]="x"` sotto il cofano? Quali condizioni di naming richiede?
> [!success]- Risposta
> È zucchero sintattico ("banana in a box") per un property binding `[value]="x()"` **più** un event binding `(valueChange)="x.set($event)"`. Richiede la convenzione di naming `prop` + `propChange`, con l'evento che emette direttamente il nuovo valore. `model()` la fornisce out-of-the-box, ma funziona anche con `input.required` + `output`. Nota: nel `[(value)]="x"` si passa il signal **senza** chiamarne il getter.

**6.** Perché aggiorni il basket con lo spread invece di mutarlo?
> [!success]- Risposta
> `this.basket.update((b) => ({ ...b, [id]: selected }))` crea un **nuovo** oggetto (nuovo riferimento). Mutare l'oggetto esistente non cambierebbe il riferimento e Angular potrebbe non rilevare la modifica; un nuovo riferimento (immutabilità) fa scattare in modo affidabile la change detection.

**In sintesi:**
- Lo stato dei componenti si modella con i **signal** (`signal`/`computed`/`set`/`update`); il template usa interpolation, property/event/two-way binding, il control flow `@if`/`@for`/`@switch` (con `@empty`, multi-value e `@default never`) e le pipe.
- Due vie ai dati del backend: `HttpClient` (Observable, tutti i verbi) e `httpResource` (signal-based, fetch reattivo con `value`/`error`/`isLoading`/`status` e gestione race condition); allineata a `OnPush`.
- Comunicazione padre-figlio con `input` (giù) / `output` (su) / `model` (scrivibile, two-way), tutti signal-based, più la **content projection** con `<ng-content>` (default content e slot multipli via `select`).
- Il two-way binding `[(prop)]` è solo uno shorthand per `[prop]` + `(propChange)`.
