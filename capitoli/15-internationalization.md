---
capitolo: 15
titolo: "Internationalization"
pagine: "400-411"
tags: [tipo/capitolo, i18n]
---
# 15 · Internationalization
> 📖 cap.15 · pp.400-411 — *Modern Angular* v2.0.0

Adattare un'app a regioni e lingue diverse va previsto **in fase di implementazione**: testi intercambiabili, ma anche formati di data e numero. Questo è l'obiettivo dell'**internationalization** (abbreviata **I18N**). L'adattamento alle singole lingue/regioni che ne consegue si chiama **localization** (**L10N**). La combinazione di regione + lingua è il **locale**: `de-DE` (tedesco standard, Germania), `de-AT` (tedesco austriaco); i **locale generici** ignorano la regione (es. `de`).

La soluzione I18N integrata in Angular è in primo luogo una soluzione **compile-time**: il compiler estrae i testi, li sostituisce con le traduzioni e produce **un set di bundle per ogni lingua**. Vantaggio: nessun costo a runtime per caricare o mostrare i testi (sono parte integrante del bundle). Prezzo: devi distribuire più versioni linguistiche e cambiare lingua significa **caricare una nuova app** via hyperlink, perdendo lo stato di quella precedente.

## Overview
> 📖 pp.400-401

Il compiler genera file in uno dei formati standard usati dai software degli studi di traduzione: **XLIFF** (XML Localization Interchange File Format) v1 e v2, **XMB** (XML Message Bundles) e, ora, anche i formati basati su JSON come **ARB** (Application Resource Bundle). Dopo la traduzione di questi file, il compiler ne integra i testi nei bundle generati.

> [!tip]
> Originariamente la CLI rieseguiva l'intero processo di build per ogni lingua → build lentissimi. Dalla v9 è più "smart": fa **un solo build** inserendo dei placeholder per tutti i testi da sostituire, poi **copia il set di bundle per ogni lingua** e rimpiazza i placeholder con le rispettive traduzioni.

## Installing @angular/localize
> 📖 p.401

Anche se la I18N del compiler è parte integrante del compiler stesso, servono tool aggiuntivi per la CLI e per il runtime, forniti dal pacchetto `@angular/localize`:

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
- **ID** (dopo `@@`) → il compiler lo usa per ricordare dove ricollocare ogni testo. Se non lo specifichi viene generato, ma l'ID auto-generato **può cambiare quando il template cambia** → assegnalo sempre a mano.

Essendo tutte e tre opzionali, sono lecite varie combinazioni:

```html
<h1 i18n="description">Flight Search</h1>
<h1 i18n="description@@flightSearch-title">Flight Search</h1>
<h1 i18n="@@flightSearch-title">Flight Search</h1>
<h1 i18n>Flight Search</h1>
```

Si possono includere **data binding** nei testi marcati:

```html
<!-- src/app/flight-search/flight-search.html -->
<div class="form-group" *ngIf="flights$ | async as flights">
  <div i18n="@@flightSearch-flightsFound">
    {{ flights.length }} flights found!
  </div>
</div>
```

> [!tip]
> Anche i **singoli attributi** HTML si traducono, combinando `i18n` col nome dell'attributo:
> ```html
> <img src="img.png" alt="Super Bild" i18n-src="@@bildname" i18n-alt="@@alt-text">
> ```

## Marking Strings in the Component Class ($localize)
> 📖 pp.402-403

Inizialmente la I18N del compiler era limitata ai testi nei template, ma per alcuni casi d'uso è troppo riduttivo: a volte i messaggi (info/errore) nascono nella **component class** e vengono mostrati via data binding. Dalla v9 si possono adattare alle varie lingue con `$localize`, anche se con una sintassi insolita:

```ts
@Component({ /* ... */ })
export class FlightSearchComponent {
  // $localize NON va importato: @angular/localize lo registra come funzione globale.
  info = $localize`:meaning|description@@flightSearch-info:Hello World!`;

  constructor() {
    console.debug('info', this.info);
  }
}
```

