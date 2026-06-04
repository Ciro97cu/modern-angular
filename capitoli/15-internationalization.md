---
capitolo: 15
titolo: "Internationalization"
pagine: "400-411"
tags: [tipo/capitolo, i18n]
---
# 15 · Internationalization
> 📖 cap.15 · pp.400-411 — *Modern Angular* v2.0.0

Adattare un'app a regioni e lingue diverse va previsto **in fase di implementazione**: testi intercambiabili, ma anche formati di data e numero. Questo è l'obiettivo dell'**internationalization** (abbreviata **I18N**). L'adattamento alle singole lingue/regioni che ne consegue si chiama **localization** (**L10N**). La combinazione di regione + lingua è il **locale**: `de-DE` (tedesco standard, Germania), `de-AT` (tedesco austriaco); i **locale generici** ignorano la regione (es. `de`).

La soluzione I18N integrata in Angular è **compile-time**: il compiler estrae i testi, li sostituisce con le traduzioni e produce **un set di bundle per ogni lingua**. Vantaggio: nessun costo a runtime per caricare/mostrare i testi (sono parte del bundle). Prezzo: devi distribuire più versioni e cambiare lingua significa **caricare una nuova app** via hyperlink, perdendo lo stato di quella precedente.

## Overview
> 📖 pp.400-401

Il compiler genera file in formati standard usati dagli studi di traduzione: **XLIFF** (XML Localization Interchange File Format) v1 e v2, **XMB** (XML Message Bundles) e i formati JSON come **ARB** (Application Resource Bundle). Dopo la traduzione, il compiler integra i testi nei bundle generati.

> [!tip] Take-away
> Dalla v9 la CLI è "smart": fa **un solo build** con dei placeholder per tutti i testi, poi **copia il set di bundle per ogni lingua** e sostituisce i placeholder con le traduzioni. Prima rieseguiva l'intero build per ogni lingua → build lentissimi.

## Installing @angular/localize
> 📖 p.401

Anche se la I18N del compiler è parte integrante del compiler stesso, servono tool per CLI e runtime, forniti dal pacchetto `@angular/localize`:

```bash
ng add @angular/localize
```

## Marking Texts (i18n attribute)
> 📖 pp.401-402

Per far sapere al compiler quali testi estrarre, si marcano nei template con l'attributo `i18n`:

```html
<!-- src/app/flight-search/flight-search.html -->
<h1 i18n="meaning|description@@flightSearch-title">Flight Search</h1>
```

Il valore di `i18n` ha **tre parti opzionali**, con `|` e `@@` come separatori:
- **meaning** e **description** → contesto per lo studio di traduzione (utili quando una parola ha più traduzioni possibili).
- **ID** (dopo `@@`) → il compiler lo usa per ricollocare il testo. Se non lo specifichi viene generato, ma l'ID auto-generato **può cambiare quando il template cambia** → assegnalo sempre a mano.

Tutte le combinazioni sono lecite:

```html
<h1 i18n="description">Flight Search</h1>
<h1 i18n="description@@flightSearch-title">Flight Search</h1>
<h1 i18n="@@flightSearch-title">Flight Search</h1>
<h1 i18n>Flight Search</h1>
```

Si possono includere **data binding** nei testi marcati:

```html
<div i18n="@@flightSearch-flightsFound">
  {{ flights.length }} flights found!
</div>
```

> [!tip] Take-away
> Anche i **singoli attributi** HTML si traducono, combinando `i18n` col nome dell'attributo:
> ```html
> <img src="img.png" alt="Super Bild" i18n-src="@@bildname" i18n-alt="@@alt-text">
> ```

## Marking Strings in the Component Class ($localize)
> 📖 pp.402-403

A volte i messaggi (info/errore) nascono nella **component class**. Dalla v9 si possono adattare con `$localize`, con una sintassi insolita:

```ts
@Component({ /* ... */ })
export class FlightSearchComponent {
  // $localize NON va importato: è una funzione globale impostata da @angular/localize
  info = $localize`:meaning|description@@flightSearch-info:Hello World!`;

  constructor() {
    console.debug('info', this.info);
  }
}
```

È una **tagged string**: backtick + funzione tag (`$localize`). Il contenuto inizia con i metadati (tra **due `:`**), poi il valore di default. La sintassi è così perché il compiler sostituisce i testi direttamente nei bundle per performance → non si possono caricare a runtime quando servono.

## Extracting Texts (extract-i18n)
> 📖 pp.403-405

Marcati i testi, li estrai col compiler:

```bash
ng extract-i18n                 # default: messages.xlf (XLIFF)
ng extract-i18n --format xmb
ng extract-i18n --format arb
```

