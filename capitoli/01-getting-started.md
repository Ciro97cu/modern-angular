---
capitolo: 1
titolo: "Getting Started with Angular"
pagine: "18-31"
tags: [tipo/capitolo, tooling, signals, angular-22]
---
# 01 · Getting Started with Angular
> 📖 cap.1 · pp.18-31 — *Modern Angular* v2.0.0

Setup dell'ambiente, generazione del progetto con la **Angular CLI** e prima lettura del codice generato. Un'app Angular è un **albero di componenti** con un root component in cima; questo capitolo arriva fino al punto in cui quel componente gira nel browser. Il progetto di esempio del libro è **flights42** (`angular-architects/flights42` su GitHub).

## Tooling
> 📖 pp.18-20

- **IDE**: in teoria basta un editor di testo, ma un IDE specializzato dà syntax highlighting, code completion e debugging integrato. Gli autori consigliano **VS Code** (gratuito, multipiattaforma) o **WebStorm/IntelliJ** (commerciali, JetBrains).
  - Su VS Code installa l'extension pack *Angular Essentials* di John Papa (View → Extensions): include l'**Angular Language Service** (code completion nei template HTML) ed **ESLint**.
  - Su IntelliJ verifica che i plugin **Angular** e **TypeScript** siano attivi.
- **Node.js**: tutto il tooling di sviluppo/build/test poggia su Node. Usa le versioni **LTS**; se ti servono versioni diverse tra progetti, gestiscile con un version manager come **NVM**.
- **Angular CLI**: tool ufficiale del team Angular per generare, buildare e testare; viene aggiornato a ogni nuova versione di Angular. Installazione globale (`-g` la rende disponibile ovunque sulla macchina; senza `-g` npm installerebbe solo nel progetto locale):

```bash
npm install -g @angular/cli
```

- **Progetto di esempio**: si clona da GitHub e si avvia con la CLI.

```bash
git clone https://github.com/angular-architects/flights42.git
cd flights42
npm install
ng serve -o   # -o apre il browser
```

> [!tip]
> Il repo di esempio ha **branch per capitolo** (vedi `readme.md`): utili per seguire il libro passo-passo.

## Generare e avviare un progetto
> 📖 pp.20-22

`ng new` scarica e configura automaticamente l'intera toolchain: compilatore TypeScript, strumenti di test e i build tool che producono i bundle ottimizzati per la produzione.

```bash
ng new flights42     # genera struttura + toolchain; rispondi Enter alle domande
cd flights42
ng serve -o          # dev server, default http://localhost:4200
ng serve -o --port 4242   # porta custom con --port
```

`ng serve` non solo serve l'app, ma **monitora i sorgenti** e li ricompila a ogni modifica, aggiornando poi la finestra del browser (live reload). Se la porta `4200` è occupata, la CLI ne propone un'altra. Provalo cambiando il `title` in `app.ts`:

```ts
protected readonly title = signal('World!');
```

salva, e il browser si aggiorna da solo (riporta poi il valore a `flights42` per restare allineato col capitolo).

> [!warning]
> Apri in IDE la **cartella root del progetto** (quella con `angular.json`), altrimenti autocompletamento ed errori vanno in tilt. La ricompilazione automatica funziona bene ma occasionalmente la CLI "perde" una modifica — capita con salvataggi rapidi in sequenza o rinomine di file. Rimedio: risalva i file interessati o, in ultima istanza, riavvia `ng serve`.

## Struttura del progetto
> 📖 pp.22-23

La CLI genera il root component `App` più i file di configurazione per build e test. I principali:

| File | Ruolo |
|---|---|
| `src/app/app.ts` | Codice TypeScript del root component (il suo comportamento) |
| `src/app/app.html` | Template del root component (il suo aspetto) |
| `src/main.ts` | Entry point: fa il **bootstrap** del root component |
| `src/index.html` | Start page; la build vi aggiunge i riferimenti ai bundle generati |
| `src/styles.css` | Stili globali |
| `package.json` | Librerie e versioni; `npm install` le scarica in `node_modules` |
| `angular.json` | Configurazione della CLI (riferimenti agli style, setup di test, ecc.) |
| `tsconfig.json` | Configurazione del compilatore TypeScript |

## Il root component & i signal
> 📖 pp.23-25

Il root component generato si chiama `App` e vive in `app.ts`. Definisce essenzialmente una proprietà `title`:

```ts
// src/app/app.ts
import { Component, signal } from "@angular/core";
import { RouterOutlet } from "@angular/router";

@Component({
  selector: "app-root",
  imports: [RouterOutlet],
  templateUrl: "./app.html",
  styleUrl: "./app.css",
})
export class App {
  protected readonly title = signal("flights42");

  // metodo aggiunto per aggiornare il signal title
  protected updateTitle(): void {
    this.title.set("Highly Sophisticated Flight App");
    console.log("Title updated", this.title());
  }
}
```

