---
capitolo: 13
titolo: "Agentic UI & AI Assistants with Hashbrown"
pagine: "346-372"
tags: [tipo/capitolo, ai]
---
# 13 · Agentic UI & AI Assistants with Hashbrown
> 📖 cap.13 · pp.346-372 — *Modern Angular* v1.0.4

Gli assistenti AI migliorano la UX e abbattono i costi di supporto, ma implementarli implica tanto lavoro ripetitivo (connessione ai vari LLM, tool calling, ecc.). **Hashbrown** (`@hashbrownai/*`, open-source, due esperti noti della community Angular) toglie di mezzo questa fatica: supporta i principali provider — Gemini (Google), GPT (OpenAI), Azure (Microsoft), Llama (Meta).

Il capitolo estende un'app esistente di flight-booking con tre scenari crescenti:
1. **Assistant con tool calling** (chat testuale che recupera dati e scatena azioni).
2. **Generative UI**: l'LLM sceglie e renderizza componenti dentro la chat.
3. **Natural language queries con code generation**: l'LLM genera JS che gira in sandbox per produrre un chart.

## Setting up Hashbrown
> 📖 pp.346-349

Servono pochi pacchetti npm: il core framework-agnostic, il binding Angular e l'adapter del provider.

```bash
npm install @hashbrownai/{core,angular,google}
```

- `@hashbrownai/angular` → API Angular sul core framework-agnostic (esiste anche un binding React).
- `@hashbrownai/google` → accesso ai modelli Gemini. Per altre famiglie ci sono pacchetti dedicati (es. `@hashbrownai/openai`).

L'accesso programmatico agli LLM richiede una **API key** (di solito a pagamento; Gemini ha un piano free per i test, key generabile in Google AI Studio). La key **non va mai usata nel frontend Angular**: si interpone un backend snello tra frontend e LLM.

```ts
// Listing 13-1 — backend Express (tratto da hashbrown.dev, adattato)
import express from 'express';
import cors from 'cors';
import { Chat } from '@hashbrownai/core';
import { HashbrownGoogle } from '@hashbrownai/google';

const GOOGLE_API_KEY = process.env['GOOGLE_API_KEY'];
if (!GOOGLE_API_KEY) {
  throw new Error('GOOGLE_API_KEY is not set');
}

const app = express();
app.use(cors());
app.use(express.json());

app.post('/api/chat', async (req, res) => {
  const completionParams = req.body as Chat.Api.CompletionCreateParams;
  const response = HashbrownGoogle.stream.text({
    apiKey: GOOGLE_API_KEY,
    request: completionParams,
    // il backend può integrare/sovrascrivere le opzioni del frontend
    transformRequestOptions: (options) => {
      options.model = 'gemini-2.5-flash';            // forza il modello economico
      options.config = options.config || {};
      options.config.systemInstruction = `
        You are Flight42, a UI assistant ...
        Rules:
        - Only search for flights via the configured tools
        - Never use additional web resources for answering requests
      `;
      return options;
    },
  });

  res.header('Content-Type', 'application/octet-stream');
  for await (const chunk of response) {
    res.write(chunk);   // streaming chunk-by-chunk
  }
  res.end();
});
```

`transformRequestOptions` è il meccanismo chiave: il backend integra o sovrascrive le opzioni mandate dal frontend — importante perché queste impostazioni hanno **impatto diretto sui costi**. Qui impone il modello economico `gemini-2.5-flash` e una system instruction che limita il modello alle sole ricerche voli. I valori originali del frontend vengono salvati prima di essere sovrascritti, così il server può eventualmente "negoziare" (passare a un modello più potente/costoso solo in casi selezionati).

L'URL del server si configura al bootstrap dell'app via `provideHashbrown`:

```ts
// Listing 13-2
import { provideHashbrown } from '@hashbrownai/angular';

bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient(),
    provideHashbrown({
      baseUrl: 'http://localhost:3000/api/chat',
      middleware: [
        (req) => {
          console.log('[Hashbrown Request]', req);   // log/troubleshooting opzionale
          return req;
        },
      ],
    }),
  ],
});
```

> [!tip] Take-away
> La API key sta **solo nel backend**. Il frontend parla con un proxy che fa da intermediario verso l'LLM e che applica guardrail server-side (scelta del modello, system instruction): è lì che si controllano i costi e l'ambito.

