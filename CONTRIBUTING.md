# Come contribuire

Prima di modificare il codice, leggere i [vincoli
architetturali](ARCHITECTURE.md#1-vincoli-architetturali), gli
[invarianti](ARCHITECTURE.md#8-invarianti) e le [insidie
note](ARCHITECTURE.md#9-insidie-note).

## Ambiente di sviluppo

Nessuno. Non c'è `npm install`, non c'è build, non ci sono dipendenze da installare: si
modifica `index.html` con un editor di testo e si ricarica il browser.

Serve una connessione a Internet, perché librerie, cartografia e geocodifica arrivano da CDN.

## Convenzioni di codice

- **JavaScript vanilla**, dentro l'unica IIFE. Niente framework, niente import: romperebbero
  l'apertura da `file://`.
- **Italiano** per interfaccia, messaggi e commenti.
- Commentare il **perché**, non il cosa. I commenti esistenti spiegano decisioni non ovvie
  (CORS, Referer, renderer canvas): mantenere quello stile.
- Ogni sezione dell'interfaccia è un `<details class="section">` con numero progressivo.
- Gli elenchi riusano le classi `.building-list` / `.building-row` / `.building-select` /
  `.delete-btn`. L'elenco delle strade lo fa già: nessun CSS duplicato.
- I colori nuovi vanno in `COLORS` (JS) e, se servono anche al CSS, come variabile in `:root`.
- Ogni nuova funzione di rendering si chiama `render<Cosa>()` e va invocata dopo le mutazioni.
- I messaggi all'utente passano da `toast()` (temporanei), `formMessage()` (nel form) o
  `storageWarning()` (footer): sempre in italiano e sempre con un'indicazione di cosa fare.
- Le azioni distruttive offrono un annullamento nel toast, che ripristina l'elemento nella
  **posizione originale** (`splice`, non `push`).

## Aggiungere una categoria di edifici

Le categorie non sono definite in un punto solo. Per aggiungerne una (come è stato fatto per
`known`) bisogna toccare, in `index.html`:

1. **`COLORS`, `LABELS` e `CATEGORY_CLASS`** nello script: sono la sorgente dei dati.
2. **CSS**: variabile di colore in `:root`, e le varianti `.category.<colore>`, `.dot.<colore>`,
   `.legend-pin.<colore>`, `.popup-category.<colore>` e `.<colore>-text` per il contatore.
3. **Form** (`name="category"`): un nuovo `<label class="category <colore>">` con il suo radio.
4. **Contatori** nell'intestazione: un nuovo `.stat` con id `<chiave>-count`, e la riga
   corrispondente in `renderBuildings()` (`count('<chiave>')`).
5. **Legenda** in `legendElement()`: serve sia per la mappa sia per il PNG.
6. `README.md` e questa documentazione.

Non servono modifiche a pin, export PNG, GeoJSON e validazione, perché leggono `COLORS` e
`LABELS` per chiave. Collaudo specifico: aggiungere un edificio per categoria, controllare i
contatori, esportare e reimportare un GeoJSON, importare un GeoJSON *vecchio* senza la nuova
categoria, e verificare che una categoria inventata venga rifiutata senza toccare l'archivio.

## Accessibilità

Il livello attuale va mantenuto: `aria-label` sui controlli icona, `role="status"` con
`aria-live` per i messaggi, `aria-pressed` sui toggle, focus visibile
(`:focus-visible` con outline), `Esc` per uscire dalle modalità, `.sr-only` per il testo
riservato agli screen reader.

## Collaudo

Non ci sono test automatici: il programma si verifica aprendolo. Percorso minimo dopo ogni
modifica:

1. aggiungere un edificio in tutti e tre i modi (ricerca, coordinate, punto sulla mappa);
2. tracciare due strade, selezionarle da elenco e da mappa, eliminarne una e annullare;
3. ricaricare la pagina e verificare che tutto sia tornato (persistenza);
4. esportare un PNG e controllare che le strade siano **rosse tratteggiate**, con legenda e
   attribuzione;
5. esportare un GeoJSON, reimportarlo e verificare che i dati coincidano;
6. controllare che la console del browser sia pulita.

⚠️ Aprendo il file in un contesto diverso da `file://` (anteprime, `data:` URL, server locale)
**si usa un `localStorage` diverso**: non si vedono i dati reali dell'utente. Utile per i test,
fuorviante se non lo si sa.

## Pubblicazione

Il sito è servito da GitHub Pages dalla radice di `main`: dopo il `git push` l'aggiornamento è
online in circa un minuto. Prima di spingere:

1. verificare il percorso di collaudo qui sopra;
2. aggiornare `README.md` se cambia qualcosa di visibile all'utente.

Se si cambia la versione di Leaflet o di html2canvas, **rigenerare gli hash SRI** dei tag in
`index.html` (attributo `integrity`). Esempio per un file scaricato:

```bash
openssl dgst -sha384 -binary leaflet.js | openssl base64 -A
```

Il valore va preceduto da `sha384-`. Un hash sbagliato fa rifiutare la libreria al browser: la
mappa non compare e la console mostra un errore di integrità.

## Miglioramenti utili

In ordine di utilità, dal punto di vista di chi usa il programma sul campo:

1. **Modifica di un edificio esistente** — oggi va eliminato e riaggiunto, perdendo le note.
2. **Fusione in import**, come alternativa alla sostituzione, per unire il lavoro di più persone.
3. **Export PDF o stampa** della mappa con l'elenco allegato.
4. **Ricerca/filtro** nell'elenco edifici, utile oltre il centinaio di rilievi.
5. **Etichetta o nome per le strade**, oggi identificate solo dal numero d'ordine.