È una **tagged string**: una stringa racchiusa in backtick e preceduta dalla funzione tag `$localize`, che la adatta. Il contenuto inizia con i metadati (racchiusi tra **due `:`**), poi viene il valore di default. La sintassi è questa perché il compiler sostituisce i testi direttamente nei bundle per performance → non si possono caricare a runtime quando servono.

## Extracting Texts (extract-i18n)
> 📖 pp.403-405

Marcati i testi, li estrai col compiler. Di default la CLI genera un file `messages.xlf` in formato XLIFF; con `--format` ne richiedi un altro tra quelli supportati:

```bash
ng extract-i18n                 # default: messages.xlf (XLIFF)
ng extract-i18n --format xmb
ng extract-i18n --format arb
```

I file finiscono nella **root del progetto**. Per ordine si crea una sottocartella `compiler-i18n` e ci si sposta `messages.xlf`; lo si **duplica** dentro la stessa cartella rinominando il duplicato `messages.de.xlf` (uno per lingua). Ogni testo estratto è una **`trans-unit`**:

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
      <trans-unit id="flightSearch-title" datatype="html">
        <source>Flight Search</source>
        <target>Flüge suchen</target>            <!-- da aggiungere a mano -->
        <context-group purpose="location">
          <context context-type="sourcefile">[...]/flight-search.html</context>
          <context context-type="linenumber">3</context>
        </context-group>
        <note priority="1" from="description">Description</note>
        <note priority="1" from="meaning">Meaning</note>
      </trans-unit>
      <trans-unit id="flightSearch-flightsFound" datatype="html">
        <source><x id="INTERPOLATION" equiv-text="..."/> flights found!</source>
        <target><x id="INTERPOLATION" equiv-text="..."/> Flüge gefunden!</target>
        <context-group purpose="location">
          <context context-type="sourcefile">[...]/flight-search.html</context>
          <context context-type="linenumber">67</context>
        </context-group>
      </trans-unit>
    </body>
  </file>
</xliff>
```

Oltre ai metadati (ID/description/meaning) c'è il testo originale nel nodo `<source>`: basta aggiungere a ciascuna `trans-unit` un nodo `<target>` con la traduzione. Qui lo facciamo a mano; in pratica lo fanno gli studi di traduzione coi loro tool.

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
- `sourceLocale` → la lingua in cui sono i template quando **non sono tradotti** (qui `en-US`).

Poi si buildano le versioni per ogni lingua:

```bash
ng build --localize
```

Nella `dist` compare **una sottocartella per locale** (es. `de`, `en-US`), ognuna con l'intera app tradotta. Si possono provare con un web server da riga di comando, es. il pacchetto npm `serve` (`npm i -g serve`, poi `serve` dentro la `dist` che contiene le due sottocartelle): il listing mostra le cartelle delle lingue, cliccabili per provare la versione desiderata.

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

...e una per `ng serve` che, non potendo impostare da sé la lingua, **referenzia** quella di build via `buildTarget` (`<app>:build:<config>`):

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

> [!warning]
> Per ogni lingua extra serve una **coppia di configurazioni** (build + serve). Gli stessi autori ammettono che l'approccio è un po' macchinoso.

## Specifying Translation Texts at Runtime (loadTranslations)
> 📖 p.408

Anche se la I18N del compiler integra i testi a compile-time, si possono impostare via codice con `loadTranslations` — ma **prima del primo uso** di ciascun testo. Non essendoci data binding per questi testi, cambiarli dopo il primo uso è impossibile senza un reload dell'app nel browser.

```ts
// src/main.ts
import { loadTranslations } from '@angular/localize';