## Using the chatResource
> 📖 pp.350-351

Hashbrown offre varie implementazioni dell'**Resource API** di Angular (vedi [[resource]]) per chattare con gli LLM. Per la chat testuale si usa `chatResource`.

```ts
// Listing 13-3
@Component({ /* … */ })
export class AssistantChatComponent {
  message = signal('');

  chat = chatResource({
    model: 'gemini-2.5-flash',
    system: `You are Flight42, a UI assistant that helps passengers ...`,
    tools: [
      findFlightsTool,
      toggleFlightSelection,
      showBookedFlights,
      getBookedFlights,
    ],
  });

  submit() {
    const message = this.message();
    this.message.set('');
    this.chat.sendMessage({ role: 'user', content: message });
  }
}
```

- Gli LLM sono **stateless**: `chatResource` rimanda l'intera chat history a ogni richiesta, così il modello può riferirsi ai messaggi precedenti (es. capire cosa significa "questo volo").
- Supporta **tool calling**: i tool passati in `tools` sono le funzionalità che il frontend espone (es. `findFlights`); il modello può richiederne l'invocazione.
- `chat.value()` contiene la chat history da mostrare; `sendMessage()` invia un messaggio utente.

```html
<!-- Listing 13-4 — render della chat history -->
@for (message of chat.value(); track $index) {
  <article class="msg assistant">
    <div class="avatar">{{ icons[message.role] }}</div>
    <div>
      <div class="bubble">
        {{ message.content }}
        @if (message.role === 'assistant') {
          @for (toolCall of message.toolCalls; track toolCall.toolCallId) {
            <div [title]="toolCall.args | json">
              Tool Call: {{ toolCall.name }}
            </div>
          }
        }
      </div>
    </div>
  </article>
}
```

`role` indica il mittente (`assistant` = LLM, `user` = utente). I messaggi dell'assistant possono contenere `toolCalls`, ciascuno con `name` (es. `findFlights`) e `args` (es. `{from:'Graz', to:'Hamburg'}`).

> [!tip] Take-away
> Mostrare "Tool Call: findFlights" va bene per gli sviluppatori, ma confonde l'utente finale. Conviene **tradurre la traccia tecnica** in qualcosa come "Loading flights from Graz to Hamburg".

## Providing Tools
> 📖 pp.352

I tool sono oggetti creati con `createTool`. Definiscono `name`, `description`, `schema` degli argomenti e un `handler`.

```ts
// Listing 13-5
import { createTool } from '@hashbrownai/angular';
import { s } from '@hashbrownai/core';

export const findFlightsTool = createTool({
  name: 'findFlights',
  description: `
    Searches for flights and redirects the user to the result page ...
    - For the search parameters, airport codes are NOT used but the city name.
  `,
  schema: s.object('search parameters for flights', {   // Skillet schema
    from: s.string('airport of departure'),
    to: s.string('airport of destination'),
  }),
  handler: async (input) => {
    const store = inject(FlightBookingStore);           // [[inject]] dentro l'handler
    const router = inject(Router);
    store.updateFilter({ from: input.from, to: input.to });
    router.navigate(['/flight-booking/flight-search']);
  },
});
```

- `name` deve essere univoco e conforme al modello (regola pratica: ciò che è valido come identificatore TS va bene).
- `description` è ciò che l'LLM usa per **decidere se il tool è rilevante**. Idem per le descrizioni testuali dentro lo schema.
- Lo `schema` è scritto in **Skillet**, il linguaggio di schema di Hashbrown (accessibile via `s`). Ricorda Zod ma è ridotto ai costrutti che gli LLM supportano in modo affidabile. È previsto il supporto futuro a JSON Schema e un ponte verso Zod.
- Lo `handler` riceve l'oggetto definito dallo schema e delega alla logica di sistema (store, router). Può anche **restituire valori al modello**:

```ts
// Listing 13-6 — handler che ritorna dati al modello
export const getLoadedFlights = createTool({
  name: 'getLoadedFlights',
  description: `Returns the currently loaded/displayed flights`,
  handler: () => {
    const store = inject(FlightBookingStore);
    return Promise.resolve(store.flightsValue());
  },
});
```

