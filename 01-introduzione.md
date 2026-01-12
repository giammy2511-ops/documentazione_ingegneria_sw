# 1. Introduzione generale

## 1.1 Scopo dell’applicazione
L’applicazione supporta la gestione operativa di **visite guidate** organizzate su un territorio (ambito territoriale), con:
- definizione di **luoghi** visitabili;
- definizione di **tipi di visita** associati ai luoghi (periodo, giorni della settimana, orari, durata, capienze, ecc.);
- gestione di **volontari** (guide) e delle loro disponibilità mensili;
- produzione mensile di un **piano di visite proposte** (istanze) per il mese di riferimento;
- prenotazioni da parte di **fruitori** con codice univoco e possibilità di disdetta entro una finestra temporale.

## 1.2 Ambito di utilizzo
Applicazione pensata per un’organizzazione che pianifica visite mensili:
- inserimento e manutenzione del catalogo dati (luoghi, tipi di visita, guide);
- raccolta disponibilità guide e **generazione automatica** del piano mensile;
- gestione prenotazioni e loro evoluzione di stato (proposta → confermata/confermata → effettuata).

## 1.3 Ruoli supportati e responsabilità

### Configuratore
È l’utente “amministrativo” che:
- inizializza il sistema (dati generali, primi luoghi e visite);
- crea/gestisce luoghi, tipi di visita, volontari e associazioni;
- gestisce date precluse e vincoli operativi;
- chiude la raccolta disponibilità e genera il piano mensile;
- esegue modifiche con **rimozioni a cascata** secondo la logica implementata.

### Volontario
È la guida che:
- accede con credenziali comunicate (password predefinita) e al **primo accesso** deve cambiarla;
- consulta i tipi di visita a cui è associato;
- inserisce disponibilità **per date** (non per fasce orarie) nel mese “target” indicato dal sistema;
- consulta l’elenco delle proprie visite dal piano (vedi nota sui limiti in §7).

### Fruitore
È l’utente che prenota:
- si registra e poi accede;
- consulta visite proposte/confermate/cancellate;
- si iscrive a una visita proposta (anche per più persone) entro limiti e capienza;
- riceve un **codice di prenotazione** univoco;
- può disdire entro la finestra temporale consentita.

## 1.4 Architettura generale

### Applicazione Java stand‑alone
- Nessuna GUI: interazione **testuale** su console tramite menù.
- Avvio da `Main` e navigazione gerarchica tramite classi `MenuConfiguratore`, `MenuVolontario`, `MenuFruitore`.

### Persistenza su file
- La persistenza avviene su file in una cartella dati (di default `./DATA`).
- Si usa Jackson (JSON) per la maggior parte degli oggetti; alcuni file sono testuali (es. date precluse).

### Flusso di controllo
- `Main` carica i dati con `GestoreDati.caricaDaDirectory("./DATA")` e apre un menu di benvenuto (selezione ruolo).
- `GestoreDati` è il “servizio” centrale: autentica utenti, applica vincoli, salva/carica file, produce il piano, aggiorna stati delle istanze, esegue rimozioni a cascata.
- `FileIO` incapsula la lettura/scrittura dei file persistenti.
