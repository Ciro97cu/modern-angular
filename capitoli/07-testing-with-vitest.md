---
capitolo: 7
titolo: "Modern Testing with Vitest"
pagine: "193-219"
tags: [tipo/capitolo, testing, http]
---
# 07 · Modern Testing with Vitest
> 📖 cap.7 · pp.193-219 — *Modern Angular* v1.0.4

Con sempre più logica nel frontend, il test manuale non basta a mantenere la stabilità dell'app. La Angular CLI integra **Vitest** come test runner di default: esecuzione veloce, ottima DX, e i costrutti Angular (`TestBed`, `HttpTestingController`, locator di Vitest) per montare i componenti e mockare le dipendenze. Il capitolo costruisce i test sulla feature `flight-search` già vista nel [[02-signal-based-components|cap.2]].

## Structure of a Vitest Test
> 📖 pp.193-195

Vitest offre `describe` (test **suite**, raggruppa casi correlati) e `it` (test **case**). Le funzioni `describe`, `it`, `beforeEach`, `expect` sono **globali**: non vanno importate. I file di test hanno estensione **`.spec.ts`** e si mettono accanto al codice testato (così restano aggiornati).

```ts
// src/app/demo.spec.ts
const add = (a: number, b: number) => a + b;   // Object under test

describe('Add', () => {
  beforeEach(() => {                            // gira prima di OGNI test case
    console.log('Preparation tasks ...');
  });

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

- Pattern **AAA**: Arrange (prepara) → Act (esegue l'azione) → Assert (verifica il risultato).
- Hook del ciclo di vita: `beforeEach` / `afterEach` (prima/dopo ogni caso), `beforeAll` / `afterAll` (una volta per suite).
- I metodi su `expect(...)` si chiamano **matcher** (concatenabili, scoprili con l'autocompletamento dell'IDE):

```ts
expect(c).not.toBe(4);
expect(c).not.toBeNull();
expect(c).toBeGreaterThan(0);
const str = 'Result: ' + c;
expect(str).toContain('Result');
expect(str).toMatch(/Result/);
```

> [!tip] Take-away
> In ottica **BDD**, suite + nome del caso letti insieme devono descrivere il comportamento atteso: `Add` + `correctly adds 1 and 2`. Le `describe` si annidano liberamente.

## Skipping Tests
> 📖 p.195

Per saltare una suite o un caso (es. un `app.spec.ts` generato e mai mantenuto che ora fallisce): sostituisci `describe` con **`describe.skip`** (e `it` con **`it.skip`**).

```ts
// src/app/app.spec.ts
describe.skip('App', () => {
  // [...]
});
```

## Turning on Browser Mode
> 📖 pp.195-196

Di default Vitest gira in **Node.js**, ma per Angular si raccomanda la **browser mode**: rispecchia il comportamento reale dell'app e rende più realistici i component test. Serve un **provider** per accedere al browser; gli autori consigliano **Playwright** (più moderno e semplice di webdriver.io). Vitest lo usa "sotto il cofano": i test **non** diventano test Playwright.

```bash
ng add @vitest/browser-playwright
npx playwright install
```

Si attiva nodo `projects/<project>/architect/test` di `angular.json`:

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

- Browser supportati dal provider Playwright: `Chromium`, `Firefox`, `Webkit` (Safari) + varianti `…Headless`.
- Di default la CLI parte in **watch mode** (ri-esegue ad ogni modifica). In CI si imposta `watch: false`.

## Running Tests
> 📖 p.196

```bash
ng test                      # esegue i test (watch mode di default)
ng test --configuration=ci   # usa la configurazione "ci"
```

A sinistra l'elenco dei test (verde = passati), a destra info per run (stack trace in caso di errore), al centro Vitest **monta** i componenti (di solito un lampo, ma utile in debug). In mancanza di supporto IDE, si possono debuggare i test nei dev tools del browser.

## Angular and Vitest: Preparing the TestBed
> 📖 pp.197-199

Il **`TestBed`** è il costrutto centrale: ci "attacchi" l'oggetto sotto test, gli fornisci le dipendenze e può **istanziare** i componenti (inietta i servizi e dà vita al template → interazione a livello DOM).

```ts
// .../feature-booking/flight-search/flight-search.spec.ts
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
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