> [!warning] Gotcha
> Skillet **non** descrive formalmente il valore di ritorno dei tool: il modello accetta qualsiasi forma di risposta. Se vuoi informarlo sulla struttura del risultato, descrivilo **come testo libero** nella `description`.

## Under the Hood (tool calling)
> 📖 pp.353-354

Guardando i messaggi inviati all'LLM si vede il meccanismo: la conversazione passa attraverso ruoli (`user`, `assistant`, `tool`), e l'elenco dei tool con i relativi metadata viene appeso in fondo alla richiesta.

```jsonc
// Listing 13-7 — estratto del payload verso l'LLM
{
  "model": "gpt-5-chat-latest",
  "system": "You are Flight42, a UI assistant [...]",
  "messages": [
    { "role": "user", "content": "Ok, let's search for flights from Graz to Hamburg." },
    { "role": "assistant", "content": "",
      "toolCalls": [
        { "id": "call_...", "type": "function",
          "function": { "name": "findFlights",
                        "arguments": "{\"from\":\"Graz\",\"to\":\"Hamburg\"}" } }
      ] },
    { "role": "tool", "content": { "status": "fulfilled" },
      "toolCallId": "call_...", "toolName": "findFlights" },
    { "role": "assistant", "content": "Here are the available flights [...]",
      "toolCalls": [] }
  ],
  "tools": [
    { "name": "findFlights",
      "description": "Searches for flights [...]",
      "parameters": { /* JSON Schema derivato da Skillet */ } }
  ]
}
```

Sequenza dei ruoli:
- `user` → query testuale dell'utente.
- `assistant` → risponde con un **tool call** (nome + argomenti).
- Hashbrown esegue il tool (search + cambio rotta).
- `tool` → Hashbrown riporta il completamento (con l'eventuale risultato).
- `assistant` → risposta finale all'utente.

L'elenco `tools` (descrizioni testuali + schema degli argomenti) viene trasferito affinché il modello **sappia quali tool ha a disposizione**.

```mermaid
sequenceDiagram
    participant U as User
    participant FE as Frontend (chatResource)
    participant BE as Backend (proxy)
    participant LLM as LLM
    U->>FE: prompt testuale
    FE->>BE: messages + tools (+ system)
    BE->>LLM: richiesta (guardrail server-side)
    LLM-->>BE: assistant + toolCall (name, args)
    BE-->>FE: toolCall
    FE->>FE: esegue handler (store/router)
    FE->>BE: ruolo "tool" (risultato/status)
    BE->>LLM: history aggiornata
    LLM-->>FE: assistant (risposta finale)
    FE-->>U: messaggio in chat
```

## UI chat con uiChatResource
> 📖 pp.355-358

`uiChatResource` va oltre: oltre al tool calling, l'LLM può **rispondere con componenti** renderizzati direttamente nella chat (non più solo testo). È un rimpiazzo drop-in di `chatResource`.

```ts
// Listing 13-8
@Component({ /* … */ })
export class AssistantChatComponent {
  message = signal('');

  chat = uiChatResource({
    model: 'gpt-5-chat-latest',
    system: `You are a UI assistant that helps with finding flights …`,
    tools: [findFlightsTool, getLoadedFlights, getBookedFlights],
    components: [flightWidget, messageWidget],   // componenti esposti all'LLM
  });

  submit() {
    const message = this.message();
    this.message.set('');
    this.chat.sendMessage({ role: 'user', content: message });
  }
}
```

I componenti vanno registrati in `components` (descritti via Skillet, vedi sotto). **Con `uiChatResource` l'LLM risponde a ogni domanda con uno o più componenti**: per questo si registra anche un `messageWidget` che riceve e mostra una semplice stringa.

Per renderizzare i componenti scelti dall'LLM si usa `hb-render-message`:

```html
<!-- Listing 13-9 -->
@for (message of messageModels(); track $index) {
  <article class="msg assistant">
    <div class="avatar">{{ message.icon }}</div>
    <div>
      <div class="bubble">
        @if (message.role === 'assistant') {
          <hb-render-message [message]="message" />
        } @else {
          <app-message [data]="message.contentString"></app-message>
        }
      </div>
    </div>
  </article>
}
```

`hb-render-message` serve solo per le risposte dell'assistant. Le richieste utente qui sono puramente testuali (in `content`). Siccome `content` può essere anche number o JSON, va convertito in stringa; questa "proiezione" (più la scelta dell'icona) si fa con [[computed]]:

```ts
// Listing 13-10
messageModels = computed(() =>
  this.chat.value().map((message) => ({
    ...message,
    contentString: String(message.content),
    icon: this.icons[message.role] || '❓',
    toolCalls: message.role === 'assistant' ? message.toolCalls : [],
  })),
);
```

## Dumb components con smart wrapper
> 📖 pp.359-360

I componenti offerti all'LLM sono **dumb component** oppure, come `flightWidget`, uno **smart wrapper** intorno a un dumb component (cfr. componenti e input/output del [[02-signal-based-components]]).

```ts
// Listing 13-11
@Component({
  selector: 'app-flight-widget',
  imports: [FlightCardComponent],
  template: `
    <div class="flight">
      <app-flight-card [item]="flight()" [selected]="isSelected()">
        <div>
          @if (isBooked()) {
            <button class="btn btn-default" (click)="checkIn()">Check in</button>
          } @else if (isSelected()) {
            <button class="btn btn-default" (click)="select(false)">Remove</button>
          } @else {
            <button class="btn btn-default" (click)="select(true)">Select</button>
          }
        </div>
      </app-flight-card>
    </div>
  `,
})
export class FlightWidgetComponent {
  router = inject(Router);
  store = inject(FlightBookingStore);

  flight = input.required<Flight>();
  status = input<'booked' | 'other'>('other');

  isBooked = computed(() => this.status() === 'booked');
  isSelected = computed(() => this.store.basket()[this.flight().id]);

  checkIn(): void { this.router.navigate(['/checkin', this.flight().id]); }
  select(selected: boolean): void { this.store.updateBasket(this.flight().id, selected); }
}
```

Il wrapper inoltra i suoi [[signal-input|input]] al dumb component e gestisce gli eventi scatenando azioni negli store o cambi rotta. Punto chiave: l'input `status` (`'booked' | 'other'`) — l'LLM deve **derivarlo dalla conversazione**; in base al valore il widget mostra il pulsante "Check in" (volo prenotato) o "Select"/"Remove" (volo trovato in ricerca).

## Describing Components
> 📖 pp.360-361

I componenti si descrivono con `exposeComponent`: `name`, `description` (quando usarlo) e schema Skillet di ogni `input`.

```ts
// Listing 13-12
import { exposeComponent } from '@hashbrownai/angular';
import { s } from '@hashbrownai/core';

export const flightWidget = exposeComponent(FlightWidgetComponent, {
  name: 'flightWidget',
  description: `
    Displays a flight or flight ticket. Use this instead of textual
    descriptions of flights or tickets.
  `,
  input: {
    flight: FlightSchema,                       // delega a uno schema esistente
    status: s.enumeration(
      `Whether the flight is booked or not […]`,
      ['booked', 'other'],
    ),
  },
});
```

```ts
// Listing 13-13 — schema riusabile
import { s } from '@hashbrownai/core';

export const FlightSchema = s.object('Flight to be displayed', {
  id: s.number('The flight id'),
  from: s.string('Departure city. No code but the city name'),
  to: s.string('Arrival city. No code but the city name'),
  date: s.string('Departure date in ISO format'),
  delay: s.number('If delayed, this represents the delay in minutes'),
});
```

## Under the Hood: Structured Output
> 📖 pp.361

Per decidere quali componenti mostrare, Hashbrown istruisce l'LLM a rispondere **solo con documenti JSON** (structured output). Il JSON arriva come stringa in `content` e contiene i componenti da mostrare con i valori degli input.

```jsonc
// Listing 13-14 — content = structured output (stringificato)
{
  "role": "assistant",
  "content": "{\"ui\":[{\"messageWidget\":{\"$props\":{\"data\":\"Yes, you have already booked a flight to France.\"}}},{\"flightWidget\":{\"$props\":{\"flightInfo\":{\"id\":2,\"from\":\"London\",\"to\":\"Paris\",\"date\":\"2025-12-05T...\",\"delay\":0,\"status\":\"booked\"}}}}]}",
  "toolCalls": []
}
```

