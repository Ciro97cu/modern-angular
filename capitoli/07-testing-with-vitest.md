---
capitolo: 7
titolo: "Modern Testing with Vitest"
pagine: "204-230"
tags: [tipo/capitolo, testing, http, angular-22]
---
# 07 · Modern Testing with Vitest
> 📖 cap.7 · pp.204-230 — *Modern Angular* v2.0.0

Con sempre più logica spostata nel frontend, il test manuale non basta a mantenere stabile un'app nel tempo. La Angular CLI integra **Vitest** come test runner di default: esecuzione veloce, ottima developer experience e supporto a vari runtime (Node.js e diversi browser). Sopra a Vitest, Angular mette a disposizione costrutti propri (`TestBed`, `HttpTestingController`) per montare i componenti e sostituire le dipendenze con dei mock. Il capitolo costruisce i test sulla feature `flight-search` già vista nel [[02-signal-based-components|cap.2]].

## Structure of a Vitest Test
> 📖 pp.204-206

Vitest fornisce due funzioni base: `describe` definisce una test **suite** (raggruppa casi correlati) e `it` definisce un test **case** (verifica una porzione di funzionalità). `describe`, `it`, `beforeEach`, `expect` e le altre sono **globali**: Vitest le rende disponibili durante l'esecuzione, quindi non vanno importate. I file di test hanno estensione **`.spec.ts`** — che li qualifica anche come *specifiche* del comportamento desiderato — e si tengono accanto al codice testato, così restano sempre aggiornati.

```ts
// src/app/demo.spec.ts

// Object under test
const add = (a: number, b: number) => a + b;

// Test suite
describe('Add', () => {
  // gira prima di OGNI test case
  beforeEach(() => {
    console.log('Preparation tasks ...');
  });

  // Test case
  it('correctly adds 1 and 2', () => {
    // Arrange
    const a = 1;
    const b = 2;
    // Act
    const c = add(a, b);
    // Assert
    expect(c).toBe(3);
  });
});
```