`createComponent` restituisce una **`ComponentFixture<T>`** con, fra gli altri:
- `componentInstance` — l'istanza della classe componente (qui `FlightSearch`).
- `nativeElement` — il nodo DOM del componente.
- `debugElement` — accesso a aspetti aggiuntivi (es. l'injector a livello componente, query su nodi/servizi figli).

Collegamenti: [[inject]] · [[providers]] · [[04-router-navigation-lazy-loading]] (il `provideRouter`).

## A First Component Test & Locators
> 📖 pp.199-201

Il primo test tratta il componente come **black box**: interagisce via DOM come farebbe l'utente. In browser mode Vitest fornisce l'oggetto **`page`**; con **`expect.element`** si ottengono assert "browser-aware" (es. `toBeDisabled`).

```ts
import { page } from 'vitest/browser';

it('disables search button when from and to are not given', async () => {
  await page.getByLabelText('From').fill('');
  await page.getByLabelText('To').fill('');
  const button = page.getByRole('button', { name: 'Search' });
  await expect.element(button).toBeDisabled();
});
```

I **locator** (`getByLabelText`, `getByRole`, …) cercano elementi tramite **ruoli e attributi ARIA**, incoraggiando l'accessibilità. Un nome accessibile deriva da `<label for>`, da `aria-label`, o dal testo del bottone.

```html
<label for="from">From:</label>
<input [formField]="filterForm.from" id="from" />
<!-- in alternativa: -->
<input [formField]="filterForm.from" id="from" aria-label="from" />
<button (click)="search()" [disabled]="!from() || !to()">Search</button>
```

- Di default i locator sono **case-insensitive** e fanno match su **sottostringa**; `{ exact: true }` per il match esatto, oppure una **regex**.
- Un locator può restituire **più elementi** (`toHaveLength(n)`).

```ts
const button = page.getByRole('button', { name: 'Search', exact: true });
await page.getByLabelText(/From|Airport of Departure/i).fill('Paris');

const headings = page.getByRole('heading', { name: 'Paris - London' });
await expect.element(headings).toHaveLength(3);
```

I locator **descrivono solo come trovare** gli elementi: il recupero avviene solo all'uso (`fill`, `click`, `expect.element`), con **retry** fino a un timeout (`{ interval, timeout }`, default `interval: 50, timeout: 100`). Per il nodo immediato: `.element()` (o `.elements()` se più nodi).

> [!warning] Gotcha
> I locator non offrono accesso per `id` o `name`: è intenzionale, per spingere verso ARIA. Come ultima spiaggia `page.getByTestId(...)` legge un attributo `data-testid`, ma mina l'accessibilità → usalo con parsimonia.

## Locating Elements via DebugElement
> 📖 p.202

Il `debugElement` permette query DOM via **selettori CSS** (non ARIA), ma è di **basso livello**: niente retry e devi **dispatchare a mano** l'evento `input` per notificare Angular.

```ts
import { By } from '@angular/platform-browser';

const to = fixture.debugElement.query(By.css('input[name=to]')).nativeElement;
to.value = 'London';
to.dispatchEvent(new Event('input'));   // necessario per notificare Angular
```

> [!tip] Take-away
> Preferisci l'oggetto **`page`** di Vitest (browser mode) al `debugElement`: meno verboso, ARIA-oriented e con retry integrato.

## Defining Default Timeouts
> 📖 pp.202-203

Timeout per-chiamata, per-suite/caso (oggetto `TestOptions` passato a `describe`/`it`), o globale via setup file.

```ts
const suiteOptions: TestOptions = { timeout: 200 };
const caseOptions: TestOptions = { timeout: 300 };

describe('FlightEdit (router)', suiteOptions, () => {
  it('navigates to flight details on click', caseOptions, async () => {
    // [...]
  });
});
```

```ts
// src/vitest-setup.ts
import { vi } from 'vitest';
vi.setConfig({ testTimeout: 3_000 });
```

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
> 📖 pp.203-205

I test isolano l'oggetto sotto test dalle dipendenze (errori attribuibili, test veloci e affidabili). Si sostituiscono con **mock** (implementazioni semplificate); le **spy** monitorano le chiamate. Grazie alla DI di Angular basta dire al `TestBed` di fornire un mock al posto del servizio reale.

```ts
import { ConfigService } from '../domains/shared/util-common/config-service';

await TestBed.configureTestingModule({
  imports: [FlightSearch],
  providers: [
    provideRouter([]),
    { provide: ConfigService, useValue: { baseUrl: '', model: '' } },  // mock
  ],
}).compileComponents();
```

`useValue` non è **tipizzato** e va duplicato fra test → conviene estrarre il provider in una funzione riusabile:

```ts
// src/app/testing/provide-test-config.ts
import { Provider } from '@angular/core';
import { Config, ConfigService } from '../domains/shared/util-common/config-service';

export function provideTestConfig(): Provider {
  return {
    provide: ConfigService,
    useValue: { baseUrl: '', model: '' },
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
> 📖 p.205

Se il servizio è fornito **a livello componente** (array `providers` del `@Component`), il `TestBed` non basta: si **sovrascrivono i metadati** del componente con `overrideComponent`.

```ts
TestBed.overrideComponent(FlightSearch, {
  add: {
    providers: [{ provide: LanguageService, useClass: DefaultLanguageService }],
  },
});
```

Questo agisce sul componente, sui suoi figli e sugli altri servizi allo stesso livello.

## Mocking Child Components & Shallow Testing
> 📖 pp.205-206

Per evitare test cross-componente (`FlightSearch` insieme a `FlightCard`) e fare **shallow testing**, si mocka il figlio con uno **stub** che ha **stesso selector, stessi input e output**, poi si scambiano gli `imports` nei metadati.

```ts
TestBed.overrideComponent(FlightSearch, {
  remove: { imports: [FlightCard] },
  add: { imports: [DummyFlightCard] },
});
```

## Mocking HTTP Calls
> 📖 pp.206-210

Per essere indipendenti dal backend si mockano le chiamate HTTP. Angular fornisce mock per i servizi interni dell'`HttpClient` via **`provideHttpClientTesting()`**: intercettano le richieste e danno risposte fake (vale anche per **`httpResource`**, che si appoggia all'`HttpClient`).

```ts
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { page } from 'vitest/browser';
import { provideTestConfig } from '../../../../testing/provide-test-config';

let ctrl: HttpTestingController;

beforeEach(async () => {
  await TestBed.configureTestingModule({
    imports: [FlightSearch],
    providers: [
      provideRouter([]),
      provideTestConfig(),
      provideHttpClientTesting(),    // mock dei provider HTTP
    ],
  }).compileComponents();

  fixture = TestBed.createComponent(FlightSearch);
  component = fixture.componentInstance;
  ctrl = TestBed.inject(HttpTestingController);

  // attende il caricamento iniziale (httpResource allo startup)
  const request = await vi.waitFor(() => ctrl.expectOne('/flight?from=Graz&to=Hamburg'));
  request.flush([]);
});
```

L'`HttpTestingController` informa di ogni richiesta e ne accetta la risposta fake con `flush(...)`: **ogni chiamata resta in pausa** finché il test non risponde.

> [!warning] Gotcha
> L'`httpResource` **non** esegue la chiamata HTTP subito: usa un [[effect]] che osserva il request signal, e gli effect girano in un **microtask successivo**. Quindi prima di `expectOne` bisogna **attendere** l'effect con `vi.waitFor(...)` (riprova fino a successo/timeout). Senza, la richiesta non esiste ancora e il componente resta in loading.

```ts
const request = await vi.waitFor(
  () => ctrl.expectOne('/flight?from=Graz&to=Hamburg'),
  { interval: 50, timeout: 1000 },   // default configurabili; { interval: 0 } basta spesso
);
```

`vi` è globale o va importato: `import { vi } from 'vitest';`. Test case completo (riempie i campi, cerca, risponde con 3 voli):

```ts
it('searches for flights when from and to are given', async () => {
  await page.getByLabelText('From').fill('Paris');
  await page.getByLabelText('To').fill('London');
  const button = page.getByRole('button', { name: 'Search' });
  await button.click();

  const request = await vi.waitFor(() => ctrl.expectOne('/flight?from=Paris&to=London'));
  request.flush([createTestFlight(1), createTestFlight(2), createTestFlight(3)]);

  const headings = page.getByRole('heading', { name: 'Paris - London' });
  await expect.element(headings).toHaveLength(3);
});
```

```ts
// helper per voli di test
export function createTestFlight(id: number, from = 'Paris', to = 'London') {
  const date = new Date().toISOString();
  const delayed = false;
  return { id, from, to, date, delayed };
}
```

> [!tip] Take-away
> Aggiungi un `afterEach(() => ctrl.verify())`: `verify()` lancia se restano richieste **senza risposta**. Mockando il risultato non verifichi il backend, ma che il componente faccia le **richieste attese** e gestisca le risposte.

Collegamenti: [[resource|httpResource]] · [[02-signal-based-components]] (la feature `flight-search`).

## Gray-Box Testing with Spies
> 📖 pp.210-211

A volte serve verificare che certi metodi **interni** vengano chiamati correttamente → test **gray-box**. Le **spy** di Vitest (`vi.spyOn`) sono proxy che decorano una funzione e ricordano parametri e numero di chiamate.

```ts
it('searches for flights when from and to are given', async () => {
  const flightStore = TestBed.inject(FlightStore);
  vi.spyOn(flightStore, 'updateFilter');

  await page.getByLabelText('From').fill('Paris');
  await page.getByLabelText('To').fill('London');
  await page.getByRole('button', { name: 'Search' }).click();

  const request = await vi.waitFor(() => ctrl.expectOne('/flight?from=Paris&to=London'));
  request.flush([createTestFlight(1), createTestFlight(2), createTestFlight(3)]);

  expect(flightStore.updateFilter).toBeCalled();
  expect(flightStore.updateFilter).toBeCalledTimes(1);
  expect(flightStore.updateFilter).toBeCalledWith('Paris', 'London');
});
```

Di default la spy **delega** al metodo reale dopo aver registrato; con `.mockImplementation(...)` la rimpiazza:

```ts
vi.spyOn(flightStore, 'updateFilter').mockImplementation((_from, _to) => {
  // comportamento mock custom
});
```

> [!warning] Gotcha
> `TestBed.inject(FlightStore)` cerca al **root level**. Se il servizio è fornito a livello componente, recuperalo dall'injector del componente: `fixture.debugElement.injector.get(FlightStore)`.

Collegamenti: [[05-state-management-services-signals]] (lo store).

## Testing Routed Components
> 📖 p.212

Per i componenti che dipendono dalla rotta corrente si usa il **`RouterTestingHarness`** (Angular Router), che simula la navigazione.

```ts
import { RouterTestingHarness } from '@angular/router/testing';

describe('FlightEdit (router)', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [FlightEdit],
      providers: [
        provideRouter([{ path: 'flight-edit/:id', component: FlightEdit }]),
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
> 📖 pp.212-213

Componenti/servizi usano timer per ritardare azioni (tipico: **debounce** dell'input). Quando possibile, **mockare** i ritardi tiene i test veloci. Per i tempi di debounce: non hardcodarli, esponili via un settings object così da poterli sovrascrivere.

```ts
// src/app/domains/shared/util-common/app-settings.ts
export interface AppSettings {
  readonly debounceTimeMs: number;
}
export const appSettings: AppSettings = { debounceTimeMs: 300 };
```

```ts
// versione con Signal Forms che debounce l'input (cap.6)
protected readonly filterForm = form(this.filter, (path) => {
  debounce(path.from, appSettings.debounceTimeMs);
  debounce(path.to, appSettings.debounceTimeMs);
});
```

```ts
import { appSettings } from '../../../shared/util-common/app-settings';

// spy su un GETTER: azzera il debounce PRIMA di creare il componente
vi.spyOn(appSettings, 'debounceTimeMs', 'get').mockReturnValue(0);
fixture = TestBed.createComponent(ReactiveFlightSearch);
component = fixture.componentInstance;
```

> [!warning] Gotcha
> Il mock va impostato **prima** di `createComponent`: il valore serve quando il componente inizializza lo schema della Signal Form. Collegamenti: [[06-signal-forms]].

## Fake Timers
> 📖 pp.214-216

Quando il mocking non è praticabile, i **fake timer** di Vitest controllano lo scorrere del tempo senza attendere davvero:
- `vi.useFakeTimers()` — attiva la modalità (rimpiazza `setTimeout`/`setInterval`).
- `vi.advanceTimersByTime(ms)` — avanza di `ms`, eseguendo i timer scaduti.
- `vi.runAllTimersAsync()` — esegue subito tutti i timer pendenti.
- `vi.useRealTimers()` — ripristina i timer reali (mettilo in `afterEach`!).

```ts
describe('reactive-flight-search with fake timers', () => {
  beforeEach(async () => {
    vi.useFakeTimers();
    await TestBed.configureTestingModule({
      imports: [ReactiveFlightSearch],
      providers: [provideRouter([]), provideHttpClientTesting(), provideTestConfig()],
    }).compileComponents();

    fixture = TestBed.createComponent(ReactiveFlightSearch);
    component = fixture.componentInstance;
    ctrl = TestBed.inject(HttpTestingController);

    await vi.runAllTimersAsync();                       // prima di expect: fa girare l'effect
    const request = ctrl.expectOne('/flight?from=Graz&to=Hamburg');
    request.flush([]);
    await vi.runAllTimersAsync();                       // dopo flush: la resource risolve una Promise (microtask)
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it('searches for flights when from and to are given', async () => {
    const flightStore = TestBed.inject(FlightStore);
    vi.spyOn(flightStore, 'updateFilter');

    await page.getByLabelText('From').fill('Paris');
    await vi.runAllTimersAsync();                       // fa scadere il debounce
    await page.getByLabelText('To').fill('London');
    await vi.runAllTimersAsync();

    const request = ctrl.expectOne('/flight?from=Paris&to=London');
    request.flush([createTestFlight(1), createTestFlight(2), createTestFlight(3)]);
    await vi.runAllTimersAsync();

    const headings = page.getByRole('heading', { name: 'Paris - London' });
    await expect.element(headings).toHaveLength(3);
    expect(flightStore.updateFilter).toBeCalledTimes(3);
  });
});
```

> [!warning] Gotcha
> Con `httpResource` + fake timer serve `runAllTimersAsync()` sia **prima** di `expectOne` (l'effect che lancia l'HTTP gira in un microtask, dopo che il timer di debounce è scaduto) sia **dopo** il `flush` (la resource risolve una Promise → altro microtask). La complessità stessa di questo caso mostra perché spesso **il mocking è la scelta migliore**.

## Testing Services
> 📖 pp.216-217

La community concorda che la maggioranza dei test frontend dovrebbero essere **component test**; testare i servizi in isolamento ha senso per servizi molto riusabili e con logica complessa. Tutto ciò che vale per `TestBed`/mock/spy si applica: l'unica differenza è che si recupera il servizio con `TestBed.inject` invece di creare una fixture.

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
    request.flush([createTestFlight(1), createTestFlight(2), createTestFlight(3)]);

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
> 📖 pp.217-218

La CLI individua candidati per nuovi test misurando quali parti del codice sono coperte (**test coverage**):

```bash
ng test --coverage
```

Genera un sito statico in `coverage/<project-name>` (es. `coverage/flights`). Selezionando un file vedi quali righe sono attraversate dai test e quante volte.

## 🔁 Ripasso lampo
1. Cosa rappresentano `describe` e `it`, e perché non vanno importati? Cos'è il pattern AAA?
2. Perché in browser mode si preferisce l'oggetto `page` (+ `expect.element`) al `debugElement`? Perché i locator usano ARIA e non `id`/`name`?
3. Cosa fa `provideHttpClientTesting()` e a cosa serve `ctrl.flush(...)` / `ctrl.verify()`?
4. Perché serve `vi.waitFor(() => ctrl.expectOne(...))` quando il dato arriva da un `httpResource`?
5. Differenza tra mockare i ritardi (settings object) e usare i fake timer? Perché con `httpResource` + fake timer servono due `runAllTimersAsync()`?
6. Come si testa un componente che dipende dalla rotta corrente? E un servizio in isolamento?

**Take-away del capitolo:**
- **Vitest** è il test runner di default della CLI; la maggioranza dei test sono **component test** in **browser mode** (provider Playwright) che imitano l'utente via locator ARIA + `page` + `expect.element`.
- Il **`TestBed`** monta i componenti e, grazie alla DI, sostituisce le dipendenze con **mock** (`useValue`, `overrideComponent` per i servizi a livello componente, stub per i figli → shallow testing).
- L'**`HttpTestingController`** (`provideHttpClientTesting`) risponde alle chiamate HTTP con dati fake; con `httpResource` ricorda i microtask degli effect/Promise (`vi.waitFor`, `runAllTimersAsync`).
- **Spy** (`vi.spyOn`) per i test gray-box, **fake timer** per debounce/delay (ma il mocking è spesso più semplice), e `ng test --coverage` per la copertura.
