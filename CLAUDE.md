# Istruzioni per assistenti AI

Contesto essenziale per lavorare su questo progetto. Approfondimenti in
[ARCHITECTURE.md](ARCHITECTURE.md) e [CONTRIBUTING.md](CONTRIBUTING.md).

## In breve

`index.html` **è** il programma: un unico file con markup, CSS e JavaScript vanilla in
una IIFE. Censisce edifici sul territorio e traccia le strade percorse nei sopralluoghi.
L'utente finale non è tecnico: apre il sito e lavora. Servito da GitHub Pages
(`https://paolo410.github.io/city-tracker/`); funziona anche aperto da `file://`.

## Regole non negoziabili

1. **Non introdurre build step, framework, moduli ES o dipendenze locali.** Il file deve
   restare apribile da `file://`. Niente `npm`, niente `import`.
2. **Non aggiungere un backend** né inviare dati a servizi esterni. L'unica richiesta con dati
   dell'utente è la geocodifica, su azione esplicita.
3. **Testo dell'utente nel DOM solo via `textContent` o `createTextNode`**, mai `innerHTML`:
   è la difesa XSS del programma.
4. **Ogni mutazione di `state` va seguita da `saveState()`** e dal `render*()` pertinente.
5. **Non salvare quando `storageReady === false`**: serve a non sovrascrivere un archivio
   corrotto e a permetterne il recupero.
6. **Internamente `[lat, lng]`, nel GeoJSON `[lng, lat]`.** Confondere l'ordine sposta i punti
   dall'altra parte del mondo, spesso senza errori evidenti.
7. **Gli export riflettono i dati, non lo stato di interazione.** L'evidenziazione viola di una
   strada selezionata non deve mai finire in un PNG.
8. **Interfaccia bilingue (it/en) tramite il dizionario `I18N`**: nessuna stringa visibile letterale nel
   codice o nel markup; ogni chiave esiste in entrambe le lingue. I commenti nel codice restano in italiano.
   Il markup statico è in italiano e coincide con il dizionario `it` (è ciò che indicizzano i motori).
9. **Non usare `tile.openstreetmap.org` come fonte primaria delle tile** senza verificarlo anche
   da `file://`: senza `Referer` risponde `403 Access blocked`. Ogni provider alternativo deve
   esporre `Access-Control-Allow-Origin: *`, altrimenti l'export PNG si rompe.
10. **Non toccare la chiave `mappa_edifici_v1`** di `localStorage` (nome storico): cambiarla
    svuota gli archivi già salvati dagli utenti.
11. **Se si cambia la versione di una libreria CDN, rigenerare l'hash `integrity`** nel tag:
    senza, o con un hash sbagliato, il browser blocca la libreria.
12. **Non cambiare lingua automaticamente in base al browser.** Lo stesso indirizzo servirebbe lingue
    diverse a utenti e crawler. La lingua è: `?lang=` > scelta salvata > italiano; ai browser non italiani
    si fa solo un suggerimento. `canonical` e `hreflang` devono restare coerenti con questa regola.
13. **Le categorie si salvano come chiave** (`worked`, `similar`, `known`), mai come etichetta tradotta:
    i backup devono importarsi in qualsiasi lingua.

## Prima di modificare

- Leggere le [insidie note](ARCHITECTURE.md#9-insidie-note): diverse cose che sembrano bug o
  sviste sono scelte deliberate (animazioni disattivate, renderer canvas, import che sostituisce
  invece di fondere).
- Le polilinee sono disegnate su **canvas**, non nel DOM: non sono ispezionabili con selettori
  CSS. Per verificarle servono screenshot o l'API di Leaflet.
- Le funzioni `render*()` ricostruiscono tutto dallo stato: non c'è reattività automatica.

## Dopo aver modificato

Verificare davvero, aprendo il programma nel browser — non limitarsi a dedurre dal codice.
Il percorso di collaudo è in [CONTRIBUTING.md](CONTRIBUTING.md#collaudo). In particolare:
persistenza dopo ricarica, ed export PNG con le strade **rosse tratteggiate**.

Se la modifica è visibile all'utente, aggiornare `README.md`. Il sito si aggiorna da solo dopo
il `git push` su `main`.

## Contesto utile per rispondere all'utente

- I dati vivono in `localStorage`, legato al browser e all'origine: non viaggiano con il file
  copiato altrove. Il GeoJSON è l'unico backup affidabile, e va raccomandato attivamente.
