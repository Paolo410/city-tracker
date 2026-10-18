# Architettura

Come è fatto il programma e perché. Per una panoramica d'uso vedi [README.md](README.md);
per le convenzioni di modifica [CONTRIBUTING.md](CONTRIBUTING.md).

## 1. Vincoli architetturali

Sono scelte deliberate: violarle rompe il modello d'uso del programma.

| Vincolo | Motivo |
|---|---|
| **Un solo file HTML**, senza build step | L'utente apre un indirizzo web (o scarica un solo file) senza installare nulla. Nessun `npm`, nessun bundler, nessun server. |
| **Nessun backend** | Nessun costo, nessuna manutenzione, nessun dato personale su server di terzi. |
| **Nessun framework** | JavaScript vanilla in una IIFE. React o Vue richiederebbero un build step. |
| **Funziona da `file://`** | Il programma è aperto come file locale, non servito via HTTP. |
| **Interfaccia bilingue (italiano e inglese)** | L'utente tipico è italiano, ma lo strumento è usabile ovunque. Il markup statico è in italiano; l'inglese si attiva con il selettore IT/EN (vedi §12). I commenti nel codice sono in italiano. |
| **Degradazione leggibile** | Senza rete la mappa non carica, ma i dati locali e l'export GeoJSON restano accessibili. |

### Dipendenze esterne (via CDN, a runtime)

| Libreria | Versione | Uso |
|---|---|---|
| Leaflet | 1.9.4 (unpkg) | mappa, marker, polilinee |
| html2canvas | 1.4.1 (cdnjs) | rasterizzazione per l'export PNG |
| Tile OSM (FOSSGIS) | `tile.openstreetmap.de` | cartografia (primaria) |
| Tile OSM France | `tile.openstreetmap.fr` | cartografia (1º ricambio) |
| Tile Esri | `server.arcgisonline.com` | cartografia (2º ricambio) |
| Photon / komoot | `photon.komoot.io` | geocodifica indirizzi |

