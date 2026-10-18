# City Tracker

**▶ Usalo online: <https://paolo410.github.io/city-tracker/>** · 🇬🇧 [English version](https://paolo410.github.io/city-tracker/?lang=en)

Strumento web per censire edifici su mappa e tracciare i percorsi dei sopralluoghi. Gratuito,
senza account e senza installazione: i dati restano nel tuo browser.

## Funzionalità

- **Edifici** con indirizzo, note e una di tre categorie: già lavorato (blu), supposto simile
  (arancione), di cui siamo a conoscenza (verde)
- **Posizionamento** con la ricerca dell'indirizzo, con le coordinate o con un clic sulla mappa
- **Strade percorse**: tracciamento a più segmenti, selezione dall'elenco o dalla mappa (la
  strada scelta diventa viola) ed eliminazione singola con possibilità di annullare
- **Esportazione** in PNG (con legenda e attribuzione cartografica) e in GeoJSON; **importazione**
  di backup GeoJSON
- **Salvataggio automatico** nel browser
- **Italiano e inglese**: il selettore IT / EN è in alto nella barra laterale

## English

**City Tracker** is a free web tool to map buildings and track the routes you walk on site visits, in any
city. It runs entirely in your browser: no account, no installation, no server. Your data (addresses,
notes, roads) never leaves your browser.

▶ **Open it: <https://paolo410.github.io/city-tracker/?lang=en>**

- Add buildings by searching an address, entering coordinates or clicking the map, in one of three
  categories (already worked, presumed similar, known to us).
- Trace roads with multiple segments, select one in the list or on the map and delete it with undo.
- Export a PNG (with legend and map attribution) or a GeoJSON; import a GeoJSON backup.
- **Using it outside Turin:** the map opens on Turin. Open *04 Map settings*, enter the latitude, longitude
  and zoom of your city and press *Apply and centre the map*. Address search favours results near that
  centre, so it works worldwide. Switch language with the **IT / EN** buttons at the top of the sidebar.
- **Back up your data:** everything is saved in this browser only. Clearing site data, switching browser or
  using a private window loses it. Export a GeoJSON regularly.

## Come si usa

1. **Aggiungi un edificio.** Inserisci indirizzo, categoria e note. Con le coordinate vuote il
   programma cerca l'indirizzo: scegli il risultato corretto. In alternativa premi «Scegli il
   punto sulla mappa» oppure inserisci latitudine e longitudine.
2. **Traccia una strada.** Premi «Inizia tracciamento», clicca almeno due punti e premi «Fine
   tracciamento». Per eliminarne una, cliccala (nell'elenco o sulla mappa): diventa viola, poi
   usa il cestino. Dopo l'eliminazione hai qualche secondo per annullare.
3. **Esporta.** Il PNG è l'immagine della mappa; il GeoJSON contiene i dati e serve come backup.

### Usarlo in un'altra città

All'apertura la mappa è centrata su Torino. Per cambiare città apri la sezione **04 Impostazioni
mappa**, inserisci latitudine, longitudine e zoom e premi «Applica e centra la mappa»: la scelta
viene salvata. La ricerca degli indirizzi dà la precedenza ai risultati vicini al centro
impostato, quindi funziona in qualsiasi città.

## I tuoi dati

I dati si salvano da soli nel browser (`localStorage`), a ogni modifica. Non esiste un
pulsante "Salva" e non esiste un server dove i dati vengano copiati.

⚠️ **Il salvataggio è legato al browser.** Cambiare browser, usare la navigazione in incognito o
cancellare i dati dei siti fa perdere l'archivio. **Esporta un GeoJSON a fine giornata**: è
l'unico backup che sopravvive a tutto, ed è anche il modo per passare il lavoro a un altro
computer. «Importa backup GeoJSON» sostituisce l'archivio corrente, dopo conferma.

Se prima usavi una copia del file salvata sul computer, i suoi dati **non compaiono qui**
(il browser li tiene separati per indirizzo): esportali da lì in GeoJSON e importali qui.

### Cosa viene inviato a terzi

- Cercando un indirizzo: il testo cercato e il centro della mappa vanno a
  [Photon / komoot](https://photon.komoot.io/). Solo quando premi «Aggiungi edificio» senza
  coordinate.
- Aprendo la mappa: le immagini della cartografia vengono richieste ai server cartografici, che
  vedono il tuo indirizzo IP e la zona visualizzata.
- Indirizzi, note e strade non lasciano mai il tuo browser.

Serve una connessione a Internet per cartografia, librerie e ricerca degli indirizzi.

## Tecnologie

JavaScript vanilla, HTML5 e CSS in un unico file (`index.html`), senza framework e senza build.
[Leaflet 1.9.4](https://leafletjs.com/) per la mappa, [html2canvas
1.4.1](https://html2canvas.hertzen.com/) per l'export PNG, cartografia
[OpenStreetMap](https://www.openstreetmap.org/copyright) (tile FOSSGIS e.V., con ricambio
automatico su altri server), geocodifica [Photon / komoot](https://photon.komoot.io/), scambio
dati in [GeoJSON](https://geojson.org/). Le librerie sono caricate da CDN con controllo di
integrità (SRI).

Architettura, modello dati e decisioni di progetto: [ARCHITECTURE.md](ARCHITECTURE.md). Per
modificarlo: [CONTRIBUTING.md](CONTRIBUTING.md). Per assistenti AI: [CLAUDE.md](CLAUDE.md).

## Come citare

> Malugani, P. (2026). *City Tracker* [Software]. https://paolo410.github.io/city-tracker/

```bibtex
@software{malugani2026citytracker,
  author = {Malugani, Paolo},
  title  = {City Tracker},
  year   = {2026},
  url    = {https://paolo410.github.io/city-tracker/}
}
```

GitHub mostra anche il pulsante «Cite this repository», generato da [CITATION.cff](CITATION.cff).

## Licenza

[MIT](LICENSE) © 2026 Paolo Malugani. La cartografia è © OpenStreetMap contributors (ODbL):
**l'attribuzione presente nell'app e nelle immagini esportate è un requisito di licenza e non va
rimossa.**