I componenti possibili (con input e descrizioni) vengono passati in una sezione separata della richiesta come **JSON Schema derivato da Skillet**.

## Supporting Different Models
> 📖 pp.362

Alcuni modelli (es. Google Gemini) **non supportano (ancora) la combinazione structured output + tool calling**. Si risolve al bootstrap con `emulateStructuredOutput`:

```ts
// Listing 13-15
bootstrapApplication(AppComponent, {
  providers: [
    provideHashbrown({
      baseUrl: 'http://localhost:3000/api/chat',
      emulateStructuredOutput: true,
    }),
  ],
});
```

Con `true`, Hashbrown definisce uno **pseudo-tool** che permette all'LLM di selezionare i componenti, e istruisce il modello a rispondere chiamando quel tool.

## Applying Few-Shot Prompting
> 📖 pp.362-364

Per far rispondere sempre con testo + eventuali componenti, e per aiutare i modelli più deboli/economici (es. Gemini Flash), si arricchisce il system prompt della resource con esempi.

```text
// Listing 13-16 — estratto del system prompt
## Rules:
- Answer questions with the messageWidget to provide some text to the user.
- When appropriate, *also* answer with other components (widgets), e.g. flightWidget ...
- Instead of describing a flight, use the flightWidget
- Don't call the same tool more than once with the same parameters!

## EXAMPLE
- User: Which flights did I book?
- Assistant:
  - UI: messageWidget(You've booked these 3 flights)
  - UI: flightWidget({id: 0, from: '...', to: '...', ...})

## NEGATIVE EXAMPLE
Don't call the same tool several times in a row with the same parameters: ...
```

Mettere esempi nel prompt migliora la qualità delle risposte: più esempi = **few-shot prompting**, un solo esempio = **one-shot prompting**. La stessa tecnica serve anche dentro la `description` di un input, per far inferire correttamente lo `status` (`booked`/`other`):

```ts
// Listing 13-17 — esempi dentro la description dell'enumeration
status: s.enumeration(
  `Whether the flight is booked or not.
   A flight has the status 'booked' **only** when retrieved via 'getBookedFlights'.
   ## Example for inferring a status 'booked'
   - User: Which flights did I book?
   - Assistant:
     - Tool: getBookedFlights()
     - UI: flightWidget({flightInfo: { id: 0, ..., status: 'booked' }})
  `,
  ['booked', 'other'],
),
```

Gli esempi in **prosa** sono facili da scrivere ma fragili (componenti/parametri potrebbero non esistere più). Hashbrown offre il tag template `prompt` che verifica gli esempi (in un dialetto XML) contro i componenti registrati:

```ts
// Listing 13-18
uiChatResource({
  system: prompt`
    <user>Hello</user>
    <assistant>
      <ui>
        <app-message data="How may I assist you?" />
      </ui>
    </assistant>
  `,
  components: [exposeComponent(MessageComponent, { /* ... */ })],
});
```

> [!warning] Gotcha
> Il tag `prompt` (con validazione XML vs componenti) funziona **solo per il system prompt nella resource**. Se sovrascrivi il system prompt lato server (per sicurezza) e/o ti servono esempi dentro le descrizioni dei componenti, resti vincolato agli **esempi in prosa** non validati.

## Natural Language Queries — Approach
> 📖 pp.365-366

Si va ancora oltre la selezione da un catalogo di componenti: l'app **genera codice** per parti dinamiche. Scenario: l'utente descrive a parole un report ("% di voli in ritardo per giorno, ordinati per data"), l'app trasforma i dati e mostra un chart.

Il punto cruciale: tra il **recupero** dei dati (tool calling) e la **visualizzazione** serve una **trasformazione** (aggregazione). Gli LLM sono bravi a continuare testo/rispondere, **non** a fare calcoli — però sono ottimi nel **descrivere cosa va fatto**. Quindi: lasciamo che l'LLM derivi i passi di elaborazione dalla richiesta, espressi come **codice JavaScript**.

