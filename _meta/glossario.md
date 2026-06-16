---
titolo: "Glossario"
tags: [tipo/glossario]
---
# 📖 Glossario

Termini tecnici ricorrenti del libro, spiegati in parole semplici. I capitoli linkano qui alla prima occorrenza di un termine; i nomi di API e keyword restano in inglese (non si traducono), ma qui trovi cosa significano davvero.

## Reattività e change detection

### change detection
Il meccanismo con cui Angular **controlla se qualcosa è cambiato** nei dati e, se sì, **aggiorna il DOM** di conseguenza. In pratica: ricalcola cosa mostrare quando lo stato cambia. Con i signal questo controllo diventa mirato (solo ciò che dipende dal valore cambiato).

### reactive context
La "zona" di codice in cui Angular **tiene traccia di quali signal leggi**, così sa cosa ricalcolare quando cambiano. Dentro un `computed`/`effect`/template sei in reactive context (le letture vengono tracciate); in una funzione normale no.

### signal graph
Il **grafo delle dipendenze** tra signal: chi legge chi. Un `computed` che legge due signal ne "dipende"; quando uno cambia, Angular sa esattamente quali nodi del grafo ricalcolare. Pensare in termini di grafo = progettare il flusso dei dati come catene di dipendenze.

### glitch-free
Garanzia che i consumatori di un signal vedono **solo valori coerenti e già stabilizzati**, mai stati intermedi incoerenti durante un aggiornamento. Niente "sfarfallii" di valori a metà.

### immutability (immutabilità)
Trattare lo stato **senza modificarlo sul posto**: invece di cambiare un oggetto/array esistente, ne crei uno nuovo. Serve perché i signal notificano un cambiamento solo se il **riferimento** cambia (`Object.is`): `arr.push(x)` non notifica, `[...arr, x]` sì.

### race condition
Quando **due operazioni asincrone** (es. due chiamate HTTP) finiscono in ordine imprevedibile e l'ultima a rispondere "vince", magari sovrascrivendo dati più recenti con dati vecchi. `httpResource` la gestisce annullando le richieste superate.

### debounce (debouncing)
**Aspettare che l'utente smetta** di digitare/agire prima di reagire: invece di lanciare un'azione a ogni tasto, attendi un attimo di pausa. Evita raffiche di chiamate inutili.

## Componenti, template e direttive

### content projection
Quando un componente **riceve del markup dal chiamante** e lo mostra dentro di sé, nel punto segnato da `<ng-content>`. È il "buco" dove infili il contenuto passato dall'esterno.

### View e Content
Due cose distinte di un componente: la **View** è ciò che il componente definisce nel **proprio template**; il **Content** è il markup che il **chiamante** gli passa (e che finisce in `<ng-content>`). Si interrogano con query diverse (`viewChild` vs `contentChild`).

### attribute directive
Una direttiva che **aggiunge comportamento a un elemento esistente** senza avere un proprio template (es. un tooltip su un bottone). Si chiama "attribute" perché di solito la applichi come un attributo HTML.

### structural directive
Una direttiva che **aggiunge o rimuove elementi dal DOM** (cambia la "struttura" della pagina), come facevano `*ngIf`/`*ngFor`. Oggi per il control flow si usano i blocchi nativi `@if`/`@for`.

### host (elemento host)
L'**elemento del DOM su cui è applicata** una direttiva o che "ospita" un componente. L'opzione `host` del decorator lega classi/stili/eventi proprio a quell'elemento.

### exportAs
Un **nome pubblico** che una direttiva si dà per poter essere "afferrata" da una template variable. Siccome su un elemento possono esserci più direttive, `#x="nomeExportAs"` dice ad Angular *quale* istanza vuoi.

### template variable
Una variabile dichiarata nel template con `#nome`, che ti dà un **riferimento** a un elemento, componente o direttiva, così puoi leggerne proprietà o chiamarne metodi da un altro punto del template.

## Dependency Injection e servizi

### dependency injection (DI)
Il pattern per cui un oggetto **non crea le proprie dipendenze ma se le fa "consegnare"** da Angular. Tu chiedi un servizio con `inject(X)` e Angular ti passa l'istanza giusta, senza che tu sappia come è costruita.

