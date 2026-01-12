# A1. Persistenza e formato dei dati

Questa appendice descrive i file che `FileIO` gestisce nella cartella dati (default `./DATA`).

## A1.1 Elenco file

### JSON (Jackson)
- `credenziali.json` – lista di `Utente`
- `luoghi.json` – lista di `Luogo`
- `visite.json` – lista di `Visita` (tipi visita attivi)
- `statoSistema.json` – `StatoSistema`
- `visiteIstanze.json` – lista di `VisitaIstanza` (piano)
- `archivioStorico.json` – `ArchivioStorico` (tipi scaduti + istanze effettuate)

### Testo
- `generale.txt` – 1 riga: ambito territoriale; 2 riga: max persone iscrivibili da fruitore
- `datePrecluse.txt` – una data per riga in formato ISO `YYYY-MM-DD`
- `disponibilitaVolontari.txt` – formato testuale con una riga per volontario (vedi A1.3)

## A1.2 Note sul salvataggio e compatibilità
- Jackson è configurato con `JavaTimeModule` e scrittura date ISO (no timestamps).
- Se un file JSON non esiste o è vuoto, i metodi di lettura di solito restituiscono strutture vuote (non `null`), tranne i casi documentati (dati generali/stato).

## A1.3 Formato `disponibilitaVolontari.txt`
Il file contiene una rappresentazione “manuale” della mappa:
- chiave = nickname volontario
- valori = insieme di date ISO

Il codice salva in ordine alfabetico per nickname. Il parsing è permissivo, ma il formato atteso è coerente con quello prodotto dal programma.

## A1.4 Pulizia dati dopo la produzione del piano
Dopo aver prodotto il piano per `meseRaccolta`, il sistema rimuove dai file:
- disponibilità con date fino a quel mese (incluso);
- date precluse fino a quel mese (incluso).

Questo evita che dati vecchi interferiscano con piani futuri.

## A1.5 Archivio storico
`archivioStorico.json` contiene:
- `tipiVisitaScaduti`: tipi visita non più attivi (in base alla data fine rispetto a “oggi”);
- `istanzeEffettuate`: visite confermate che, dopo la data, sono marcate `effettuata` e spostate in archivio.