I file finiscono nella **root del progetto**. Per ordine si crea una sottocartella `compiler-i18n` e ci si sposta `messages.xlf`; lo si **duplica** in `messages.de.xlf` (uno per lingua). Ogni testo estratto è una **`trans-unit`**:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<xliff version="1.2" xmlns="urn:oasis:names:tc:xliff:document:1.2">
  <file source-language="en-US" datatype="plaintext" original="ng2.template">
    <body>
      <trans-unit id="flightSearch-info" datatype="html">
        <source>Hello World!</source>
        <target>Hallo Welt!</target>            <!-- da aggiungere a mano -->
        <context-group purpose="location">
          <context context-type="sourcefile">[...]/flight-search.ts</context>
          <context context-type="linenumber">18</context>
        </context-group>
        <note priority="1" from="description">description</note>
        <note priority="1" from="meaning">meaning</note>
      </trans-unit>
      <trans-unit id="flightSearch-flightsFound" datatype="html">
        <source><x id="INTERPOLATION" equiv-text="..."/> flights found!</source>
        <target><x id="INTERPOLATION" equiv-text="..."/> Flüge gefunden!</target>
      </trans-unit>
    </body>
  </file>
</xliff>
```

Oltre ai metadati (ID/description/meaning) c'è il testo originale nel nodo `<source>`: basta aggiungere un `<target>` con la traduzione. Qui lo facciamo a mano; in pratica lo fanno gli studi di traduzione coi loro tool.

## Integrating Translated Texts into Builds (angular.json)
> 📖 pp.405-406

I file di traduzione si registrano in `angular.json`, con un nodo `i18n` sotto `projects/<nome>`:

```jsonc
{
  "projects": {
    "flight-app": {
      "projectType": "application",
      "root": "",
      "sourceRoot": "src",
      "prefix": "app",
      "i18n": {
        "locales": {
          "de": "compiler-i18n/messages.de.xlf"
        },
        "sourceLocale": "en-US"
      }
    }
  }
}
```

- `locales` → mappa ogni locale supportato al suo file di traduzione.
- `sourceLocale` → la lingua dei template **non tradotti** (qui `en-US`).

Poi si buildano le versioni per ogni lingua:

```bash
ng build --localize
```

Nella `dist` compare **una sottocartella per locale** (es. `de`, `en-US`), ognuna con l'intera app tradotta. Si possono provare con un web server da riga di comando, es. il pacchetto npm `serve` (`npm i -g serve` poi `serve` dentro `dist`): il listing mostra le cartelle delle lingue, cliccabili.

Collegamenti: build e packaging approfonditi in [[14-monorepos-libraries]].

## Setting the Language for Development (ng serve)
> 📖 pp.406-407

`ng serve` **non ha uno switch per la lingua**. Si aggira con due configurazioni in `angular.json`: una per `ng build` che imposta la lingua...

```jsonc
{
  "projects": {
    "flight-app": {
      "architect": {
        "build": {
          "configurations": {
            "de": {
              "localize": ["de"]
            }
          }
        }
      }
    }
  }
}
```

...e una per `ng serve` che **referenzia** quella di build via `buildTarget` (`<app>:build:<config>`):

```jsonc
{
  "projects": {
    "flight-app": {
      "architect": {
        "serve": {
          "configurations": {
            "de": {
              "buildTarget": "flight-app:build:de"
            }
          }
        }
      }
    }
  }
}
```

```bash
ng serve --configuration de
```

> [!warning] Gotcha
> Per ogni lingua extra serve una **coppia di configurazioni** (build + serve). Gli stessi autori ammettono che l'approccio è un po' macchinoso.

## Specifying Translation Texts at Runtime (loadTranslations)
> 📖 p.408

Anche se la I18N del compiler integra i testi a compile-time, si possono impostare via codice con `loadTranslations` — ma **prima del primo uso** di ciascun testo (non essendoci data binding, dopo il primo uso non cambiano più senza un reload):

```ts
// src/main.ts
import { loadTranslations } from '@angular/localize';