```js
// Listing 13-19 — codice GENERATO dall'LLM per preparare i dati del chart
const routes = [
  { from: 'Graz', to: 'Hamburg' }, { from: 'Hamburg', to: 'Graz' },
  { from: 'Graz', to: 'New York' }, { from: 'New York', to: 'Graz' },
  // ...
];

let flights = [];
routes.forEach((route) => {
  flights = flights.concat(loadFlights(route));   // funzione fornita dall'app
});

const flightsByDate = {};
for (const flight of flights) {
  const date = flight.date.split('T')[0];          // ignora l'orario
  if (!flightsByDate[date]) flightsByDate[date] = { total: 0, delayed: 0 };
  flightsByDate[date].total++;
  if (flight.delay > 0) flightsByDate[date].delayed++;
}

const data = Object.keys(flightsByDate).sort().map((date) => {
  const { total, delayed } = flightsByDate[date];
  const percentage = ((delayed / total) * 100) || 0.1;
  return { name: date, value: percentage };
});

generateChart({ data });   // sink fornito dall'app
```

Il codice delega a due funzioni fornite dall'app: `loadFlights` (recupera i dati) e `generateChart` (sink che mostra il chart). L'LLM **non genera solo codice**, decide anche quali funzioni dell'app il codice deve invocare.

## Implementation with Hashbrown
> 📖 pp.367-368

Due building block: `structuredCompletionResource` + una **runtime JavaScript**.

`structuredCompletionResource` è **stateless** (a differenza di `chatResource`/`uiChatResource`): non è una conversazione lunga, ma una singola risposta a una richiesta concreta. Qui la risposta è un oggetto con un messaggio per l'utente **e** il codice generato. La resource inoltra il codice alla runtime via tool calling; la runtime lo **esegue in sandbox**, senza accesso diretto all'app.

```ts
// Listing 13-20
import {
  createRuntime,
  createRuntimeFunction,
  createToolJavaScript,
  structuredCompletionResource,
} from '@hashbrownai/angular';

@Component({ selector: 'app-reporting', /* … */ })
export class ReportingComponent {
  message = signal('');
  input = signal<string | undefined>(undefined);
  data = signal<DataItem[]>([]);

  runtime = createRuntime({
    functions: [
      createRuntimeFunction(/* ... */),   // loadFlights
      createRuntimeFunction(/* ... */),   // generateChart
    ],
  });

  generator = structuredCompletionResource({
    model: 'gpt-5-chat-latest',
    input: this.input,                     // signal: cambiando, ri-triggera la resource
    system: `You are Report42, a UI assistant that [...] generate JavaScript code that [...]`,
    schema: s.object(`Whether request was successful`, {
      type: s.enumeration(`Success or error?`, ['success', 'error']),
      message: s.string(`Additional information for the user`),
      code: s.string(`The generated JavaScript code`),
    }),
    tools: [
      createToolJavaScript({ runtime: this.runtime }),   // la runtime è registrata come tool
    ],
  });

  submit(): void {
    this.input.set(this.message());        // copia l'input → triggera la resource
  }
}
```

- Il [[signal]] `message` è legato a una textbox; `submit()` lo copia in `input`, che è il signal di input della resource → la fa partire.
- `createRuntime` riceve le funzioni che il codice generato può chiamare e crea la runtime JS; viene registrata come tool via `createToolJavaScript`.
- Lo `schema` impone la struttura della risposta: esito (`success`/`error`), `message` per l'utente e `code` generato.

## Runtime Functions
> 📖 pp.369-370

`createRuntimeFunction` crea una funzione che il codice generato può chiamare nella runtime. Definisce `name`, `description`, `args` (schema input), `result` (schema output) e `handler`.

```ts
// Listing 13-21 — loadFlights: DATA SOURCE
createRuntimeFunction({
  name: 'loadFlights',
  description: `
    Searches for flights and returns them.
    ## Rule
    For the search parameters, airport codes are NOT used but the city name.
  `,
  args: s.object('search parameters for flights', {
    from: s.string('airport of departure'),
    to: s.string('airport of destination'),
  }),
  result: s.array(`loaded flights`, FlightSchema),   // qui il result È descritto via Skillet
  handler: async (input) => {
    const flightService = inject(FlightService);
    const result = flightService.find(input.from, input.to);
    return await firstValueFrom(result);
  },
});
```

```ts
// Listing 13-23 — generateChart: SINK
createRuntimeFunction({
  name: 'generateChart',
  description: `Creates a chart`,
  args: s.object(`Chart description`, {
    data: s.array(
      `name/value pairs to display in chart`,
      s.object(`a single name/value pair to display in the chart`, {
        name: s.string(`name`),
        value: s.number(`the value to display`),
      }),
    ),
  }),
  handler: async (input) => {
    this.data.set(input.data);   // mette i dati della runtime in un signal del componente
  },
});
```