loadTranslations({
  'flightSearch-info': 'From main.ts with love!',
  'flightSearch-title': 'Flight Search!!',
  'flightSearch-flightsFound': '{$INTERPOLATION} flights found!',
});
```

Il placeholder `{$INTERPOLATION}` rappresenta il numero di voli che il template imposta via data binding. Per scoprire i nomi dei placeholder, conviene estrarre temporaneamente i testi marcati con `i18n` in formato JSON:

```bash
ng extract-i18n --format json   # genera messages.json con le espressioni
```

> [!warning]
> Puoi caricare i testi di `loadTranslations` anche via HTTP (es. `HttpClient`), ma **devono essere disponibili prima del primo uso**. Essendo le richieste HTTP asincrone, non è facile garantirlo: usa un [[glossario#resolver|resolver]] che ritarda il routing (cfr. cap. 8).

## Taking Grammatical Forms into Account (ICU: plural / select)
> 📖 pp.408-410

Scambiare i testi è solo uno dei compiti dell'internazionalizzazione: vanno gestite anche le **forme grammaticali** come il genere e la distinzione singolare/plurale (lingue diverse hanno numeri diversi di generi e di forme plurali). Il compiler I18N offre una grammatica dedicata nel template (sintassi ICU — Unicode "International Components for Unicode", lo standard per esprimere plurali e generi nei messaggi tradotti).

**Plural** — "1 flight" vs "3 flights":

```html
<!-- src/app/flight-search/flight-search.html -->
<div i18n="@@flightSearch-flightsFound">
  {flights.length, plural, =1 {1 flight found} other {# flights found}}
</div>
```

Si definisce un testo per il valore `=1` e uno per tutti gli altri (`other`); il `#` riporta il valore. La CLI e il compiler estraggono l'**intera espressione** nei file di lingua, così tradurre verso una lingua con più forme plurali permette di specificare testi anche per ulteriori valori (es. da 2 a 5). Oltre ai valori concreti (`=1`), Angular supporta le categorie: `zero`, `one`, `two`, `few`, `many`, `other`.

**Select** — genere grammaticale. Si estende il `PassengerSearchComponent` con una proprietà `passenger`:

```ts
// src/app/flight-booking/passenger-search/passenger-search.ts
@Component(/* ... */)
export class PassengerSearchComponent {
  passenger = {
    name: 'Max',
    gender: 'male',
  };
}
```

```html
<!-- src/app/flight-booking/passenger-search/passenger-search.html -->
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

Vanno rispettate anche le convenzioni locali per **numeri e date**. Angular include i metadati di tutti i locale dallo **Unicode CLDR** (Common Locale Data Repository): durante il build la CLI li aggiunge alle versioni di lingua. Le pipe `date` e `number` li leggono a runtime.

Perché funzioni, alla pipe `date` vanno passati **formati logici** (`short`, `medium`, `long`, ...):

```html
<!-- src/app/flight-booking/flight-card/flight-card.html -->
<p>Date: {{ item?.date | date:'short' }}</p>
```

La pipe `number` non richiede aggiustamenti: usa automaticamente i separatori delle migliaia e dei decimali del locale corrente. Per le versioni buildate con `ng build --localize` non serve altro per ottenere il formato corretto.

> [!warning]
> Le pipe servono **solo per l'output**. Per localizzare l'**input** servono controlli di terze parti (es. il datepicker gratuito di Angular Material) o un `ControlValueAccessor` custom (il pezzo di codice che fa da ponte tra un controllo di form Angular e il widget HTML, traducendo avanti e indietro il valore) che supporti il formato di input desiderato (cfr. cap. 18).

Collegamenti: pipe `date`/`number` locale-aware viste in [[02-signal-based-components]].

## Outlook: Community Solutions (ngx-translate, Transloco)
> 📖 p.411

La I18N del compiler eccelle in **performance** (testi intessuti nei bundle, zero costo runtime per load e data binding), ma paga in **flessibilità**: la rinuncia deliberata al data binding significa **niente cambio lingua a runtime**, solo hyperlink ad altre versioni di lingua. Qui si perdono i vantaggi delle SPA (Single Page Application — l'app gira in un'unica pagina senza ricaricarsi a ogni navigazione, mantenendo lo stato).

Le soluzioni community **ngx-translate** e **Transloco** **invertono** vantaggi e svantaggi: permettono load e switch di lingua **a runtime**, ma si appoggiano al data binding, che (teoricamente) può incidere sulla performance.

> [!tip]
> Compiler I18N = performance, un set di bundle per lingua, switch via hyperlink (si perdono i vantaggi della SPA). ngx-translate/Transloco = flessibilità, switch a runtime, al costo del caricamento dei testi e del data binding.

## 🔁 Ripasso lampo

**1.** Perché la I18N del compiler genera un set di bundle separato per ogni lingua, e che prezzo si paga a runtime?
> [!success]- Risposta
> Perché il compiler sostituisce i testi tradotti **direttamente nei bundle a compile-time**: ogni lingua ottiene il proprio set completo. Vantaggio: a runtime **nessun costo** per caricare o mostrare i testi (sono parte del bundle). Prezzo: devi distribuire più versioni e il cambio lingua avviene caricando **una nuova app** via hyperlink → si perde lo stato della SPA precedente.

**2.** Cosa rappresentano le tre parti del valore di `i18n` (`meaning|description@@id`) e perché conviene assegnare l'ID a mano?
> [!success]- Risposta
> **meaning** e **description** danno contesto allo studio di traduzione (utili quando una parola ha più traduzioni). L'**ID** (dopo `@@`) serve al compiler per ricollocare il testo tradotto. Conviene assegnarlo a mano perché l'ID auto-generato **può cambiare quando il template cambia**, e allora il compiler fatica ad abbinare la traduzione.

**3.** Come si traduce una stringa creata nella component class? Va importato `$localize`?
> [!success]- Risposta
> Con una **tagged string** `$localize`: backtick preceduti dalla funzione tag, contenuto che inizia con i metadati tra **due `:`** seguiti dal valore di default, es. `` $localize`:meaning|description@@id:Hello World!` ``. `$localize` **non va importato**: `@angular/localize` lo registra come funzione globale.

**4.** Qual è il flusso `extract-i18n` → traduzione → `ng build --localize`? Dove si registrano i file in `angular.json`?
> [!success]- Risposta
> 1) `ng extract-i18n` genera `messages.xlf` (XLIFF) con una `trans-unit` per ogni testo. 2) Si duplica per lingua (`messages.de.xlf`) e si aggiunge un nodo `<target>` con la traduzione in ogni `trans-unit`. 3) Si registrano i file nel nodo `i18n` di `angular.json`, sotto `projects/<nome>`: `locales` mappa ogni locale al suo file, `sourceLocale` indica la lingua dei template non tradotti. 4) `ng build --localize` produce una sottocartella per locale nella `dist`.

**5.** Come si fa partire `ng serve` in una lingua specifica, dato che manca lo switch nativo?
> [!success]- Risposta
> Servono due configurazioni in `angular.json`: una sotto `build` che imposta la lingua (`"localize": ["de"]`) e una sotto `serve` che la referenzia via `buildTarget` (`flight-app:build:de`). Poi si lancia `ng serve --configuration de`. Per ogni lingua extra serve una nuova coppia di configurazioni.

**6.** A cosa servono `plural` e `select` (ICU)? Quali categorie plurali esistono?
> [!success]- Risposta
> Sono la grammatica ICU per le **forme grammaticali**. `plural` distingue singolare/plurale (es. `=1 {1 flight found} other {# flights found}}`, con `#` che riporta il valore); `select` sceglie il testo in base a un valore discreto come il genere (`male`/`female`/`other`). Le categorie plurali supportate sono `zero`, `one`, `two`, `few`, `many`, `other`, oltre ai valori concreti come `=1`.

**7.** Differenza chiave tra compiler I18N e soluzioni come ngx-translate/Transloco?
> [!success]- Risposta
> **Compiler I18N**: massima performance (testi tessuti nei bundle, niente data binding), ma niente cambio lingua a runtime → solo hyperlink ad altre versioni. **ngx-translate/Transloco**: invertono il trade-off → load e switch di lingua a runtime grazie al data binding, al costo di un possibile impatto sulla performance.

**In sintesi:**
- La I18N integrata è **compile-time**: marca i testi con `i18n` (template) o `$localize` (classe), estrai con `ng extract-i18n`, traduci i `<target>`, registra i locale in `angular.json` e builda con `ng build --localize`.
- In dev si seleziona la lingua con una coppia di configurazioni build+serve (`buildTarget`); a runtime si possono iniettare testi con `loadTranslations`, ma solo prima del primo uso.
- Forme grammaticali via ICU (`plural`, `select`); numeri/date via pipe locale-aware (CLDR) — solo per l'output, non per l'input.
- Trade-off: ottima performance ma niente switch lingua a runtime; per quello servono **ngx-translate** o **Transloco**.
