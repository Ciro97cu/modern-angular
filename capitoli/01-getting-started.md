---
capitolo: 1
titolo: "Getting Started with Angular"
pagine: "18-31"
tags: [tipo/capitolo, tooling, signals, angular-22]
---
# 01 · Getting Started with Angular
> 📖 cap.1 · pp.18-31 — *Modern Angular* v2.0.0

Setup dell'ambiente, generazione del progetto con la **Angular CLI** e prima lettura del codice generato. Il progetto di esempio del libro è **flights42** (`angular-architects/flights42` su GitHub).

## Tooling
> 📖 pp.18-20

- **IDE**: VS Code (con l'extension pack *Angular Essentials* di John Papa → include Angular Language Service ed ESLint) o WebStorm/IntelliJ.
- **Node.js**: usare le versioni **LTS**; con più versioni in parallelo, gestirle con **NVM**.
- **Angular CLI**: tool ufficiale per generare/buildare/testare. Installazione globale:

```bash
npm install -g @angular/cli
```

- **Progetto di esempio**:

```bash
git clone https://github.com/angular-architects/flights42.git
cd flights42
npm install
ng serve -o   # -o apre il browser
```

> [!tip] Take-away
> Il repo di esempio ha **branch per capitolo** (vedi `readme.md`): utili per seguire passo-passo.

## Generare e avviare un progetto
> 📖 pp.20-22

```bash
ng new flights42     # genera struttura, TS compiler, test, build tools
cd flights42
ng serve -o          # dev server su http://localhost:4200
ng serve -o --port 4242   # porta custom
```

`ng serve` ricompila e ricarica il browser ad ogni modifica (live reload).

> [!warning] Gotcha
> Apri in IDE la **cartella root del progetto** (quella con `angular.json`), altrimenti autocompletamento ed errori vanno in tilt. Se la CLI "perde" una modifica (salvataggi rapidi, rinomine file), risalva o riavvia `ng serve`.

## Struttura del progetto
> 📖 pp.22-23

L'app è un **albero di componenti** con un root component `App`. File chiave:

| File | Ruolo |
|---|---|
| `src/app/app.ts` | Logica del root component |
| `src/app/app.html` | Template del root component |
| `src/main.ts` | Entry point: fa il **bootstrap** del root component |
| `src/index.html` | Start page (la build vi inietta i bundle) |
| `src/styles.css` | Stili globali |
| `package.json` | Dipendenze e versioni |
| `angular.json` | Configurazione della CLI |
| `tsconfig.json` | Configurazione del compilatore TS |

## Il root component & i signal
> 📖 pp.23-25

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

  protected updateTitle(): void {
    this.title.set("Highly Sophisticated Flight App");
    console.log("Title updated", this.title());
  }
}
```

- `title` è un [[signal]] (`WritableSignal<string>`, tipo inferito dal default): lo **leggi chiamandolo** `title()`, lo aggiorni con `.set()`.
- Per Style Guide: proprietà usate solo nel template → `protected`; i signal → `readonly` (non si rimpiazzano, si aggiornano).
- `@Component` è un **decorator** che marca la classe come componente; `imports` elenca ciò che il template usa (es. `RouterOutlet`).

Template con due binding (interpolation + event):

```html
<!-- src/app/app.html -->
<h1>Hello, {{ title() }}</h1>
<button (click)="updateTitle()">Update Title</button>
```

Collegamenti: [[signal]] · approfondimenti binding in [[02-signal-based-components]].

## Bootstrap dell'applicazione
> 📖 pp.25-27

```ts
// src/main.ts
import { bootstrapApplication } from "@angular/platform-browser";
import { appConfig } from "./app/app.config";
import { App } from "./app/app";

bootstrapApplication(App, appConfig).catch((err) => console.error(err));
```

```ts
// src/app/app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [provideBrowserGlobalErrorListeners(), provideRouter(routes)],
};
```

`bootstrapApplication(rootComponent, config)`: la config registra i **servizi globali** via [[providers|provider functions]] (`provideRouter`, error listener, ecc.). `index.html` contiene `<app-root></app-root>` come punto di innesto.

## CLI: pacchetti, config e build
> 📖 pp.27-30

- **Aggiungere pacchetti**: `npm i <pkg>` oppure `ng add <pkg>` (installa **e** configura, es. `ng add @angular/material`, `npm install @ngrx/signals`).
- **Configurare gli schematics** in `angular.json` per ridurre il rumore in fase di studio e usare **OnPush**:

```jsonc
"schematics": {
  "@schematics/angular:component": {
    "style": "none",
    "skipTests": true,
    "changeDetection": "OnPush"
  },
  "@schematics/angular:directive": { "skipTests": true },
  "@schematics/angular:pipe":      { "skipTests": true },
  "@schematics/angular:service":   { "skipTests": true }
}
```

- **Linter**: `ng lint` (ESLint, config in `eslint.config.js`).
- **Build di produzione**: `ng build` → output in `dist/<app>/browser` (minify + tree-shaking).

> [!tip] Take-away
> **OnPush** è la strategia di change detection raccomandata e i signal la rendono naturale: Angular aggiorna un componente solo quando i suoi dati cambiano. **Da Angular 22 è il default.** Il libro continua comunque a scrivere `changeDetection: ChangeDetectionStrategy.OnPush` esplicitamente in ogni `@Component`: rende la scelta visibile e resta retrocompatibile con versioni < 22.

## 🔁 Ripasso lampo
1. Differenza tra `npm i <pkg>` e `ng add <pkg>`?
2. Come leggi e come aggiorni il valore di un `signal`? Perché si dichiara `readonly`?
3. Cosa fa `bootstrapApplication` e dove registri i servizi globali?
4. A cosa serve `OnPush` e perché i signal lo rendono conveniente?
5. Quali file devi guardare per capire root component, bootstrap e start page?

**Take-away del capitolo:**
- La CLI genera una struttura professionale con un solo `ng new` (compiler, test, build, ottimizzazioni).
- L'app è un **albero di componenti** con root `App` bootstrappato da `main.ts` con `appConfig`.
- I **signal** sono già onnipresenti dal componente generato; `OnPush` + signal = change detection efficiente.