Dettaglio importante: a differenza di `loadFlights` (data source), `generateChart` è un **sink** — il suo handler inoltra i dati ricevuti direttamente al componente (li mette nel signal `data`). Il chart vero è renderizzato con **chart.js** dentro un [[effect]] (non mostrato).

> [!warning] Gotcha
> Nelle runtime function il `result` **è** descritto via Skillet (es. `s.array(..., FlightSchema)`), così il modello sa cosa aspettarsi come ritorno — diversamente dai tool del chatResource, dove il valore di ritorno non era schematizzato.

## System Prompt con One-Shot Prompting
> 📖 pp.371-372

Il system prompt di `structuredCompletionResource` definisce i guardrail per la generazione del codice: passi chiave, un esempio, regole generali.

```text
// Listing 13-24 — system prompt (estratto)
You are Report42, a UI assistant that helps passengers with creating ... a chart ...

## Your Tasks
1. Take the user's request ... and generate JavaScript code that ...
   a) uses the tool _loadFlights_ as often as needed to retrieve the needed data
   b) Aggregate the received data according to the user's request. Replace 0 with 0.1
   c) Pass the data to the tool _generateChart_ to display a chart
2. Pass the JavaScript code to the runtime

## Example for the JavaScript Code
- User: How many flights are there from Graz to London and from Graz to Munich?
- Assistant
  - Code:
    const flights1 = loadFlights({ from: 'Graz', to: 'London' });
    const flights2 = loadFlights({ from: 'Graz', to: 'Munich' });
    const data = [
      { name: 'Graz - London', value: flights1.length },
      { name: 'Graz - Munich', value: flights2.length },
    ];
    generateChart({ data });
  - Answer: Here is your chart.

## Rules
- Never use additional web resources for answering requests
- **Always** pass the generated code to the JavaScript runtime
```

Fornire un esempio (**one-shot prompting**) migliora la performance del modello su task come questo.

> [!tip] Take-away
> Code generation = l'LLM descrive *come* trasformare i dati (cosa in cui è bravo), non *fa* il calcolo. Il codice gira in **sandbox** con una **allow-list esplicita** di funzioni (`loadFlights` source, `generateChart` sink), schema strutturato per la risposta ed esempi per la consistenza.

## 🔁 Ripasso lampo
1. Perché la API key dell'LLM non va nel frontend e cosa permette di fare `transformRequestOptions` nel backend?
2. Cosa distingue `chatResource`, `uiChatResource` e `structuredCompletionResource`? Quale è stateless e perché?
3. Cosa definisce `createTool` (name/description/schema/handler) e perché la `description` è critica per l'LLM? Skillet descrive anche il valore di ritorno?
4. Cos'è la generative UI con `exposeComponent` + `hb-render-message`? A cosa serve `emulateStructuredOutput`?
5. Differenza tra one-shot e few-shot prompting? Dove si possono mettere gli esempi e cosa fa il tag template `prompt`?
6. Nel code generation, che ruolo hanno `createRuntime`, `createRuntimeFunction` e `createToolJavaScript`? Perché l'esecuzione avviene in sandbox?

**Take-away del capitolo:**
- **Hashbrown** connette Angular a più provider LLM e gestisce il plumbing ripetitivo; la sicurezza/costi si controllano con un **backend proxy** (system instruction, scelta del modello).
- **Tool calling** trasforma la chat in un'interfaccia che recupera dati e scatena azioni reali: `createTool` + schema **Skillet** + descrizioni testuali rendono le chiamate affidabili. Tradurre le tracce tecniche in messaggi user-friendly.
- **Generative UI** (`uiChatResource` + `exposeComponent`) fa scegliere all'LLM componenti predefiniti via **structured output**, con `emulateStructuredOutput` come fallback e **few/one-shot prompting** per i modelli più deboli.
- **Code generation** (`structuredCompletionResource` + runtime JS) per scenari dinamici: l'LLM genera JavaScript per le trasformazioni, eseguito in **sandbox** con allow-list di funzioni (data source / sink).