- `title` è un [[signal]]: un oggetto che **contiene un valore** e notifica le parti interessate quando cambia; Angular usa questa notifica per aggiornare la UI. Lo **leggi chiamandolo come funzione** — `this.title()` — e lo aggiorni con `.set()`. Il tipo è `WritableSignal<string>`, inferito dal valore di default (non serve dichiararlo).
- Secondo la Angular style guide: le proprietà usate **solo nel template** si dichiarano `protected` (evita modifiche accidentali dall'esterno della classe). I signal vanno marcati `readonly`: non si **rimpiazzano**, si aggiornano.
- `@Component` è un **decorator** che marca la classe come componente (i decorator definiscono metadati per i building block di Angular e sono preceduti da `@`). Il `selector` è il nome dell'elemento HTML custom che rappresenta il componente: lo richiami con `<app-root></app-root>`. Il decorator referenzia anche template (`templateUrl`) e CSS locale (`styleUrl`, vuoto di default).
- `export` rende la classe usabile in altri file; gli `import` in cima portano dentro i costrutti di Angular (la funzione `signal`, il decorator `Component`, ecc.).
- `imports` elenca i costrutti che il **template** usa: qui `RouterOutlet`, il placeholder con cui il router mostra componenti diversi (→ [[04-router-navigation-lazy-loading]]).

Sostituisci il template generato con questo frammento, che mostra due binding:

```html
<!-- src/app/app.html -->
<h1>Hello, {{ title() }}</h1>
<button (click)="updateTitle()">Update Title</button>
```

- `{{ title() }}` è una **interpolation**: nota la chiamata al getter del signal (`title()`) per leggerne il valore corrente.
- `(click)="updateTitle()"` è un **event binding**: parentesi tonde attorno al nome dell'evento.

Collegamenti: [[signal]] · approfondimenti su componenti e binding in [[02-signal-based-components]].

## Bootstrap dell'applicazione
> 📖 pp.25-27

All'avvio Angular esegue `main.ts`, che fa il **bootstrap** del root component: da lì in poi mostra l'intero albero di componenti.

```ts
// src/main.ts
import { bootstrapApplication } from "@angular/platform-browser";
import { appConfig } from "./app/app.config";
import { App } from "./app/app";

bootstrapApplication(App, appConfig).catch((err) => console.error(err));
```

`bootstrapApplication(rootComponent, config)` prende il root component e una configurazione dell'applicazione. La config registra i **servizi disponibili a livello globale** e sta in `app.config.ts`:

```ts
// src/app/app.config.ts
import {
  ApplicationConfig,
  provideBrowserGlobalErrorListeners,
} from "@angular/core";
import { provideRouter } from "@angular/router";
import { routes } from "./app.routes";

export const appConfig: ApplicationConfig = {
  providers: [provideBrowserGlobalErrorListeners(), provideRouter(routes)],
};
```

I servizi globali si registrano in `providers` tramite [[providers|provider functions]]: qui `provideBrowserGlobalErrorListeners()` (intercetta gli errori non gestiti della pagina; l'error handler di default li stampa nella console del browser) e `provideRouter(routes)` (configura il router). Nel resto del libro questa config si arricchisce di altri servizi.

```html
<!-- src/index.html -->
[...]
<body>
  <app-root></app-root>
</body>
[...]
```

`index.html` contiene `<app-root></app-root>` come **punto di innesto**: in fase di build la CLI compila e bundla i sorgenti e aggiunge qui i riferimenti ai bundle, uno dei quali contiene il codice di `main.ts` che avvia l'applicazione.

## CLI: pacchetti, componenti e setup di studio
> 📖 pp.27-30

- **Aggiungere pacchetti**:
  - `npm i <pkg>` (alias di `npm install <pkg>`) installa la libreria — es. `npm install @ngrx/signals`, usata più avanti per la gestione dello stato.
  - `ng add <pkg>` installa **e configura**: es. `ng add @angular/material` imposta theming e tipografia di Material (il libro ne usa parti selezionate per dialog e toast). Rispondi Enter alle domande per i default.
- **Componenti e stili pronti**: il libro fa copiare nel progetto i file del repo `angular-architects/flights42-assets` (uno `styles.css` globale e i componenti `navbar`/`sidebar` con i rispettivi template, più un `app.html` modificato che li referenzia) per non perdere tempo su styling e menu.
- **Configurare gli schematics** in `angular.json` (nodo `projects/<project-name>/schematics`) per ridurre il rumore in fase di studio e usare **OnPush**:

```jsonc
"schematics": {
  "@schematics/angular:component": {
    "style": "none",
    "skipTests": true,
    "changeDetection": "OnPush"
  },
  "@schematics/angular:directive": {
    "skipTests": true
  },
  "@schematics/angular:pipe": {
    "skipTests": true
  },
  "@schematics/angular:service": {
    "skipTests": true
  }
}
```

`skipTests: true` evita di generare i file di test per componenti, direttive, pipe e service — non perché lo scaffolding sia sbagliato, ma perché in [[07-testing-with-vitest]] questi file si scrivono a mano per mostrarne i concetti.

- **Linter**: `ng lint` esegue ESLint. La CLI lo **configura alla prima esecuzione** (Enter per i default → set di regole Angular + TS); personalizzi in `eslint.config.js`. In VS Code gli errori compaiono mentre scrivi se hai l'estensione ESLint (inclusa nell'Angular Essentials pack).
- **Build di produzione**: `ng build` compila TS → JS, bundla e ottimizza (minify + tree-shaking, che rimuove il codice inutilizzato). Output in `dist/<app>/browser`, es. `dist/flights42/browser`: copiali su un web server per il deploy.

> [!info] Angular 22+
> **OnPush** è la strategia di change detection raccomandata: Angular aggiorna un componente **solo quando i suoi dati cambiano** (es. quando un signal fornisce un nuovo valore) invece di ricontrollare tutta l'app. I signal la rendono naturale. **Da Angular 22 è il default.**

> [!tip]
> Anche se OnPush è ormai il default, il libro continua a scrivere `changeDetection: ChangeDetectionStrategy.OnPush` esplicitamente in **ogni** `@Component`, per due motivi: rende la scelta **visibile a colpo d'occhio** (codice auto-documentante) e resta **retrocompatibile** con versioni < 22, dove gli snippet copiati si comportano allo stesso modo.

Collegamenti: [[providers]] · gestione dello stato con NgRx in [[09-ngrx-signal-store]] · testing in [[07-testing-with-vitest]].

## 🔁 Ripasso lampo

**1.** Che differenza c'è tra `npm i <pkg>` e `ng add <pkg>`?
> [!success]- Risposta
> `npm i <pkg>` (alias di `npm install`) si limita a **installare** la libreria. `ng add <pkg>` la installa **e esegue passi di setup aggiuntivi** — es. `ng add @angular/material` configura theming e tipografia di Material nel progetto.

**2.** Come leggi e come aggiorni il valore di un `signal`? Perché si dichiara `readonly`?
> [!success]- Risposta
> Lo **leggi chiamandolo come funzione**: `this.title()`. Lo **aggiorni** con `.set()` (es. `this.title.set('...')`). Si dichiara `readonly` perché i signal **non si rimpiazzano, si aggiornano**: il riferimento all'oggetto signal resta lo stesso, cambia il valore che contiene.

**3.** Cosa fa `bootstrapApplication` e dove registri i servizi globali?
> [!success]- Risposta
> In `main.ts`, `bootstrapApplication(App, appConfig)` fa il **bootstrap** del root component, da cui Angular mostra l'intero albero di componenti. I servizi globali si registrano nell'array `providers` di `appConfig` (in `app.config.ts`) tramite **provider functions** come `provideRouter()` e `provideBrowserGlobalErrorListeners()`.

**4.** A cosa serve `OnPush` e perché i signal lo rendono conveniente?
> [!success]- Risposta
> `OnPush` aggiorna un componente **solo quando i suoi dati cambiano**, invece di ricontrollare l'intera app a ogni ciclo → change detection più efficiente. I signal notificano esplicitamente i cambi di valore, quindi calzano perfettamente con OnPush. Da **Angular 22** è la strategia di default.

**5.** Quali file guardi per capire root component, bootstrap e start page?
> [!success]- Risposta
> Root component: `src/app/app.ts` (logica) e `src/app/app.html` (template). Bootstrap: `src/main.ts` (chiama `bootstrapApplication`) e `src/app/app.config.ts` (i servizi globali). Start page: `src/index.html`, che contiene `<app-root></app-root>` come punto di innesto.

**6.** Perché il libro scrive `changeDetection: ChangeDetectionStrategy.OnPush` esplicitamente se da Angular 22 è già il default?
> [!success]- Risposta
> Per due ragioni: rende la scelta **visibile a colpo d'occhio** in ogni componente (codice auto-documentante) e resta **retrocompatibile** con versioni di Angular precedenti alla 22, dove il default non era OnPush — così chi copia gli snippet in un progetto più vecchio ottiene lo stesso comportamento.

**In sintesi:**
- La CLI genera una struttura professionale con un solo `ng new` (compiler, test, build, ottimizzazioni) e con `ng serve -o` avvii il dev server con live reload.
- L'app è un **albero di componenti** con root `App`, bootstrappato da `main.ts` tramite `bootstrapApplication(App, appConfig)`; i servizi globali stanno nei `providers` di `appConfig`.
- I **signal** sono onnipresenti fin dal componente generato: si leggono chiamandoli (`title()`), si aggiornano con `.set()`, si dichiarano `readonly`.
- Per lo studio: configura gli schematics in `angular.json` (`skipTests`, `style: none`, `OnPush`), abilita il linter con `ng lint` e produci il bundle con `ng build`. **OnPush è il default da Angular 22.**