- Pattern **AAA**: *Arrange* (prepara l'esecuzione) → *Act* (esegue l'azione da testare) → *Assert* (verifica che il risultato atteso sia stato ottenuto).
- Hook del ciclo di vita: `beforeEach` / `afterEach` girano prima/dopo **ogni** caso; `beforeAll` / `afterAll` girano **una volta** per suite (prima del primo caso / dopo l'ultimo).
- Nella fase di Assert si usa `expect(...)`, concatenabile con i cosiddetti **matcher** (scoprili con l'autocompletamento dell'IDE):

```ts
expect(c).not.toBe(4);
expect(c).not.toBeNull();
expect(c).toBeGreaterThan(0);
const str = 'Result: ' + c;
expect(str).toContain('Result');
expect(str).toMatch(/Result/);
```

> [!tip]
> In ottica **BDD** (Behavior-Driven Development), il nome della suite e quello del caso, letti insieme, devono descrivere il comportamento atteso: `Add` + `correctly adds 1 and 2` → *"Add correctly adds 1 and 2"*. Le `describe` si annidano liberamente chiamando un'altra `describe` dentro un blocco `describe`.

## Skipping Tests
> 📖 p.206

Per saltare un caso o un'intera suite — tipico un `app.spec.ts` generato e mai mantenuto, che ora fallisce — sostituisci `describe` con **`describe.skip`** (e `it` con **`it.skip`** per i singoli casi).

```ts
// src/app/app.spec.ts
describe.skip('App', () => {
  // [...]
});
```

## Turning on Browser Mode
> 📖 pp.206-207

Di default Vitest gira in ambiente **Node.js**, ma per i test Angular si raccomanda la **browser mode**: rispecchia più fedelmente il comportamento reale dell'app e rende più realistici i component test (imitano l'interazione dell'utente). Per accedere al browser, Vitest ha bisogno di un **provider**: gli autori consigliano **Playwright** (più moderno di webdriver.io, più semplice da configurare e molto performante). Vitest lo usa "sotto il cofano": i test **non** diventano test Playwright.

```bash
ng add @vitest/browser-playwright
npx playwright install
```

Si attiva il browser nel nodo `projects/<project-name>/architect/test` di `angular.json`:

```json
"test": {
  "builder": "@angular/build:unit-test",
  "options": {
    "browsers": ["Chromium"]
  },
  "configurations": {
    "ci": {
      "watch": false,
      "browsers": ["ChromiumHeadless"]
    }
  }
}
```

- Browser supportati dal provider Playwright: `Chromium`, `Firefox`, `Webkit` (Safari), più le varianti **headless** (browser che gira senza finestra grafica, tipico in CI dove non c'è uno schermo): `ChromiumHeadless`, `FirefoxHeadless`, `WebkitHeadless`.
- Di default la CLI parte in **watch mode** (ri-esegue i test a ogni modifica): feedback rapido in sviluppo. In CI conviene eseguirli una sola volta → `watch: false` nella configurazione `ci`.

## Running Tests
> 📖 p.207

```bash
ng test                      # esegue i test (watch mode di default)
ng test --configuration=ci   # usa la configurazione "ci"
```

Nella UI di Vitest: a sinistra l'elenco dei test eseguiti (spunta verde = passato), a destra le info per ogni run (stack trace in caso di errore), al centro Vitest **monta** i componenti (di solito solo un lampo, ma in debug riflette lo stato corrente). Vari IDE supportano il debug dei test; come ultima spiaggia si possono sempre debuggare nei dev tools del browser.

## Angular and Vitest: Preparing the TestBed
> 📖 pp.208-210

Il **`TestBed`** è il costrutto centrale che Angular offre per i test: come un banco di prova, ci si "attacca" l'oggetto sotto test e gli si forniscono gli input. Permette di definire l'oggetto sotto test e le sue dipendenze, e può **istanziare** i componenti: inietta i servizi dipendenti e dà vita al template, abilitando l'interazione a livello DOM.

```ts
// .../ticketing/feature-booking/flight-search/flight-search.spec.ts
import {
  HttpTestingController,
  provideHttpClientTesting,
} from '@angular/common/http/testing';
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { provideRouter } from '@angular/router';
import { FlightSearch } from './flight-search';

describe('flight-search', () => {
  let component: FlightSearch;
  let fixture: ComponentFixture<FlightSearch>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [FlightSearch],          // oggetto sotto test
      providers: [
        provideRouter([]),              // serve perché il componente usa RouterLink
      ],
    }).compileComponents();

    fixture = TestBed.createComponent(FlightSearch);
    component = fixture.componentInstance;
  });

  it('can be created', () => {
    expect(component).not.toBeUndefined();
  });
});
```

`createComponent` restituisce una **`ComponentFixture<T>`** che rappresenta i vari aspetti del componente. Tra i suoi membri:
- `componentInstance` — l'istanza della classe componente (qui di tipo `FlightSearch`).
- `nativeElement` — il nodo DOM del componente.
- `debugElement` — accesso ad aspetti aggiuntivi (es. l'injector a livello componente, query su nodi DOM o servizi subordinati).

Collegamenti: [[inject]] · [[providers]] · [[04-router-navigation-lazy-loading]] (il `provideRouter`).

## A First Component Test & Locators
> 📖 pp.210-212

Il primo test tratta il componente — più o meno — come una **black box** (scatola nera: non si guarda dentro al codice, si interagisce solo da fuori): interagisce via DOM, come farebbe l'utente. In browser mode Vitest fornisce l'oggetto **`page`**; usando **`expect.element`** (invece del semplice `expect`) si ottengono assert "browser-aware" (es. `toBeDisabled`).

```ts
import { page } from 'vitest/browser';

it('disables search button when from and to are not given', async () => {
  await page.getByLabelText('From').fill('');
  await page.getByLabelText('To').fill('');
  const button = page.getByRole('button', { name: 'Search' });
  await expect.element(button).toBeDisabled();
});
```

I **locator** (`getByLabelText`, `getByRole`, …) trovano gli elementi DOM tramite il loro **nome accessibile**, indirizzando ruoli e attributi ARIA per incoraggiare app accessibili. Il nome accessibile deriva da una `<label for>`, dall'attributo `aria-label`, oppure dal testo del bottone.

```html
<div>
  <label for="from">From:</label>
  <input [formField]="filterForm.from" id="from" />
</div>

<!-- in alternativa, senza label: -->
<input [formField]="filterForm.from" id="from" aria-label="from" />

<button (click)="search()" [disabled]="!from() || !to()">Search</button>
```

- Di default i locator sono **case-insensitive** e fanno match su **sottostringa** (un bottone `Search Flights` verrebbe comunque trovato da `'Search'`). Con `{ exact: true }` accetti solo il match esatto, oppure usi una **regex**.
- Un locator può restituire **più elementi** (`toHaveLength(n)`).

```ts
const button = page.getByRole('button', { name: 'Search', exact: true });
await page.getByLabelText(/From|Airport of Departure/i).fill('Paris');

const headings = page.getByRole('heading', { name: 'Paris - London' });
await expect.element(headings).toHaveLength(3);
```

I locator **descrivono solo come trovare** gli elementi: il recupero effettivo avviene solo all'uso (`fill`, `click`, dentro `expect.element`), con **retry** fino a un timeout — tra un tentativo e l'altro si attende un intervallo per far completare i task asincroni. Si configurano con un secondo parametro opzionale (default `{ interval: 50, timeout: 100 }`):

```ts
await expect.element(button, { interval: 50, timeout: 100 }).toBeDisabled();
```

Per accedere subito al nodo DOM dietro un locator: `.element()` (o `.elements()` se rappresenta più nodi).

> [!warning]
> I locator non offrono accesso per `id` o `name`: è **intenzionale**, per spingere verso i ruoli/attributi ARIA. Come ultima spiaggia `page.getByTestId(...)` legge un attributo `data-testid`, ma mina l'accessibilità → usalo con parsimonia.

## Locating Elements via DebugElement
> 📖 p.213

Anche il `debugElement` della fixture permette query sul DOM, ma tramite **selettori CSS** (non ARIA). È più di **basso livello** (lavori più vicino al DOM, scrivendo a mano cose che `page` farebbe per te) e quindi più verboso: niente retry integrato, e devi **dispatchare a mano** l'evento `input` (cioè scatenarlo tu via codice) per notificare ad Angular il cambio di valore.

```ts
import { By } from '@angular/platform-browser';

const to = fixture.debugElement.query(By.css('input[name=to]')).nativeElement;
to.value = 'London';
to.dispatchEvent(new Event('input'));   // necessario per notificare Angular
```

> [!tip]
> Preferisci l'oggetto **`page`** di Vitest (browser mode) al `debugElement`: meno verboso, orientato ad ARIA e con retry integrato.

## Defining Default Timeouts
> 📖 pp.213-214

Oltre al timeout per-chiamata, puoi definirne uno di default per-suite o per-caso passando un oggetto opzioni (`TestOptions`) a `describe` / `it`:

```ts
const suiteOptions: TestOptions = { timeout: 200 };
const caseOptions: TestOptions = { timeout: 300 };

describe('FlightEdit (router)', suiteOptions, () => {
  it('navigates to flight details on click', caseOptions, async () => {
    // [...]
  });
});
```

Per sovrascrivere il timeout di default di **tutte** le suite e i casi, crea un file di setup e imposta `testTimeout`:

```ts
// src/vitest-setup.ts
import { vi } from 'vitest';
vi.setConfig({
  testTimeout: 3_000,
});
```

Poi registra il file per il builder `unit-test` in `angular.json`:

```json
"test": {
  "builder": "@angular/build:unit-test",
  "options": {
    "browsers": ["Chromium"],
    "setupFiles": ["src/vitest-setup.ts"]
  }
}
```

## Mocking Services
> 📖 pp.214-216

I test isolano l'oggetto sotto test dalle sue dipendenze: così gli errori sono attribuibili con chiarezza e i test restano veloci e affidabili, senza dipendere da sistemi esterni. Si sostituiscono con **mock** (implementazioni semplificate); le **spy**, invece, monitorano le chiamate (con quali parametri e quante volte). Grazie alla [[glossario#dependency-injection-di|DI]] di Angular (il meccanismo con cui Angular fornisce le dipendenze a un componente) basta dire al `TestBed` di fornire un mock al posto del servizio reale — qui il `ConfigService`, che espone dati di configurazione come la base URL del backend.

```ts
import { ConfigService } from '../domains/shared/util-common/config-service';

await TestBed.configureTestingModule({
  imports: [FlightSearch],
  providers: [
    provideRouter([]),
    // mock del ConfigService
    { provide: ConfigService, useValue: { baseUrl: '', model: '' } },
  ],
}).compileComponents();
```

`useValue` permette di fornire un oggetto semplice invece di un servizio class-based completo, ma ha due svantaggi: il valore **non è tipizzato** (può divergere dalla forma reale del `ConfigService`) e va duplicato fra i test. Conviene estrarre questi mock provider in una funzione riusabile:

```ts
// src/app/testing/provide-test-config.ts
import { Provider } from '@angular/core';
import {
  Config,
  ConfigService,
} from '../domains/shared/util-common/config-service';

export function provideTestConfig(): Provider {
  return {
    provide: ConfigService,
    useValue: {
      baseUrl: '',
      model: '',
    },
  };
}
```

```ts
providers: [
  provideRouter([]),
  provideTestConfig(),   // mock del ConfigService
],
```

Collegamenti: [[providers]] · [[05-state-management-services-signals]] (servizi e DI).

## Mocking Services on the Component Level
> 📖 p.216

Fornire mock via `TestBed` funziona per i servizi registrati al **root level**. Se invece un servizio è fornito **a livello componente** (array `providers` del `@Component`), la configurazione del `TestBed` non basta: si **sovrascrivono i metadati** del componente con `overrideComponent`.

```ts
TestBed.overrideComponent(FlightSearch, {
  add: {
    providers: [{
      provide: LanguageService,
      useClass: DefaultLanguageService,
    }],
  },
});
```

Questo ha effetto sul componente, sui suoi figli e sugli altri servizi allo stesso livello.

## Mocking Child Components & Shallow Testing
> 📖 pp.216-217

Per evitare test cross-componente (`FlightSearch` testato insieme a `FlightCard`) e concentrarsi su un solo componente — **shallow testing** — si mocka il figlio con uno **stub** che ha **stesso selector, stessi input e stessi output** dell'originale. Poi si scambiano gli `imports` nei metadati del componente sotto test:

```ts
TestBed.overrideComponent(FlightSearch, {
  remove: { imports: [FlightCard] },
  add: { imports: [DummyFlightCard] },
});
```

## Mocking HTTP Calls
> 📖 pp.217-221

Per essere indipendenti dal backend si mockano le chiamate HTTP. Si potrebbe mockare il `FlightClient`, ma c'è una via più semplice: Angular fornisce implementazioni mock per i servizi interni dell'`HttpClient` tramite **`provideHttpClientTesting()`**, che intercettano le richieste e restituiscono risposte fake senza vere richieste di rete. Vale anche per **`httpResource`**, costruito sopra l'`HttpClient`.

```ts
// .../feature-booking/flight-search/flight-search.spec.ts
import {
  HttpTestingController,
  provideHttpClientTesting,
} from '@angular/common/http/testing';
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { provideRouter } from '@angular/router';
import { page } from 'vitest/browser';
import { provideTestConfig } from '../../../../testing/provide-test-config';
import { FlightSearch } from './flight-search';
import { FlightStore } from './flight-store';

describe('flight-search', () => {
  let component: FlightSearch;
  let fixture: ComponentFixture<FlightSearch>;
  let ctrl: HttpTestingController;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [FlightSearch],
      providers: [
        provideRouter([]),
        provideTestConfig(),           // mock del ConfigService
        provideHttpClientTesting(),    // mock dei provider HTTP
      ],
    }).compileComponents();

    fixture = TestBed.createComponent(FlightSearch);
    component = fixture.componentInstance;
    ctrl = TestBed.inject(HttpTestingController);

    // attende il caricamento iniziale (httpResource allo startup)
    const request = await vi.waitFor(
      () => ctrl.expectOne('/flight?from=Graz&to=Hamburg'),
    );
    request.flush([]);
  });

  // [...]
});
```

Una volta inclusi i mock provider, si può iniettare l'**`HttpTestingController`**: informa di ogni richiesta HTTP scatenata e ne accetta la risposta fake. **Ogni chiamata resta in pausa** finché il test non risponde con `flush(...)`. L'app carica già alcuni voli allo startup via `httpResource`: bisogna gestire quella richiesta, altrimenti la resource resterebbe appesa all'infinito e il componente non uscirebbe mai dallo stato di loading.

> [!warning]
> L'`httpResource` **non** esegue la chiamata HTTP subito: internamente usa un [[effect]] che osserva il request signal, e per definizione gli effect girano in un **microtask successivo** (un microtask è un compito che il browser esegue subito dopo il codice in corso, prima del prossimo "giro" dell'event loop: qui serve a evitare stati intermedi inutili). Quindi prima di `expectOne` bisogna **attendere** che l'effect parta con `vi.waitFor(...)` (riprova la funzione finché ha successo o scatta il timeout). Senza, la richiesta non esiste ancora e il componente resta in loading.

```ts
const request = await vi.waitFor(
  () => ctrl.expectOne('/flight?from=Graz&to=Hamburg'),
  { interval: 50, timeout: 1000 },   // timeout/interval configurabili
);
```

Qui basta in genere un solo retry (si attende solo il microtask pendente che scatena la resource), quindi spesso è sufficiente `{ interval: 0 }` per attendere il tick successivo dell'event loop. `vi` è globale oppure va importato: `import { vi } from 'vitest';`.

Test case completo (riempie i campi, cerca, risponde con 3 voli):

```ts
it('searches for flights when from and to are given', async () => {
  await page.getByLabelText('From').fill('Paris');
  await page.getByLabelText('To').fill('London');
  const button = page.getByRole('button', { name: 'Search' });
  await button.click();

  const request = await vi.waitFor(() =>
    ctrl.expectOne('/flight?from=Paris&to=London'),
  );
  request.flush([
    createTestFlight(1),
    createTestFlight(2),
    createTestFlight(3),
  ]);

  const headings = page.getByRole('heading', { name: 'Paris - London' });
  await expect.element(headings).toHaveLength(3);
});
```

```ts
// src/app/testing/create-test-flight.ts
export function createTestFlight(id: number, from = 'Paris', to = 'London') {
  const date = new Date().toISOString();
  const delayed = false;
  const flight = { id, from, to, date, delayed };
  return flight;
}
```

> [!tip]
> Aggiungi un `afterEach(() => ctrl.verify())`: `verify()` lancia un'eccezione se restano richieste **senza risposta**. Mockando il risultato non verifichi che il backend funzioni, ma che il componente faccia le **richieste attese** e gestisca correttamente le risposte.

Collegamenti: [[resource|httpResource]] · [[02-signal-based-components]] (la feature `flight-search`).

## Gray-Box Testing with Spies
> 📖 pp.221-222

Idealmente il test si concentra sul **comportamento** dell'oggetto sotto test, senza assunzioni sul suo funzionamento interno. A volte però conviene verificare che certi metodi **interni** vengano chiamati correttamente → test **gray-box** (scatola grigia: una via di mezzo, dove si conoscono alcuni dettagli interni e non solo l'esterno come nella black box). Le **spy** di Vitest (`vi.spyOn`) avvolgono una funzione/metodo (gli stanno "intorno" senza cambiarne il comportamento) e ricordano con quali parametri è stata chiamata e quante volte.

```ts
// .../feature-booking/flight-search/flight-search.spec.ts
import { page } from 'vitest/browser';
import { createTestFlight } from '../../../../testing/create-test-flight';
import { FlightStore } from './flight-store';

it('searches for flights when from and to are given', async () => {
  const flightStore = TestBed.inject(FlightStore);
  vi.spyOn(flightStore, 'updateFilter');

  await page.getByLabelText('From').fill('Paris');
  await page.getByLabelText('To').fill('London');
  const button = page.getByRole('button', { name: 'Search' });
  await button.click();

  const request = await vi.waitFor(() =>
    ctrl.expectOne('/flight?from=Paris&to=London'),
  );
  request.flush([
    createTestFlight(1),
    createTestFlight(2),
    createTestFlight(3),
  ]);

  const headings = page.getByRole('heading', { name: 'Paris - London' });
  await expect.element(headings).toHaveLength(3);

  expect(flightStore.updateFilter).toBeCalled();
  // ulteriori opzioni
  expect(flightStore.updateFilter).toBeCalledTimes(1);
  expect(flightStore.updateFilter).toBeCalledWith('Paris', 'London');
});
```

Di default la spy **delega** al metodo reale dopo aver registrato la chiamata; con `.mockImplementation(...)` la rimpiazza con un comportamento custom:

```ts
vi.spyOn(flightStore, 'updateFilter').mockImplementation((_from, _to) => {
  // comportamento mock custom
});
```

> [!warning]
> `TestBed.inject(FlightStore)` cerca il servizio al **root level**. Se è fornito a livello componente, recuperalo dall'injector del componente: `fixture.debugElement.injector.get(FlightStore)`.

Collegamenti: [[05-state-management-services-signals]] (lo store).

## Testing Routed Components
> 📖 p.223

Per i componenti che dipendono dalla rotta corrente si usa il **`RouterTestingHarness`** (incluso nell'Angular Router), che simula la navigazione. Lo si istanzia con il metodo statico `create`, poi si chiama `navigateByUrl` per simulare l'utente che passa alla rotta da testare.

```ts
import { RouterTestingHarness } from '@angular/router/testing';

describe('FlightEdit (router)', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [FlightEdit],
      providers: [
        // rotte di test
        provideRouter([
          { path: 'flight-edit/:id', component: FlightEdit },
        ]),
      ],
    }).compileComponents();
  });

  it('shows the route id in the id field', async () => {
    const harness = await RouterTestingHarness.create();
    await harness.navigateByUrl('/flight-edit/42');
    const input = page.getByLabelText('ID');
    await expect.element(input).toHaveValue(42);
  });
});
```

Collegamenti: [[04-router-navigation-lazy-loading]].

## Testing Timers & Debouncing — Mocking Delays
> 📖 pp.223-224

Componenti e servizi usano timer per ritardare certe azioni: il caso tipico è il [[glossario#debounce-debouncing|debounce]] dell'input (aspetta che l'utente smetta di digitare prima di reagire, invece di reagire a ogni tasto), per evitare elaborazioni eccessive. Quando possibile, gli autori raccomandano di **mockare** i ritardi per tenere i test veloci e affidabili. Perché ciò sia possibile, i tempi di debounce non vanno scritti fissi nel codice (*hardcodati*): si espongono via un oggetto di configurazione (*settings object*) così da poterli sovrascrivere.

```ts
// src/app/domains/shared/util-common/app-settings.ts
export interface AppSettings {
  readonly debounceTimeMs: number;
}
export const appSettings: AppSettings = {
  debounceTimeMs: 300,
};
```

> [!info] Angular 22+
> Le **Signal Forms** (`form()`, `debounce()`) sono la nuova API form basata sui signal, introdotta in Angular 21+. Qui `ReactiveFlightSearch` le usa per fare il debounce dell'input prima di lanciare la ricerca. Vedi [[06-signal-forms]].

```ts
// versione con Signal Forms che debounce l'input (cap.6)
protected readonly filterForm = form(this.filter, (path) => {
  debounce(path.from, appSettings.debounceTimeMs);
  debounce(path.to, appSettings.debounceTimeMs);
});
```

Per influenzare il tempo di debounce nei test, la suite spia il **getter** `debounceTimeMs` e ne forza il valore a `0`:

```ts
import { appSettings } from '../../../shared/util-common/app-settings';

// spy su un GETTER: azzera il debounce
vi.spyOn(appSettings, 'debounceTimeMs', 'get').mockReturnValue(0);
fixture = TestBed.createComponent(ReactiveFlightSearch);
component = fixture.componentInstance;
```

> [!warning]
> Il mock va impostato **prima** di `createComponent`: il valore serve quando il componente inizializza lo schema della Signal Form.

## Fake Timers
> 📖 pp.225-227

Quando il mocking non è praticabile, i **fake timer** di Vitest controllano e simulano lo scorrere del tempo senza attendere davvero. Metodi principali su `vi`:
- `vi.useFakeTimers()` — attiva la modalità fake timer: rimpiazza `setTimeout`/`setInterval` con mock.
- `vi.advanceTimersByTime(ms)` — avanza i fake timer di `ms`, eseguendo i timer scaduti in quel lasso.
- `vi.runAllTimersAsync()` — esegue subito tutti i timer pendenti.
- `vi.useRealTimers()` — ripristina i timer reali (mettilo in `afterEach`, così i test non si influenzano a vicenda).

```ts
describe('reactive-flight-search with fake timers', () => {
  let component: ReactiveFlightSearch;
  let fixture: ComponentFixture<ReactiveFlightSearch>;
  let ctrl: HttpTestingController;

  beforeEach(async () => {
    vi.useFakeTimers();
    await TestBed.configureTestingModule({
      imports: [ReactiveFlightSearch],
      providers: [
        provideRouter([]),
        provideHttpClientTesting(),
        provideTestConfig(),
      ],
    }).compileComponents();

    fixture = TestBed.createComponent(ReactiveFlightSearch);
    component = fixture.componentInstance;
    ctrl = TestBed.inject(HttpTestingController);

    await vi.runAllTimersAsync();                       // prima di expectOne: fa girare l'effect dopo il debounce
    const request = ctrl.expectOne('/flight?from=Graz&to=Hamburg');
    request.flush([]);
    await vi.runAllTimersAsync();                       // dopo flush: la resource risolve una Promise (microtask)
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it('can be created', () => {
    expect(component).not.toBeUndefined();
  });

  it('searches for flights when from and to are given', async () => {
    const flightStore = TestBed.inject(FlightStore);
    vi.spyOn(flightStore, 'updateFilter');

    await page.getByLabelText('From').fill('Paris');
    await vi.runAllTimersAsync();                       // fa scadere il debounce
    await page.getByLabelText('To').fill('London');
    await vi.runAllTimersAsync();

    const request = ctrl.expectOne('/flight?from=Paris&to=London');
    request.flush([
      createTestFlight(1),
      createTestFlight(2),
      createTestFlight(3),
    ]);
    await vi.runAllTimersAsync();

    const headings = page.getByRole('heading', { name: 'Paris - London' });
    await expect.element(headings).toHaveLength(3);
    expect(flightStore.updateFilter).toBeCalled();
    expect(flightStore.updateFilter).toBeCalledTimes(3);
    expect(flightStore.updateFilter).toBeCalledWith('Paris', 'London');
  });
});
```

> [!warning]
> Con `httpResource` + fake timer serve `runAllTimersAsync()` **due volte**: *prima* di `expectOne` (il timer di debounce deve scadere e poi l'effect che lancia l'HTTP gira in un microtask successivo) e *dopo* il `flush` (alla risposta, l'`httpResource` risolve internamente una Promise → altro microtask). La complessità stessa di questo caso mostra perché spesso **il mocking è la scelta migliore**.

## Testing Services
> 📖 pp.227-228

La community concorda che la maggioranza dei test frontend dovrebbero essere **component test**; testare i servizi in isolamento ha senso soprattutto per servizi molto riusabili e con logica complessa. Tutto ciò che vale per `TestBed`, mock e spy si applica anche qui: l'unica differenza è che, invece di creare una fixture, si recupera il servizio direttamente con `TestBed.inject`.

```ts
describe('flight-store', () => {
  let ctrl: HttpTestingController;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      providers: [FlightStore, provideHttpClientTesting(), provideTestConfig()],
    });
    ctrl = TestBed.inject(HttpTestingController);
  });

  it('loads flights when from and to given', async () => {
    const store = TestBed.inject(FlightStore);
    store.updateFilter('Paris', 'London');

    const request = await vi.waitFor(
      () => ctrl.expectOne('/flight?from=Paris&to=London'),
      { interval: 0 },
    );
    request.flush([
      createTestFlight(1),
      createTestFlight(2),
      createTestFlight(3),
    ]);

    await vi.waitFor(
      () => {
        const flights = store.flightsValue();
        expect(flights.length).toBe(3);
      },
      { interval: 0 },
    );
    ctrl.verify();
  });
});
```

Collegamenti: [[05-state-management-services-signals]].

## Determining Test Coverage
> 📖 pp.228-229

La CLI aiuta a individuare candidati per nuovi test misurando quali porzioni di codice sono coperte dai test esistenti e quanto bene (**test coverage**):

```bash
ng test --coverage
```

Esegue i test e genera un sito statico in `coverage/<project-name>` (es. `coverage/flights`). Selezionando un file vedi quali righe sono attraversate dai test e quante volte.

## 🔁 Ripasso lampo

**1.** Cosa rappresentano `describe` e `it`, perché non vanno importati, e cos'è il pattern AAA?
> [!success]- Risposta
> `describe` definisce una test **suite** (gruppo di casi correlati), `it` un test **case** (verifica una porzione di funzionalità). Non si importano perché Vitest le rende **globali** durante l'esecuzione (come `beforeEach`, `expect`, `vi`). Il pattern **AAA** struttura il caso in tre fasi: *Arrange* (prepara), *Act* (esegue l'azione da testare), *Assert* (verifica il risultato con `expect` + matcher).

**2.** Perché in browser mode si preferisce l'oggetto `page` (+ `expect.element`) al `debugElement`? Perché i locator usano ARIA e non `id`/`name`?
> [!success]- Risposta
> `page` è meno verboso, orientato ad ARIA e ha **retry** integrato fino a un timeout; il `debugElement` è di basso livello (selettori CSS, niente retry, devi dispatchare a mano l'evento `input`). I locator indirizzano ruoli e attributi **ARIA** di proposito, per spingere a costruire app accessibili; non offrono accesso per `id`/`name`. Come ultima spiaggia c'è `getByTestId` (`data-testid`), da usare con parsimonia perché mina l'accessibilità.

**3.** Cosa fa `provideHttpClientTesting()`, e a cosa servono `request.flush(...)` e `ctrl.verify()`?
> [!success]- Risposta
> `provideHttpClientTesting()` sostituisce i servizi interni dell'`HttpClient` con mock che **intercettano** le richieste e restituiscono risposte fake (vale anche per `httpResource`). Ogni richiesta resta in pausa finché non la si risolve con `request.flush(...)` (la risposta fake). `ctrl.verify()` lancia un'eccezione se restano richieste **senza risposta** → tipicamente in `afterEach`.

**4.** Perché serve `vi.waitFor(() => ctrl.expectOne(...))` quando il dato arriva da un `httpResource`?
> [!success]- Risposta
> Perché l'`httpResource` **non** parte subito: usa un `effect` che osserva il request signal, e gli effect girano in un **microtask successivo**. Al momento dell'`expectOne` la richiesta potrebbe non esistere ancora. `vi.waitFor` riprova la funzione finché ha successo o scatta il timeout (spesso basta `{ interval: 0 }`), aspettando che l'effect parta.

**5.** Differenza tra mockare i ritardi (settings object) e usare i fake timer? Perché con `httpResource` + fake timer servono due `runAllTimersAsync()`?
> [!success]- Risposta
> Mockare il ritardo significa azzerarlo alla fonte: si spia il getter `debounceTimeMs` e si forza `0` con `mockReturnValue(0)` (più semplice; va fatto **prima** di `createComponent`). I **fake timer** (`vi.useFakeTimers`) invece simulano lo scorrere del tempo e si fanno avanzare con `runAllTimersAsync`. Con `httpResource` ne servono due: uno **prima** di `expectOne` (scaduto il debounce, l'effect che lancia l'HTTP gira in un microtask) e uno **dopo** il `flush` (la resource risolve una Promise → altro microtask).

**6.** Come si testa un componente che dipende dalla rotta corrente? E un servizio in isolamento?
> [!success]- Risposta
> Per il componente routed si usa il **`RouterTestingHarness`**: lo si crea con `RouterTestingHarness.create()` e si simula la navigazione con `harness.navigateByUrl('/...')`. Per il servizio in isolamento valgono `TestBed`, mock e spy come per i componenti, ma invece di creare una fixture lo si recupera direttamente con `TestBed.inject(...)` e se ne testano i metodi.

**In sintesi:**
- **Vitest** è il test runner di default della CLI; la maggioranza dei test sono **component test** in **browser mode** (provider Playwright) che imitano l'utente via locator ARIA + `page` + `expect.element`.
- Il **`TestBed`** monta i componenti e, grazie alla DI, sostituisce le dipendenze con **mock** (`useValue`, `overrideComponent` per i servizi a livello componente, stub per i figli → shallow testing).
- L'**`HttpTestingController`** (`provideHttpClientTesting`) risponde alle chiamate HTTP con dati fake; con `httpResource` bisogna tenere conto dei microtask di effect/Promise (`vi.waitFor`, `runAllTimersAsync`).
- **Spy** (`vi.spyOn`) per i test gray-box, **fake timer** per debounce/delay (ma mockare i ritardi è spesso più semplice), e `ng test --coverage` per la copertura.
