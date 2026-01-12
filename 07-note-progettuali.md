# 7. Note progettuali

## 7.1 Scelte interpretative rispetto ai requisiti
1. **Algoritmo di scheduling**: implementato come scelta greedy “prima guida disponibile” e “una visita per tipo per giorno”; non ottimizza copertura né bilanciamento.
2. **Disponibilità del volontario**: per **date** (non per orari), come da requisiti; ma la pianificazione limita implicitamente a **una visita al giorno** per guida (semplificazione extra).
3. **Evoluzione stati**: gestita in modo “pull” (aggiornata a runtime durante operazioni) e non come processo autonomo.
4. **Cascate di rimozione**: la chiusura transitiva rimuove dati “orfani” in modo ripetuto; è coerente con la pulizia dati ma può risultare più distruttiva di quanto un utente si aspetti.

## 7.2 Limiti dell’implementazione (da dichiarare in consegna)
- **Vista volontario delle visite confermate**: il filtro stati nella stampa per volontario appare errato e potrebbe escludere le confermate (vedi §5.6).
- **Assenza di bootstrap utenti**: il codice non crea automaticamente un configuratore iniziale; l’avvio richiede dati iniziali (o modifica manuale di `credenziali.json`).
- **Dipendenza forte dall’orologio di sistema**: finestre 16/15, chiusure e cambi stato dipendono da `LocalDate.now()`.
- **Persistenza mista**: alcuni file sono JSON, altri sono testo; la coerenza è garantita dal codice, ma modifiche manuali possono rompere parsing e invarianti.
- **Nessuna gestione concorrente**: l’app è single‑user da console; nessun lock sui file.

## 7.3 Coerenza con processo incrementale
La struttura del codice mostra chiaramente evoluzioni:
- v2: introduzione volontari, disponibilità, primo accesso;
- v3: introduzione stato sistema, piano istanze, archivio storico istanze, gestione cascata e produzione mensile.

## 7.4 Possibili estensioni future (coerenti con l’architettura)
- Correggere/estendere la vista volontario per includere `confermata` e mostrare i codici prenotazione aggregati.
- Introdurre un algoritmo di scheduling che:
  - massimizzi numero visite realizzabili;
  - bilanci carichi sui volontari;
  - consenta più visite al giorno se non sovrapposte temporalmente.
- Migliorare la gestione credenziali (hash password, reset, politiche di robustezza).
- Aggiungere comandi “report” (es. statistiche iscrizioni, no‑show, ecc.).
- Migrare a un formato persistenza uniforme (tutto JSON) e validazione schema.

## 7.5 Checklist rapida per consegna universitaria
- Descrivere chiaramente il comportamento “visto da utente” con esempi console.
- Evidenziare dove il codice **diverge** dai requisiti (bug o interpretazioni).
- Allegare/descrivere la struttura della cartella `DATA` e i file prodotti.