loadTranslations({
  'flightSearch-info': 'From main.ts with love!',
  'flightSearch-title': 'Flight Search!!',
  'flightSearch-flightsFound': '{$INTERPOLATION} flights found!',
});
```

Il placeholder `{$INTERPOLATION}` rappresenta il valore iniettato dal data binding del template. Per scoprire i nomi dei placeholder, estrai temporaneamente in JSON:

```bash
ng extract-i18n --format json   # genera messages.json con le espressioni
```

> [!warning] Gotcha
> Puoi caricare i testi di `loadTranslations` anche via HTTP (es. `HttpClient`), ma **devono essere disponibili prima del primo uso**. Essendo le richieste HTTP asincrone, non è garantito: usa un **resolver** che ritarda il routing (cfr. cap. sui resolver).

## Taking Grammatical Forms into Account (ICU: plural / select)
> 📖 pp.408-410

Oltre ai testi, vanno gestite le **forme grammaticali**: genere, singolare/plurale (lingue diverse hanno numeri diversi di generi e forme plurali). Il compiler I18N offre una grammatica dedicata nel template (sintassi ICU).

**Plural** — "1 flight" vs "3 flights":

```html
<div i18n="@@flightSearch-flightsFound">
  {flights.length, plural, =1 {1 flight found} other {# flights found}}
</div>
```

Si definisce un testo per `=1` e uno per tutti gli altri (`other`); il `#` riporta il valore. L'intera espressione finisce nei file di lingua, così si possono specificare testi per ulteriori valori (es. 2–5) nelle lingue con più forme plurali. Categorie supportate: `zero`, `one`, `two`, `few`, `many`, `other` (oltre ai valori concreti come `=1`).

**Select** — genere grammaticale:

```ts
// passenger-search.ts
export class PassengerSearchComponent {
  passenger = { name: 'Max', gender: 'male' };
}
```

```html
<p i18n="@@passenger-bookedTicket">
  You've booked a ticket for {{ passenger.name }}
</p>
<p i18n="@@passenger-forward">
  {
    passenger.gender, select,
    male {Forward it to him.}
    female {Forward it to her.}
    other {Forward it to them.}
  }
</p>
```

## Supporting Different Formats (locale-aware pipes)
> 📖 pp.410-411

Vanno rispettate anche le convenzioni locali per **numeri e date**. Angular include i metadati di tutti i locale dal **Unicode CLDR** (Common Locale Data Repository): durante il build la CLI li aggiunge alle versioni di lingua. Le pipe `date` e `number` li leggono a runtime.

Per la pipe `date` servono **formati logici** (`short`, `medium`, `long`, ...):

```html
<p>Date: {{ item?.date | date:'short' }}</p>
```

La pipe `number` non richiede aggiustamenti: usa automaticamente separatori delle migliaia/decimali del locale corrente. Nulla d'altro serve per le versioni buildate con `ng build --localize`.

> [!warning] Gotcha
> Le pipe servono **solo per l'output**. Per localizzare l'**input** servono controlli di terze parti (es. il datepicker di Angular Material) o un `ControlValueAccessor` custom.

Collegamenti: pipe `date`/`number` locale-aware viste in [[02-signal-based-components]].

## Outlook: Community Solutions (ngx-translate, Transloco)
> 📖 p.411

La I18N del compiler eccelle in **performance** (testi intessuti nei bundle, zero costo runtime per load/data binding), ma paga in **flessibilità**: niente data binding → **niente cambio lingua a runtime**, solo hyperlink ad altre versioni di lingua.

Le soluzioni community **ngx-translate** e **Transloco** **invertono** vantaggi/svantaggi: permettono load e switch di lingua **a runtime**, ma si appoggiano al data binding, che (teoricamente) può incidere sulla performance.

> [!tip] Take-away
> Compiler I18N = performance, un bundle set per lingua, switch via hyperlink (si perdono i vantaggi della SPA). ngx-translate/Transloco = flessibilità, switch a runtime, costo di caricamento + data binding.

## 🔁 Ripasso lampo
1. Perché la I18N del compiler genera un set di bundle separato per ogni lingua, e che prezzo si paga a runtime?
2. Cosa rappresentano le tre parti del valore di `i18n` (`meaning|description@@id`) e perché conviene assegnare l'ID a mano?
3. Come si traduce una stringa creata nella component class? Va importato `$localize`?
4. Qual è il flusso `extract-i18n` → traduzione → `ng build --localize`? Dove si registrano i file in `angular.json`?
5. Come si fa partire `ng serve` in una lingua specifica, dato che manca lo switch nativo?
6. A cosa servono `plural` e `select` (ICU)? Quali categorie plurali esistono?
7. Differenza chiave tra compiler I18N e soluzioni come ngx-translate/Transloco?

**Take-away del capitolo:**
- La I18N integrata è **compile-time**: marca i testi con `i18n` (template) o `$localize` (classe), estrai con `ng extract-i18n`, traduci i `<target>`, registra i locale in `angular.json` e builda con `ng build --localize`.
- In dev si seleziona la lingua con una coppia di configurazioni build+serve (`buildTarget`); a runtime si possono iniettare testi con `loadTranslations` ma solo prima del primo uso.
- Forme grammaticali via ICU (`plural`, `select`); numeri/date via pipe locale-aware (CLDR) — solo per l'output.
- Trade-off: ottima performance ma niente switch lingua a runtime; per quello servono **ngx-translate** o **Transloco**.