### injection context
Il **momento/luogo in cui Angular sa risolvere le dipendenze**: dentro il costruttore, gli initializer di campo, o le factory dei provider. Funzioni come `inject()` funzionano solo qui; fuori da questo contesto danno errore.

### provider
La **ricetta** che dice ad Angular *come ottenere* il valore per un certo token: con quale classe (`useClass`), valore (`useValue`), factory (`useFactory`) o alias (`useExisting`). Lo configuri nell'array `providers`.

### injection token
La **chiave** con cui chiedi una dipendenza. Spesso è una classe, ma può essere un token esplicito (`InjectionToken`) per iniettare valori che non sono classi (config, URL, costanti).

### standalone
Un componente/direttiva/pipe che **non ha bisogno di un NgModule**: dichiara da solo cosa importa. Dalla v17+ è il default; scrivere `standalone: true` è ormai ridondante.

## Routing e ciclo di vita

### lazy loading
Caricare il codice di una parte dell'app **solo quando serve** (es. quando navighi su quella rotta), invece che tutto all'avvio. Riduce il peso iniziale e velocizza il primo caricamento.

### guard
Una funzione che **decide se una navigazione può avvenire** (attivare o lasciare una rotta), es. un redirect al login se non sei autenticato. Nelle SPA è una questione di *usabilità*: la sicurezza vera la fa il backend.

### resolver
Una funzione che **carica i dati prima** che la rotta venga attivata, così il componente parte già con i dati pronti invece di mostrare uno stato vuoto.

### interceptor (HttpInterceptor)
Un "filtro" che **intercetta tutte le richieste/risposte HTTP** per modificarle in modo centralizzato: aggiungere header (es. il token di auth), loggare, gestire errori.

## Performance, rendering e SSR

### tree-shaking
L'eliminazione automatica, in fase di build, del **codice mai usato**, così il bundle finale è più leggero. "Scuoti l'albero" e cade ciò che non è agganciato.

### zoneless / Zone.js
**Zone.js** è la vecchia libreria che faceva scattare la change detection "magicamente" a ogni evento. **Zoneless** = farne a meno, lasciando che siano i **signal** a dire ad Angular cosa aggiornare: più prevedibile e leggero.

### deferrable view (`@defer`)
Un blocco di template che viene **caricato e renderizzato in ritardo**, su un trigger (quando entra nel viewport, al click, in idle…). Serve a non pagare subito il costo di parti pesanti o secondarie.

### SSR (Server-Side Rendering)
Generare l'HTML della pagina **sul server** e mandarlo già pronto al browser, invece di costruirlo tutto lato client. L'utente vede contenuto prima; utile anche per la SEO.

### hydration
Il processo per cui Angular, ricevuto l'HTML renderizzato dal server, lo **"riattiva" lato client** riallacciando event listener e stato, **senza ricostruirlo da zero** (evita lo sfarfallio e il lavoro doppio).

### incremental hydration
Variante in cui l'idratazione avviene **a pezzi, su richiesta** (es. quando una sezione entra nel viewport), invece che tutta insieme. Dalla v22 è il comportamento di default.

### event replay
Cattura gli **eventi dell'utente avvenuti prima** che la pagina sia idratata (es. un click precoce) e li **ri-esegue** una volta pronta, così nessuna interazione va persa.

## Architettura e scaling

### vertical slicing
Organizzare il codice **per funzionalità/dominio** (una "fetta verticale" che contiene tutto ciò che serve a quella feature) invece che per tipo tecnico (tutti i servizi insieme, tutti i componenti insieme).

### modulith
Un **monolite ben strutturato in moduli** con confini chiari: un solo deployable, ma internamente diviso in parti che si nascondono i dettagli a vicenda (information hiding). Via di mezzo tra monolite e microservizi.

### barrel
Un file `index.ts` che **ri-esporta** il contenuto pubblico di una cartella, così gli altri importano da un unico punto (`@app/feature`) invece che dai file interni. Definisce l'API pubblica di un modulo.

### monorepo
**Un unico repository** che contiene più app e librerie insieme, condividendo codice e configurazioni, invece di tanti repo separati.