Le versioni sono **pinnate** e i tre file caricati da CDN hanno un attributo `integrity` (SRI, sha384): il
browser rifiuta il file se i byte non coincidono. Aggiornando una versione vanno rigenerati gli hash
(vedi [CONTRIBUTING.md](CONTRIBUTING.md#pubblicazione)) e va ritestato l'export PNG, la parte più fragile.

## 2. Struttura del file

`index.html` è l'unico file con logica:

1. `<head>` — meta, CSS di Leaflet, `<style>` con l'intero foglio di stile (token in `:root`).
2. `<body>` — layout a due colonne: `.sidebar` (controlli) + `.map-area` (mappa).
   La sidebar contiene quattro `<details class="section">` numerate 01–04 e l'elenco edifici.
3. `<script>` — una IIFE `(() => { 'use strict'; … })()`. Nessuna variabile globale esportata.

Lo script è organizzato per blocchi, in quest'ordine: costanti e stato → validazione →
persistenza → notifiche UI → impostazioni → edifici → geocodifica → strade → GeoJSON →
export PNG → inizializzazione mappa.

## 3. Modello dati

Unica struttura condivisa, nella variabile `state`:

```js
{
  version: 1,                    // riconoscimento del formato in import
  buildings: [
    { id, address, lat, lng, category, notes }
  ],
  roads: [
    { id, points: [[lat, lng], …] }   // ≥ 2 punti
  ],
  draft: null | { id, points: [] },   // tracciamento in corso, ≥ 0 punti
  settings: { center: [lat, lng], zoom }
}
```

- `id` — stringa da `uid()` (timestamp base36 + random), unica nell'intero stato.
- `category` — una delle chiavi di `CATEGORY_CLASS`: `'worked'` (già lavorato, blu), `'similar'`
  (supposto simile, arancione) o `'known'` (di cui siamo a conoscenza, verde). Colori in `COLORS`,
  classe CSS in `CATEGORY_CLASS`, etichette tradotte nel dizionario (chiavi `cat.<categoria>`).
  `validateState()` accetta solo le chiavi di `CATEGORY_CLASS`: un file con una categoria sconosciuta
  viene rifiutato per intero (vedi §9). La categoria si salva **come chiave**, mai come etichetta:
  per questo un backup si importa in qualsiasi lingua.
- `points` — coppie **`[lat, lng]`**, l'ordine di Leaflet.
  ⚠️ Il GeoJSON usa l'ordine inverso `[lng, lat]`: la conversione avviene solo ai confini,
  in `geojson()` e `parseImport()`.
- `draft` — persistito come il resto, quindi un tracciamento interrotto si recupera alla
  riapertura (`initMap()` ripristina `drawing`).

### Stato di sessione (non persistito)

`drawing`, `picking`, `exporting`, `searching`, `selectedRoadId`, `previewMarker`.
Flag di interazione: vivono solo finché la pagina è aperta.

## 4. Rendering

Non c'è un sistema reattivo: le funzioni `render*()` ricostruiscono il DOM e i layer dallo
stato, e vanno invocate esplicitamente dopo ogni mutazione.

| Funzione | Ricostruisce |
|---|---|
| `renderBuildings()` | marker sulla mappa, elenco edifici, contatori |
| `renderRoads()` | polilinee, bozza, contatori, controlli della sezione 02 |
| `renderRoadList()` | elenco delle strade salvate |
| `renderSettings()` | campi del form impostazioni, etichetta del centro |

I tre layer Leaflet (`buildingsLayer`, `roadsLayer`, `draftLayer`) sono `L.layerGroup`
svuotati e ripopolati a ogni render. È volutamente semplice: il numero di elementi previsto
(ordine delle centinaia) non giustifica un rendering incrementale.

## 5. Persistenza

- **Dove:** `localStorage`, chiave `mappa_edifici_v1`, valore `JSON.stringify(state)`.
- **Quando:** `saveState()` dopo *ogni* mutazione e su `pagehide`. Nessun pulsante "Salva".
- **Caricamento:** `loadState()` all'avvio, con validazione.

### Gestione degli errori

Il flag `storageReady` implementa una regola precisa: **se l'archivio salvato non è
interpretabile, non viene sovrascritto**. `saveState()` esce subito, così i dati corrotti
restano sul disco e possono essere recuperati a mano invece di essere distrutti da un
salvataggio automatico. All'utente viene chiesto di esportare un GeoJSON.

`localStorage` può inoltre lanciare eccezioni (modalità privata, policy aziendali, quota
esaurita): ogni accesso è in `try/catch` e degrada mostrando un avviso nel footer.

### Limiti

`localStorage` è legato all'**origine**. Da `file://` questo significa in pratica che
spostare o rinominare il file, cambiare browser o cancellare i dati dei siti **può rendere
l'archivio irraggiungibile**. Il GeoJSON è l'unico backup che sopravvive a tutto: è il motivo
per cui l'export è presentato con tanta insistenza nell'interfaccia.

## 6. Validazione e sicurezza

`validateState()` e `validateSettings()` sono l'unico ingresso per dati non fidati (archivio
locale e file importati). Applicano tipi, lunghezze massime (indirizzo 300, note 5000),
intervalli di coordinate, `version === 1` e unicità degli `id` (`uniqueId()` rigenera i
duplicati). Ogni anomalia lancia un errore con messaggio già presentabile, nella lingua corrente (`t('err.…')`).

**Prevenzione XSS.** Il testo dell'utente entra nel DOM **solo** via `textContent` o
`document.createTextNode`. Gli unici `innerHTML` contengono markup statico scritto dagli
sviluppatori (icone SVG, legenda). Mantenere la regola: è la difesa principale, dato che
indirizzi e note sono testo libero che viaggia anche nei GeoJSON scambiati tra persone.

**Rete.** L'unica richiesta che trasmette dati dell'utente è la geocodifica, e parte **solo**
su azione esplicita (submit del form senza coordinate). Con coordinate inserite o punto scelto
sulla mappa, nulla lascia il computer. La `fetch` usa `credentials:'omit'`, un
`AbortController` con timeout di 12 s, throttling a 1,5 s, cache per query e un contatore di
sequenza (`searchSequence`) per scartare le risposte fuori ordine.

## 7. Flussi principali

### Aggiunta di un edificio

```
submit del form
   ├─ coordinate presenti → addBuilding(point)
   └─ coordinate assenti  → searchAddress(address) → lista risultati
                             └─ clic su un risultato → addBuilding(point)
```

Terza via: `pick-location` attiva `picking`, e il clic sulla mappa chiama `choosePoint()`, che
piazza un marker di anteprima e compila le coordinate. L'indirizzo è **sempre obbligatorio**:
è l'etichetta che identifica il rilievo.

### Tracciamento di una strada

`startRoad()` crea `state.draft` → ogni clic sulla mappa aggiunge un punto (scartando quelli a
meno di 0,5 m dal precedente) → `finishRoad()` sposta la bozza in `state.roads` (minimo 2
punti). `cancelRoad()` e l'eliminazione offrono un **annullamento** tramite l'azione nel toast,
che ripristina l'elemento nella posizione originale.

### Selezione di una strada (bidirezionale)

`selectedRoadId` è la sorgente unica di verità. `selectRoad(id, fromMap)` fa toggle e richiama
`renderRoads()`, che ridisegna tutto:

- **sulla mappa:** la strada selezionata è viola (`COLORS.roadSelected`), continua, spessa 7 e
  portata in primo piano; le altre restano rosse tratteggiate;
- **nell'elenco:** la riga corrispondente riceve la classe `.selected`;
- con `fromMap = true` la sezione 02 si apre e la riga viene portata in vista.

La selezione **non è persistita** e **non compare negli export** (vedi §8).

### Export PNG

Il percorso più delicato, in `exportPNG()`:

1. blocca l'interfaccia (`exporting = true`, `inert`, overlay);
2. verifica che le tile visibili siano caricate (`dataset.failed`);
3. costruisce uno *stage* fuori schermo (`left: -20000px`) e vi **ricopia** le tile come `<img>`
   con `crossOrigin='anonymous'`;
4. disegna strade e pin su un `<canvas>` sovrapposto, convertendo le coordinate con
   `map.latLngToContainerPoint()`; i pin sono ordinati per `y` per riprodurre la
   sovrapposizione di Leaflet;
5. aggiunge legenda, titolo e attribuzione OSM;
6. `html2canvas` → `toBlob` → download.

La mappa reale **non viene mai modificata**: si lavora su una copia. Ogni errore (tile mancanti,
CORS, timeout 15 s) produce un messaggio che rimanda all'export GeoJSON.

### GeoJSON

`geojson(includeRoads)` produce una `FeatureCollection` standard: edifici come `Point`, strade
come `LineString`, distinti da `properties.kind`. Include `map_settings` ed `exported_at`, ed è
leggibile da QGIS e simili.

`parseImport()` accetta **due formati**: lo stato nativo (`version: 1`) e la `FeatureCollection`
esportata. L'import **sostituisce** l'intero archivio, previa conferma esplicita
(`window.confirm`). Non esiste una fusione: è una scelta consapevole, perché unire due insiemi
di rilievi richiederebbe una politica di deduplicazione che l'utente non saprebbe governare.

## 8. Invarianti

Da preservare in qualsiasi modifica:

1. **Ogni mutazione di `state` è seguita da `saveState()`** e dal `render*()` pertinente.
2. **Nessun salvataggio quando `storageReady === false`** (non sovrascrivere dati corrotti).
3. **Testo utente nel DOM solo via `textContent`/`createTextNode`.**
4. **`[lat, lng]` internamente, `[lng, lat]` solo nel GeoJSON.**
5. **`exporting === true` blocca ogni interazione**: tutti gli handler iniziano con un guard.
6. **Gli export riflettono i dati, non lo stato di interazione** (selezione, anteprime).
7. **Le azioni distruttive offrono un annullamento** nel toast.
8. **Tutto deve restare eseguibile da `file://`**, senza fetch di risorse locali.

## 9. Insidie note

**Il colore di selezione non deve entrare nel PNG.** Funziona oggi perché `exportPNG()`
ridisegna le strade da zero con `COLORS.road` fisso, senza leggere `selectedRoadId`. Se l'export
venisse rifatto catturando il layer Leaflet, la strada selezionata uscirebbe viola
nell'immagine consegnata al cliente. È una regressione silenziosa: va verificata a vista.

**Leaflet usa il renderer canvas** (`preferCanvas: true`). Le polilinee **non sono nel DOM**:
non si possono ispezionare né testare con selettori CSS. Servono screenshot o l'API di Leaflet.

**Le animazioni sono disattivate** (`zoomAnimation`, `fadeAnimation`, `markerZoomAnimation`):
rendono affidabile la cattura delle tile nell'export. Non riattivarle a cuor leggero.

**Le tile di OpenStreetMap vanno richieste con un `Referer`.** La
[tile usage policy](https://operations.osmfoundation.org/policies/tiles/) chiede alle
applicazioni di identificarsi e vieta l'uso intensivo. Aprendo il programma da `file://` il
browser non invia alcun `Referer`, e `tile.openstreetmap.org` risponde con riquadri
*«Access blocked»* (HTTP 403) al posto della mappa. Servito da un sito (GitHub Pages) il
`Referer` c'è: da un test con `curl` i server rispondono con la mappa vera, ma la causa precisa
è un'ipotesi non ancora isolata (potrebbe contare anche lo `User-Agent`). Per non dipendere
dall'ipotesi il provider primario è un mirror che rende lo **stesso stile** e funziona in
entrambi i casi (`tile.openstreetmap.de`, poi `tile.openstreetmap.fr`, Esri per ultimo).
**Non passare a `tile.openstreetmap.org` come primario senza verificarlo anche da `file://`.**
Lo stile cartografico conta per l'utente, che riconosce il territorio dal disegno della mappa:
un ricambio deve somigliare all'originale, non solo funzionare. Un provider va scelto anche in
base a `Access-Control-Allow-Origin: *`, senza il quale l'export PNG smette di funzionare
(CARTO, per esempio, oggi richiede una API key e restituisce tile con la scritta
*API KEY REQUIRED*).

**Le categorie hanno più punti di dipendenza.** Aggiungerne una richiede di toccare CSS, form,
contatori, legenda, popup e lista (checklist in [CONTRIBUTING.md](CONTRIBUTING.md#aggiungere-una-categoria-di-edifici)).
Il contatore delle categorie è calcolato **per categoria**: in passato era "totale meno lavorati",
che con tre categorie avrebbe contato quella nuova come "simile". Non reintrodurre quella scorciatoia.

**Compatibilità dei backup.** I file vecchi (due categorie) si importano senza problemi nella
versione nuova. Il contrario no: un GeoJSON che contiene `known` **non si importa** in una copia
vecchia del programma, che risponde "Il backup contiene un edificio non valido". Chi riceve backup
da una versione più recente deve prima aggiornare `index.html`. Per questo non si è
alzato `version`: i dati vecchi restano validi.

**Non esiste la modifica di un edificio**: solo aggiunta ed eliminazione. È una lacuna reale,
non una scelta di design.

## 10. Glossario del dominio

- **Già lavorato** (`worked`, blu) — edificio su cui l'attività è stata svolta.
- **Supposto simile** (`similar`, arancione) — edificio ritenuto analogo, da verificare.
- **Di cui siamo a conoscenza** (`known`, verde) — edificio che conosciamo ma su cui non abbiamo
  lavorato né supposto un'analogia.
- **Strada percorsa** (`road`, rosso tratteggiato) — percorso effettivamente battuto durante il
  sopralluogo, tracciato a mano. Ogni tracciamento è un segmento indipendente: le strade non
  sono collegate tra loro e non hanno un ordine cronologico esplicito.

## 11. Pubblicazione

Il programma è servito da **GitHub Pages**: `https://paolo410.github.io/city-tracker/`, dalla
radice del ramo `main`. Un `git push` aggiorna il sito in circa un minuto. Non c'è altro processo.

- **L'origine cambia, quindi cambia l'archivio.** `localStorage` è legato a schema+host: i dati
  salvati aprendo il file da `file://` non compaiono sul sito. Si trasferiscono con
  esporta/importa GeoJSON.
- **Tutti i progetti sotto `paolo410.github.io` condividono lo stesso `localStorage`.** La chiave
  `mappa_edifici_v1` è abbastanza specifica da non collidere, ma qualunque altra pagina ospitata
  sotto lo stesso utente potrebbe leggere gli archivi. Non pubblicare pagine di terzi sotto lo
  stesso account.
- **La chiave di salvataggio conserva il vecchio nome** (il programma si chiamava «Mappa
  edifici»). Rinominarla svuoterebbe gli archivi già salvati.
- **Cosa lascia il browser:** il testo cercato e il centro mappa (verso Photon/komoot, solo su
  azione esplicita) e le richieste di tile (verso i server cartografici, che vedono IP e zona).
  Nessun dato inserito dall'utente arriva a un server dell'autore: non ne esiste uno.

## 12. Lingue (italiano e inglese)

I testi stanno tutti nel dizionario `I18N` (chiavi uguali in `it` ed `en`), all'inizio dello script.
`it` è la lingua predefinita e **coincide con il markup statico**, che è quello che vedono i motori di
ricerca e chi non ha JavaScript.

- **Testo statico**: attributi `data-i18n` (testo), `data-i18n-html` (testo con markup scritto dagli
  sviluppatori, per esempio un link) e `data-i18n-attr="attributo:chiave;…"` (placeholder, `aria-label`,
  `title`). `applyStaticLang()` li riscrive.
- **Testo dinamico** (messaggi, etichette, elenchi): `t('chiave', { variabili })`, e `tn('chiave', n)` per
  il plurale (chiavi `.one` / `.other`). Non comporre frasi concatenando pezzi: l'ordine delle parole
  cambia da lingua a lingua.
- **Cambio lingua**: `setLang()` aggiorna la lingua, salva la scelta in `localStorage` (chiave
  `city_tracker_lang`, **separata dall'archivio** e non inclusa negli export), aggiorna `?lang=` nell'indirizzo
  e chiama `applyLang()`, che ridisegna elenchi, legenda, stato di salvataggio e testata.
- **Priorità**: `?lang=` nell'indirizzo > scelta salvata > italiano. **Non si cambia lingua da soli in base
  al browser**: lo stesso indirizzo servirebbe lingue diverse a utenti e crawler. A chi ha un browser non
  italiano e non ha ancora scelto compare solo un suggerimento (`#lang-suggest`).
- **`storageMessage` contiene una chiave del dizionario**, non un testo, così il messaggio di avviso segue
  il cambio di lingua.
- Le **categorie** si salvano come chiave (`worked`, `similar`, `known`); l'etichetta tradotta compare solo
  in interfaccia e nel campo informativo `category_label` del GeoJSON (che l'import ignora).
- Formati: numeri e orari seguono `it-IT` o `en-GB`; la data nel PNG è `18/10/2026` in italiano e
  `18 Oct 2026` in inglese.

## 13. SEO e condivisione

Tutto il necessario sta nella testata di `index.html` e in due file accanto:

- **Titolo, descrizione, `canonical`, Open Graph, Twitter Card** in italiano nel markup statico; con
  `?lang=en` lo script li riscrive in inglese (`applyStaticLang()`), compreso il `canonical`.
- **`hreflang`** `it`, `en` e `x-default` (che punta all'italiano). La versione inglese è
  `…/?lang=en`: una variante per parametro, non una cartella separata. È una scelta dovuta al vincolo del
  file singolo; Google la gestisce se `canonical` e `hreflang` sono coerenti, ma una cartella `/en/` sarebbe
  più robusta (richiederebbe di duplicare o separare il codice).
- **Dati strutturati** JSON-LD `WebApplication` (nome, autore, licenza MIT, gratuito, lingue). Non ha
  valutazioni: non vanno inventate, e senza di esse Google non mostra snippet arricchiti.
- **`og-image.jpg`** (1200×630, circa 155 KB): anteprima dei link su WhatsApp, LinkedIn, Facebook. È un
  montaggio dell'export PNG reale dell'app con il marchio; rigenerarla se cambia l'aspetto dell'interfaccia.
- **`sitemap.xml`** con le due varianti e i loro `hreflang`. Da inviare in Google Search Console.
- **Testo indicizzabile**: la sezione «05 Informazioni» e il blocco `<noscript>` (in entrambe le lingue).
- **`robots.txt` non è utile qui**: un sito di progetto GitHub Pages vive sotto
  `paolo410.github.io/city-tracker/`, e i crawler leggono solo `robots.txt` alla radice del dominio,
  che non controlliamo. Si usa `<meta name="robots">`.
- Se cambia l'indirizzo del sito, aggiornare `canonical`, `hreflang`, `og:url`, `og:image`, JSON-LD,
  `sitemap.xml`, `CITATION.cff` e `README.md`.