### micro frontend
Spezzare un'app frontend grande in **parti indipendenti**, sviluppate e rilasciate da team diversi, che a runtime si compongono in un'unica applicazione.

### Native Federation / Module Federation
Tecnologie per **caricare a runtime pezzi di app costruiti separatamente** (i micro frontend), condividendo le librerie comuni. "Federation" = federare moduli che vivono in build diverse.

### web component / custom element
Un componente impacchettato come **elemento HTML standard** (`<mio-widget>`), usabile in qualsiasi pagina o framework, non solo in Angular.

## Stato (NgRx Signal Store)

### store
Un contenitore **centralizzato dello stato** di una parte dell'app, con un modo controllato di leggerlo e modificarlo, così il flusso dei dati è prevedibile.

### reducer
Una funzione **pura** che, dato lo stato attuale e un evento, **restituisce il nuovo stato** (senza modificare il vecchio). È il punto unico dove lo stato cambia.

### dispatch
**Inviare un evento** allo store ("è successa questa cosa"), che poi i reducer/handler trasformano in un cambiamento di stato. Tu annunci l'intenzione, lo store decide come reagire.

### entity / normalization
Una **entity** è un singolo oggetto con un id (un volo, un utente). La **normalization** è tenerle in una mappa `id → entity` invece che in liste annidate, per evitare duplicati e aggiornamenti incoerenti.

### optimistic update
Aggiornare subito la UI **come se l'operazione fosse già riuscita**, prima della risposta del server, e fare rollback solo se fallisce. La UI sembra istantanea.

## Sicurezza (auth)

### OAuth 2 / OIDC
**OAuth 2** è lo standard per **autorizzare** l'accesso a risorse senza condividere la password; **OpenID Connect (OIDC)** ci aggiunge sopra l'**autenticazione** (sapere *chi* è l'utente). Spesso usati insieme.

### JWT (JSON Web Token)
Un **token firmato** che contiene informazioni sull'utente (claims) in formato JSON. Il server lo firma; chi lo riceve può verificarne l'autenticità senza interrogare di nuovo chi l'ha emesso.

### PKCE
Un'estensione di OAuth 2 che **lega la richiesta di login al client** che l'ha iniziata (con una coppia `code_verifier`/`code_challenge`), così un token intercettato non è riutilizzabile da un attaccante.

### XSRF / CSRF
Attacco in cui un sito malevolo **fa partire richieste a tua insaputa** sfruttando i cookie già presenti nel browser. Si mitiga con token anti-XSRF e attributi sui cookie (`SameSite`).

### Bearer token
Un token di accesso che il client **allega a ogni richiesta** nell'header `Authorization: Bearer <token>` per dimostrare di essere autorizzato. "Bearer" = "al portatore": chi lo possiede può usarlo, quindi va protetto.

## Tooling, build e avvio

### bootstrap
L'**avvio dell'applicazione** Angular: il root component viene istanziato e "montato" nel DOM (con `bootstrapApplication()`), mettendo in moto tutta l'app.

### schematics
Le **"ricette" della Angular CLI**: script che, con `ng generate` / `ng add`, sanno quali file creare o modificare e con quali opzioni di default. Sono il motore dietro i comandi di scaffolding.

### path mapping
Un **alias di import** definito in `tsconfig.json` che fa corrispondere un nome breve (es. `@app/feature`) a una cartella/file su disco, così gli import non dipendono dai percorsi relativi.

### shadowing (DI)
Quando un provider in uno **scope più interno oscura** quello esterno con lo stesso token: chi sta dentro riceve la versione più vicina nella gerarchia DI, non quella di livello app.

### environment provider
Un provider registrato a livello di **"ambiente"** (root o rotta), valido per intere porzioni dell'app e non per un singolo componente — usato ad esempio dal router per i servizi route-local.

### auto-provided
Un service **registrato automaticamente** nel root injector grazie a `@Service()` (cioè `providedIn: 'root'`), senza bisogno di inserirlo a mano in un array `providers`. Con `@Service({ autoProvided: false })` lo escludi, per fornirlo tu dove serve.
