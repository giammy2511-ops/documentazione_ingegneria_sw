# Attori e confini del sistema

## Attori (ruoli reali)
- **Configuratore**: configura luoghi e tipi di visita; gestisce date precluse; governa ciclo mensile (chiusura raccolta, produzione piano, richieste di modifica a cascata); consulta elenchi/stati/piano/archivio.
- **Volontario**: consulta tipi visita associati; inserisce disponibilità nel mese di raccolta (se aperta); consulta piano visite assegnate.
- **Fruitore**: si registra/accede; consulta visite; si iscrive a visite proposte; consulta “le mie visite”; disdice iscrizione tramite codice entro i termini.

## Attori secondari / sistemi esterni (se presenti)
- **File system (directory `./DATA`)**: persistenza di utenti, luoghi, tipi visita, stato sistema, date precluse, disponibilità, piano visite, archivio storico (classe `FileIO`).

## Confini del sistema
**Dentro**:
- Applicazione Java **CLI/console** (`Main`, `MyMenu`, `InputDati`), logica applicativa (`GestoreDati`, `GestoreCalendario`), modello dati (`Luogo`, `Visita`, `VisitaIstanza`, `Iscrizione`, `StatoSistema`, `ArchivioStorico`), persistenza (`FileIO`).

**Fuori**:
- Utenti umani (configuratore/volontario/fruitore).
- File system (lettura/scrittura su `./DATA`).

---

# Elenco casi d’uso

| ID   | Titolo | Attore primario | Priorità | Stato |
|------|--------|------------------|----------|-------|
| UC01 | Accedere al sistema come configuratore/volontario | Configuratore / Volontario | Alta | da codice |
| UC02 | Cambiare credenziali al primo accesso | Configuratore / Volontario | Media | da codice |
| UC03 | Accedere o registrarsi come fruitore | Fruitore | Alta | da codice |
| UC04 | Inizializzare dati generali del sistema | Configuratore | Alta | da codice |
| UC05 | Creare luogo con prima visita obbligatoria | Configuratore | Alta | da codice |
| UC06 | Creare tipo visita e associarlo a luogo (esistente o nuovo) | Configuratore | Alta | da codice |
| UC07 | Inserire e consultare date precluse | Configuratore | Media | da codice |
| UC08 | Modificare numero massimo iscrivibili da un fruitore | Configuratore | Media | da codice |
| UC09 | Visualizzare tipi visita associati | Volontario | Media | da codice |
| UC10 | Inserire disponibilità nel mese di raccolta | Volontario | Alta | da codice |
| UC11 | Chiudere raccolta disponibilità del mese | Configuratore | Alta | da codice |
| UC12 | Produrre piano visite del mese | Configuratore | Alta | da codice |
| UC13 | Visualizzare visite (per fruitore o globali) | Fruitore | Alta | da codice |
| UC14 | Iscriversi a una visita proposta | Fruitore | Alta | da codice |
| UC15 | Disdire iscrizione tramite codice | Fruitore | Media | da codice |
| UC16 | Consultare stato visite e piano prodotto | Configuratore | Media | da codice |
| UC17 | Rimuovere luogo/tipo visita/volontario con cascata | Configuratore | Media | da codice |
| UC18 | Visualizzare piano visite associate al volontario | Volontario | Media | da codice |
| UC19 | Visualizzare archivio storico | Configuratore | Bassa | da codice |

---

## UC01 — Accedere al sistema come configuratore/volontario

**Attori primari:** Configuratore, Volontario  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** autenticarsi come utente “configuratore” o “volontario” ed entrare nel menu coerente col ruolo.  
**Precondizioni:** esiste un utente in `credenziali.json` con ruolo supportato; per il volontario: utente **attivo**.  
**Postcondizioni (successo):** utente autenticato; se necessario avviato cambio credenziali; avviato menu di ruolo.  
**Postcondizioni (fallimento):** nessun accesso; richiesta credenziali ripetuta.  
**Trigger:** selezione “Accedi come configuratore/volontario” nel menu “Benvenuto”.

### Flusso principale
1. L’attore seleziona “Accedi come configuratore/volontario”.
2. Il sistema chiede **nome utente** e **password**.
3. Il sistema valida credenziali cercando corrispondenza tra username/password e ruolo supportato.
4. Se l’utente è un **volontario non attivo**, il sistema nega l’accesso.
5. In caso di successo, il sistema verifica se l’utente è al **primo accesso**.
6. Il sistema (se necessario) avvia la procedura di cambio credenziali (UC02).
7. Il sistema verifica/presenta i dati generali (UC04) e avvia il menu del ruolo.

### Flussi alternativi / eccezioni
- **A1 — Credenziali errate o utente non attivo**:  
  1. Condizione: username/password non trovati **oppure** volontario `attivo=false`.  
  2. Passi: il sistema stampa messaggio di errore e richiede nuovamente le credenziali.  
  3. Esito: accesso non effettuato finché non vengono inserite credenziali valide.

- **A2 — Errore I/O in avvio**:  
  1. Condizione: errore durante `GestoreDati.caricaDaDirectory("./DATA")`.  
  2. Passi: il sistema stampa stack trace.  
  3. Esito: avvio degradato/terminazione anomala.

### Regole di business / vincoli
- RB1: un volontario può autenticarsi solo se `attivo=true` (volontari disattivati automaticamente se non associati a visite attive).

### Dati coinvolti
- Input: username, password  
- Output: menu di ruolo avviato / messaggio errore  
- Entità/modelli: `Utente`, `GestoreDati`, `FileIO`

### Tracciabilità
- **Codice:** `Main.menuBenvenuto()`, `Main.login()`, `Main.verificaCredenziali()`, `GestoreDati.autentica()`  
- **Requisiti/slide:**  
  > Nota: non forniti; non è possibile tracciare a requisiti esterni.

---

## UC02 — Cambiare credenziali al primo accesso

**Attori primari:** Configuratore, Volontario  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** sostituire le credenziali iniziali, disattivando lo stato “primo accesso”.  
**Precondizioni:** utente autenticato; `utente.primoAccesso=true`.  
**Postcondizioni (successo):** credenziali aggiornate e salvate; `primoAccesso=false`.  
**Postcondizioni (fallimento):** credenziali non modificate; utente resta in primo accesso.  
**Trigger:** subito dopo login riuscito se `isPrimoAccesso()`.

### Flusso principale
1. Il sistema chiede le nuove credenziali.
2. **Se l’attore è volontario**, il sistema non permette di cambiare username (mantiene quello attuale).
3. Il sistema chiede una nuova password.
4. Il sistema valida che la nuova password sia non vuota e diversa dalla precedente.
5. Il sistema aggiorna l’oggetto `Utente` e salva su file.
6. Il sistema conferma l’avvenuto aggiornamento.

### Flussi alternativi / eccezioni
- **A1 — Username già esistente (non volontario)**:  
  1. Condizione: l’attore (non volontario) inserisce uno username già presente.  
  2. Passi: il sistema segnala errore e richiede reinserimento.  
  3. Esito: aggiornamento non eseguito finché non valido.

- **A2 — Password uguale alla precedente**:  
  1. Condizione: nuova password = vecchia password.  
  2. Passi: il sistema richiede una password diversa.  
  3. Esito: aggiornamento sospeso finché non valido.

- **A3 — Errore di persistenza**:  
  1. Condizione: `FileIO.salvaUtenti()` fallisce.  
  2. Passi: il sistema stampa errore/stack trace.  
  3. Esito: aggiornamento fallito.

### Regole di business / vincoli
- RB1: i volontari **non possono** cambiare username, solo password.
- RB2: password non vuota e diversa dalla precedente.

### Dati coinvolti
- Input: (eventuale) nuovo username, nuova password  
- Output: conferma o errore  
- Entità/modelli: `Utente`, `GestoreDati.aggiornaCredenziali()`

### Tracciabilità
- **Codice:** `Main.cambioCredenziali()`, `GestoreDati.aggiornaCredenziali()`  
- **Requisiti/slide:**  
  > Nota: non forniti.

---

## UC03 — Accedere o registrarsi come fruitore

**Attori primari:** Fruitore  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** consentire al fruitore di entrare nel sistema con credenziali esistenti o crearne di nuove.  
**Precondizioni:** sistema avviato.  
**Postcondizioni (successo):** fruitore autenticato o creato e salvato; avvio menu fruitore.  
**Postcondizioni (fallimento):** nessun accesso; ritorno al menu di accesso fruitore o annullamento.  
**Trigger:** selezione “Accedi come fruitore” nel menu “Benvenuto”.

### Flusso principale (Accesso)
1. L’attore sceglie “Accedi”.
2. Il sistema richiede username e password.
3. Il sistema autentica e verifica che il ruolo sia `fruitore`.
4. Il sistema avvia il menu fruitore.

### Flussi alternativi / eccezioni
- **A1 — Credenziali errate o ruolo diverso**:  
  1. Condizione: autenticazione fallisce o utente non ha ruolo `fruitore`.  
  2. Passi: il sistema mostra messaggio e propone di riprovare o registrarsi.  
  3. Esito: accesso non effettuato.

- **A2 — Registrazione con username già esistente**:  
  1. Condizione: username scelto è già presente.  
  2. Passi: il sistema propone di reinserire username o tornare indietro.  
  3. Esito: registrazione non completata finché non valido.

- **A3 — Errore di salvataggio in registrazione**:  
  1. Condizione: fallisce `GestoreDati.creaFruitore()` / `FileIO.salvaUtenti()`.  
  2. Passi: il sistema mostra errore.  
  3. Esito: utente non creato.

### Regole di business / vincoli
- RB1: username fruitore deve essere univoco (vincolo su `utenti`).

### Dati coinvolti
- Input: username, password  
- Output: menu fruitore / messaggi di errore  
- Entità/modelli: `Utente`, `GestoreDati.autentica()`, `GestoreDati.creaFruitore()`

### Tracciabilità
- **Codice:** `Main.loginFruitore()`, `MenuFruitore.accessoORRegistrazione()`, `GestoreDati.creaFruitore()`  
- **Requisiti/slide:**  
  > Nota: non forniti.

---

## UC04 — Inizializzare dati generali del sistema

**Attori primari:** Configuratore  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** impostare ambito territoriale e numero massimo di persone iscrivibili per singolo fruitore.  
**Precondizioni:** autenticazione effettuata; `generale.txt` mancante o vuoto.  
**Postcondizioni (successo):** dati generali salvati su file.  
**Postcondizioni (fallimento):** dati generali non disponibili; operazioni dipendenti (es. iscrizione) possono fallire.  
**Trigger:** accesso al sistema quando `FileIO.leggiDatiGenerali()` restituisce `null`.

### Flusso principale
1. Il sistema tenta di leggere i dati generali.
2. Se assenti, il sistema verifica che l’utente corrente sia `configuratore`.
3. Il sistema chiede **ambito territoriale**.
4. Il sistema chiede **numero massimo iscrivibili da fruitore**.
5. Il sistema salva i dati generali su `generale.txt`.

### Flussi alternativi / eccezioni
- **A1 — Utente non configuratore e dati mancanti**:  
  1. Condizione: dati generali assenti e utente non è configuratore.  
  2. Passi: il sistema informa e riporta al menu di benvenuto.  
  3. Esito: dati non inizializzati.

- **A2 — Errore di persistenza**:  
  1. Condizione: fallisce `FileIO.salvaDatiGenerali()`.  
  2. Passi: stampa errore/stack trace.  
  3. Esito: dati non salvati.

### Regole di business / vincoli
- RB1: solo il configuratore può inizializzare il sistema se mancano i dati generali.
- RB2: il max iscrivibili limita `MenuFruitore.menuIscrizione()`.

### Dati coinvolti
- Input: `ambitoTerritoriale`, `maxPersoneIscrivibiliDaFruitore`  
- Output: dati salvati; stampa riepilogo se presenti  
- Entità/modelli: `DatiGenerali`, `FileIO`

### Tracciabilità
- **Codice:** `Main.chiediDatiGenerali()`, `FileIO.leggiDatiGenerali()`, `FileIO.salvaDatiGenerali()`  
- **Requisiti/slide:**  
  > Nota: non forniti.

---

## UC05 — Creare luogo con prima visita obbligatoria

**Attori primari:** Configuratore  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** creare un nuovo luogo visitabile e associargli almeno un tipo visita (vincolo di setup).  
**Precondizioni:** configuratore autenticato.  
**Postcondizioni (successo):** luogo creato e salvato; prima visita creata, validata (no overlap) e salvata.  
**Postcondizioni (fallimento):** se la visita non viene completata, il luogo appena creato viene rimosso e non resta nel sistema.  
**Trigger:** menu configuratore → “Setup iniziale” → “Crea luogo + prima visita (obbligatoria)” **oppure** menu modifiche.

### Flusso principale
1. L’attore seleziona “Crea luogo + prima visita”.
2. Il sistema chiede: nome, descrizione, latitudine, longitudine.
3. Il sistema genera ID luogo (derivato da nome+coordinate) e verifica che non esista già un luogo con stesso ID.
4. Il sistema salva il luogo.
5. Il sistema avvia la creazione della prima visita (UC06: inserimento dati visita, scelta volontari, scheduling).
6. Il sistema prova ad aggiungere la visita tramite `GestoreCalendario.aggiungiVisita()` (vincolo anti-overlap nello stesso luogo).
7. Se non ci sono sovrapposizioni, il sistema salva visita e aggiorna persistenza.
8. Il sistema conferma creazione luogo+visita.

### Flussi alternativi / eccezioni
- **A1 — Luogo duplicato (stesso nome+coordinate)**:  
  1. Condizione: ID già presente tra i luoghi.  
  2. Passi: il sistema chiede di reinserire i dati del luogo.  
  3. Esito: luogo non creato finché non univoco.

- **A2 — Overlap tra visite nello stesso luogo**:  
  1. Condizione: `GestoreCalendario.aggiungiVisita()` restituisce `false`.  
  2. Passi: il sistema mostra “OVERLAP rilevato” e permette di reinserire **solo scheduling** (date/ora/durata/giorni) oppure annullare.  
  3. Esito: se si corregge scheduling, riprova; se si annulla, il luogo viene rimosso.

- **A3 — Annullamento durante creazione visita**:  
  1. Condizione: la visita risulta `null` (nel codice esiste ramo che gestisce questa possibilità).  
  2. Passi: il sistema rimuove il luogo appena creato e salva i luoghi.  
  3. Esito: nessuna creazione completata.  
  > Nota: nel codice attuale `creaVisita()` non ritorna mai `null`; questo ramo sembra “difensivo” ma non è attivabile dai flussi visibili.

- **A4 — Errore di persistenza**:  
  1. Condizione: fallisce `FileIO.salvaLuoghi()` o `FileIO.salvaVisite()`.  
  2. Passi: stampa stack trace.  
  3. Esito: creazione parziale o fallita.

### Regole di business / vincoli
- RB1: un luogo è “valido” nel setup solo se ha almeno una visita associata.
- RB2: ID luogo derivato da nome+coordinate; duplicati non ammessi.

### Dati coinvolti
- Input: dati luogo; dati visita (titolo, descrizione, punto incontro, min/max, biglietto, volontari, scheduling)  
- Output: conferma o messaggi di correzione/errore  
- Entità/modelli: `Luogo`, `Visita`, `PuntoIncontro`, `SchedulingVisita`, `GiorniSettimana`

### Tracciabilità
- **Codice:** `MenuConfiguratore.creaLuogoConVisitaObbligatoria()`, `MenuConfiguratore.creaLuogo()`, `GestoreDati.aggiungiLuogo()`, `GestoreDati.aggiungiVisitaALuogo()`, `GestoreCalendario.aggiungiVisita()`  
- **Requisiti/slide:**  
  > Nota: non forniti.

---

## UC06 — Creare tipo visita e associarlo a luogo (esistente o nuovo)

**Attori primari:** Configuratore  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** creare un “tipo visita” programmabile e associarlo a un luogo esistente o nuovo, rispettando vincoli (titolo univoco nel luogo, almeno un volontario, no overlap).  
**Precondizioni:** configuratore autenticato; esistono volontari oppure è possibile crearne al volo.  
**Postcondizioni (successo):** visita creata e salvata; associata al luogo; sistema sincronizzato.  
**Postcondizioni (fallimento):** visita non inserita; nessuna modifica (o solo luogo nuovo se creato tramite UC05).  
**Trigger:** menu configuratore → setup/modifiche → “Crea visita (associa subito…)” / “Aggiungi tipo visita”.

### Flusso principale
1. L’attore sceglie se associare a luogo esistente o creare un nuovo luogo.
2. Se luogo esistente: il sistema mostra lista luoghi e l’attore ne seleziona uno.
3. Il sistema chiede i dati del tipo visita:
   1. Titolo (vincolo: unico nel luogo).
   2. Descrizione.
   3. Punto di incontro (via/piazza, civico, descrizione).
   4. Min partecipanti, Max partecipanti (max ≥ min).
   5. Flag “biglietto acquistabile”.
   6. Selezione volontari (almeno uno), con opzione “aggiungi nuovo volontario”.
   7. Scheduling: data inizio/fine, durata, ora, giorni della settimana (almeno uno tra quelli nel range).
4. Il sistema tenta di aggiungere il tipo visita verificando overlap nel luogo.
5. Se valido, salva visite e luoghi.

### Flussi alternativi / eccezioni
- **A1 — Titolo già usato nel luogo**:  
  1. Condizione: esiste già visita con stesso titolo nel luogo (case-insensitive).  
  2. Passi: il sistema richiede reinserimento titolo.  
  3. Esito: prosegue solo con titolo univoco.

- **A2 — Nessun volontario selezionato**:  
  1. Condizione: l’attore prova a terminare selezione senza aver scelto nessuno.  
  2. Passi: il sistema impone “Devi selezionare almeno un volontario”.  
  3. Esito: obbligo di selezione minima.

- **A3 — Nessun giorno selezionato**:  
  1. Condizione: l’attore termina scelta giorni senza averne selezionato alcuno.  
  2. Passi: il sistema impone “Devi selezionare almeno un giorno”.  
  3. Esito: obbligo di selezione minima.

- **A4 — Overlap nello stesso luogo**:  
  1. Condizione: `GestoreCalendario.aggiungiVisita()` fallisce.  
  2. Passi: il sistema propone reinserimento del solo scheduling o annullamento.  
  3. Esito: visita inserita solo dopo scheduling compatibile.

- **A5 — Creazione “nuovo volontario” con username già esistente**:  
  1. Condizione: username già presente.  
  2. Passi: il sistema chiede un username diverso.  
  3. Esito: volontario creato solo con username univoco.

### Regole di business / vincoli
- RB1: un tipo visita è identificato da `idTipoVisita` derivato da `idLuogo + titolo`.
- RB2: overlap vietato tra due tipi visita nello stesso luogo quando programmabili nello stesso giorno e intervalli temporali si sovrappongono.
- RB3: almeno un volontario associato al tipo visita.

### Dati coinvolti
- Input: dati visita e scheduling; eventuali credenziali nuovo volontario  
- Output: conferma inserimento o richiesta correzioni  
- Entità/modelli: `Visita`, `Luogo`, `Utente`(volontario), `GestoreCalendario`

### Tracciabilità
- **Codice:** `MenuConfiguratore.creaVisitaConAssociazioneObbligatoria()`, `MenuConfiguratore.creaVisita()`, `MenuConfiguratore.menuSceltaVolontari()`, `MenuConfiguratore.inserimentoCredenzialiNuovoVolontario()`, `GestoreDati.creaVolontario()`, `GestoreDati.aggiungiVisitaALuogo()`  
- **Requisiti/slide:**  
  > Nota: non forniti.

---

## UC07 — Inserire e consultare date precluse

**Attori primari:** Configuratore  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** bloccare (precludere) specifiche date in modo che non vengano usate per la pianificazione.  
**Precondizioni:** configuratore autenticato.  
**Postcondizioni (successo):** data aggiunta alla lista e persistita; elenco consultabile.  
**Postcondizioni (fallimento):** nessuna modifica; messaggi informativi.  
**Trigger:** menu configuratore → “Attività” → “Inserimento date precluse”.

### Flusso principale (Aggiunta)
1. L’attore entra nel menu date precluse.
2. Il sistema calcola il mese di riferimento per le precluse (a partire da “meseBase+3”).
3. L’attore sceglie “aggiunta data preclusa”.
4. Il sistema chiede un giorno valido del mese e costruisce la data.
5. Il sistema verifica che la data non sia già presente.
6. Il sistema aggiunge la data e la appende al file `datePrecluse.txt`.

### Flussi alternativi / eccezioni
- **A1 — Data già preclusa**:  
  1. Condizione: data già presente nel set.  
  2. Passi: il sistema informa e non duplica.  
  3. Esito: nessuna modifica.

- **A2 — Consultazione senza dati**:  
  1. Condizione: nessuna data preclusa per quel mese.  
  2. Passi: il sistema stampa messaggio “Non ci sono date…”.  
  3. Esito: consultazione vuota.

- **A3 — Errore I/O**:  
  1. Condizione: fallisce `appendDataPreclusa()`.  
  2. Passi: stack trace.  
  3. Esito: possibile incoerenza tra memoria e file.

### Regole di business / vincoli
- RB1: le date precluse vengono escluse sia dalla selezione disponibilità volontari sia dalla produzione del piano.
- RB2: le date precluse possono essere “ripulite” automaticamente fino a un mese (vedi UC12, `dimenticaDatiFinoAlMeseIncluso`).

### Dati coinvolti
- Input: giorno del mese  
- Output: lista date precluse o conferme  
- Entità/modelli: `FileIO.appendDataPreclusa()`, `GestoreDati.getDatePrecluse()`

### Tracciabilità
- **Codice:** `MenuConfiguratore.inserimentoDatePrecluse()`, `FileIO.appendDataPreclusa()`, `GestoreDati.produciPianoVisite()`  
- **Requisiti/slide:**  
  > Nota: non forniti.

---

## UC08 — Modificare numero massimo iscrivibili da un fruitore

**Attori primari:** Configuratore  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** aggiornare il parametro che limita quante persone un fruitore può iscrivere in un’unica prenotazione.  
**Precondizioni:** configuratore autenticato; dati generali presenti o leggibili.  
**Postcondizioni (successo):** parametro aggiornato in `generale.txt`.  
**Postcondizioni (fallimento):** parametro invariato.  
**Trigger:** menu configuratore → “Menù generale” → “Modificare numero max iscrivibili da fruitore”.

### Flusso principale
1. Il sistema legge i dati generali correnti.
2. L’attore inserisce il nuovo valore massimo.
3. Il sistema salva `DatiGenerali(ambito, nuovoMax)` su file.

### Flussi alternativi / eccezioni
- **A1 — Errore I/O**:  
  1. Condizione: fallisce lettura/scrittura.  
  2. Passi: stack trace.  
  3. Esito: nessun aggiornamento garantito.

### Regole di business / vincoli
- RB1: il valore è usato per vincolare `numPersone` in UC14 (range 1..max).

### Dati coinvolti
- Input: nuovo massimo  
- Output: conferma implicita (nessun messaggio specifico)  
- Entità/modelli: `DatiGenerali`, `FileIO`

### Tracciabilità
- **Codice:** `MenuConfiguratore.modificaNumeroMaxIscrivibili()`, `FileIO.leggiDatiGenerali()`, `FileIO.salvaDatiGenerali()`  
- **Requisiti/slide:**  
  > Nota: non forniti.

---

## UC09 — Visualizzare tipi visita associati

**Attori primari:** Volontario  
**Attori secondari:** —  
**Scopo:** permettere al volontario di vedere i tipi visita in cui risulta guida abilitata.  
**Precondizioni:** volontario autenticato.  
**Postcondizioni (successo):** elenco stampato (anche vuoto).  
**Postcondizioni (fallimento):** nessuna modifica; messaggi informativi.  
**Trigger:** menu volontario → “Visualizza i tipi di visita associati”.

### Flusso principale
1. L’attore seleziona la voce di menu.
2. Il sistema cerca, nei luoghi e nelle visite associate, quelle che contengono il nickname del volontario.
3. Il sistema stampa i dettagli semplificati delle visite trovate.

### Flussi alternativi / eccezioni
- **A1 — Nessuna visita associata**:  
  1. Condizione: lista vuota.  
  2. Passi: il sistema stampa “Non risulti associato…”.  
  3. Esito: consultazione vuota.

### Regole di business / vincoli
- RB1: l’associazione è data dall’inclusione del nickname in `Visita.volontari`.

### Dati coinvolti
- Input: —  
- Output: elenco visite  
- Entità/modelli: `Luogo`, `Visita`

### Tracciabilità
- **Codice:** `MenuVolontario.visualizzaTipiVisitaAssociati()`, `MenuVolontario.getTipiVisitaAssociati()`  
- **Requisiti/slide:**  
  > Nota: non forniti.

---

## UC10 — Inserire disponibilità nel mese di raccolta

**Attori primari:** Volontario  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** inserire date di disponibilità per il mese di raccolta, nel rispetto di preclusioni e compatibilità con i tipi visita associati.  
**Precondizioni:** volontario autenticato; raccolta aperta; esiste almeno un tipo visita programmabile nel mese.  
**Postcondizioni (successo):** disponibilità aggiunta e salvata.  
**Postcondizioni (fallimento):** disponibilità non aggiunta; nessuna modifica.  
**Trigger:** menu volontario → “Esprimi disponibilità (MESE)”.

### Flusso principale
1. L’attore seleziona “Esprimi disponibilità”.
2. Il sistema carica lo stato (`StatoSistema`) e verifica `raccoltaAperta=true`.
3. Il sistema verifica che esista almeno un tipo visita associato programmabile nel mese.
4. L’attore sceglie “Aggiungi una data di disponibilità”.
5. Il sistema costruisce l’elenco di date selezionabili nel mese:
   - non precluse,
   - non già inserite dal volontario,
   - compatibili con almeno un tipo visita associato (`isProgrammabile`).
6. L’attore seleziona una data.
7. Il sistema salva la data nella mappa disponibilità e persiste su file.

### Flussi alternativi / eccezioni
- **A1 — Raccolta chiusa**:  
  1. Condizione: `stato.isRaccoltaAperta()==false`.  
  2. Passi: il sistema informa e termina l’operazione.  
  3. Esito: nessuna disponibilità aggiunta.

- **A2 — Nessun tipo programmabile nel mese**:  
  1. Condizione: nessuna visita associata risulta programmabile in alcun giorno del mese.  
  2. Passi: messaggio informativo.  
  3. Esito: nessuna disponibilità aggiunta.

- **A3 — Nessuna data selezionabile**:  
  1. Condizione: tutte le date sono precluse, già inserite, o incompatibili.  
  2. Passi: messaggio “Non ci sono date selezionabili…”.  
  3. Esito: nessuna disponibilità aggiunta.

- **A4 — Errore I/O nel salvataggio disponibilità**:  
  1. Condizione: fallisce `salvaDisponibilitaVolontari`.  
  2. Passi: stampa errore e termina.  
  3. Esito: disponibilità non garantita.

### Regole di business / vincoli
- RB1: disponibilità inseribile solo se raccolta aperta.
- RB2: una data è selezionabile solo se non preclusa e compatibile con almeno un tipo visita associato.

### Dati coinvolti
- Input: scelta data  
- Output: conferma “Disponibilità aggiunta…” / elenchi consultati  
- Entità/modelli: `StatoSistema`, `Visita.isProgrammabile()`, `FileIO.salvaDisponibilitaVolontari()`

### Tracciabilità
- **Codice:** `MenuVolontario.inserisciDisponibilita()`, `MenuVolontario.menuSceltaDisponibilitaMese()`, `MenuVolontario.buildDateSelezionabili()`, `GestoreDati.aggiungiDisponibilitaVolontario()`  
- **Requisiti/slide:**  
  > Nota: non forniti.

---

## UC11 — Chiudere raccolta disponibilità del mese

**Attori primari:** Configuratore  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** chiudere la fase di raccolta disponibilità (impedendo ulteriori inserimenti).  
**Precondizioni:** configuratore autenticato; raccolta aperta; fase temporale valida (finestra definita da `StatoSistema`).  
**Postcondizioni (successo):** `raccoltaAperta=false` salvato nello stato.  
**Postcondizioni (fallimento):** stato invariato.  
**Trigger:** menu configuratore → “Attività” → “Chiudi raccolta disponibilità”.

### Flusso principale
1. Il sistema carica `StatoSistema`.
2. Il sistema verifica che raccolta sia aperta e piano non prodotto.
3. Il sistema verifica se esiste almeno una disponibilità inserita (controllo informativo).
4. Se nessuna disponibilità, il sistema chiede conferma esplicita per chiudere comunque.
5. Il sistema chiude la raccolta e salva lo stato.

### Flussi alternativi / eccezioni
- **A1 — Raccolta già chiusa**:  
  1. Condizione: `raccoltaAperta=false`.  
  2. Passi: messaggio “Raccolta già chiusa…”.  
  3. Esito: nessuna modifica.

- **A2 — Piano già prodotto**:  
  1. Condizione: `pianoProdotto=true`.  
  2. Passi: messaggio informativo.  
  3. Esito: nessuna modifica.

- **A3 — Fuori finestra temporale**:  
  1. Condizione: oggi non è nella finestra prevista per il `meseRaccolta`.  
  2. Passi: eccezione/errore (“Fuori finestra…”).  
  3. Esito: chiusura negata.

### Regole di business / vincoli
- RB1: la chiusura è consentita solo in una finestra temporale calcolata rispetto a `meseRaccolta` (16 del mese-2 → 15 del mese-1).
- RB2: la chiusura può avvenire anche senza disponibilità, ma solo con conferma.

### Dati coinvolti
- Input: conferma (sì/no) in caso di zero disponibilità  
- Output: conferma “CHIUSA…” o messaggi informativi  
- Entità/modelli: `StatoSistema`, `FileIO.salvaStatoSistema()`, mappa disponibilità

### Tracciabilità
- **Codice:** `MenuConfiguratore.chiudiRaccoltaDisponibilita()`, `GestoreDati.chiudiRaccoltaDisponibilita()`, `StatoSistema.chiudiRaccolta()`  
- **Requisiti/slide:**  
  > Nota: non forniti.

---

## UC12 — Produrre piano visite del mese

**Attori primari:** Configuratore  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** generare le istanze di visita del mese di raccolta assegnando volontari disponibili e impostando lo stato iniziale.  
**Precondizioni:** configuratore autenticato; raccolta chiusa; piano non già prodotto; fase temporale valida.  
**Postcondizioni (successo):** piano salvato (`visiteIstanze.json`), `pianoProdotto=true`, pulizia dati vecchi (disponibilità e precluse fino al mese incluso).  
**Postcondizioni (fallimento):** piano non prodotto o parziale; stato non garantito.  
**Trigger:** menu configuratore → “Attività” → “Produci piano visite”.

### Flusso principale
1. Il sistema carica `StatoSistema`.
2. Il sistema verifica: raccolta chiusa e piano non prodotto.
3. Il sistema legge disponibilità volontari e date precluse da file.
4. Per ogni giorno del mese target:
   1. Se giorno è precluso, lo salta.
   2. Per ogni luogo e per ogni tipo visita del luogo:
      - se il tipo visita è programmabile in quel giorno,
      - sceglie il **primo volontario** (nell’ordine della lista del tipo visita) che:
        - è disponibile quel giorno,
        - non è già stato assegnato a un’altra visita nello stesso giorno.
      - crea `VisitaIstanza` con stato iniziale `proposta`.
5. Il sistema salva tutte le istanze prodotte.
6. Il sistema marca `pianoProdotto=true` e salva lo stato sistema.
7. Il sistema elimina dai file le disponibilità e le date precluse fino al mese incluso (`dimenticaDatiFinoAlMeseIncluso`).

### Flussi alternativi / eccezioni
- **A1 — Raccolta ancora aperta**:  
  1. Condizione: `raccoltaAperta=true`.  
  2. Passi: messaggio “Prima devi chiudere…”.  
  3. Esito: piano non prodotto.

- **A2 — Piano già prodotto**:  
  1. Condizione: `pianoProdotto=true`.  
  2. Passi: il sistema rilegge e restituisce il piano esistente.  
  3. Esito: nessuna rigenerazione.

- **A3 — Fuori finestra temporale**:  
  1. Condizione: oggi fuori finestra per il mese.  
  2. Passi: eccezione con intervallo consentito.  
  3. Esito: piano non prodotto.

- **A4 — Nessun volontario disponibile**:  
  1. Condizione: per un tipo visita in un giorno non si trova volontario disponibile.  
  2. Passi: quell’istanza non viene creata (silenziosamente).  
  3. Esito: piano potenzialmente “scarno”.

### Regole di business / vincoli
- RB1: una guida/volontario non può essere assegnato a più visite nello stesso giorno.
- RB2: le date precluse sono escluse.
- RB3: stato iniziale istanze create = `proposta`.

### Dati coinvolti
- Input: disponibilità volontari; date precluse; calendario tipi visita  
- Output: lista istanze create; stato sistema aggiornato  
- Entità/modelli: `VisitaIstanza`, `StatiVisita`, `StatoSistema`, `FileIO.salvaVisiteIstanze()`

### Tracciabilità
- **Codice:** `MenuConfiguratore.produciPianoVisite()`, `GestoreDati.produciPianoVisite()`, `GestoreDati.scegliVolontarioDisponibile()`, `FileIO.dimenticaDatiFinoAlMeseIncluso()`  
- **Requisiti/slide:**  
  > Nota: non forniti.

---

## UC13 — Visualizzare visite (per fruitore o globali)

**Attori primari:** Fruitore  
**Attori secondari:** —  
**Scopo:** consultare visite (proposte/confermate/cancellate) e, in modalità “mie visite”, anche complete.  
**Precondizioni:** fruitore autenticato; piano presente o consultazione possibile (anche vuota).  
**Postcondizioni (successo):** elenco stampato.  
**Postcondizioni (fallimento):** messaggio “Nessuna visita…” o errore lettura piano.  
**Trigger:** menu fruitore → “Visualizza visite…” oppure “Le mie visite…”.

### Flusso principale
1. L’attore sceglie la voce di visualizzazione desiderata.
2. Il sistema carica il piano e aggiorna automaticamente gli stati in base alla data corrente.
3. Il sistema filtra le istanze:
   - vista globale: `proposta`, `confermata`, `cancellata`;
   - “mie visite”: `proposta`, `completa`, `confermata`, `cancellata` e solo se il fruitore risulta iscritto.
4. Il sistema stampa i dettagli della visita (se tipo visita è reperibile).

### Flussi alternativi / eccezioni
- **A1 — Piano mancante o vuoto**:  
  1. Condizione: nessuna istanza.  
  2. Passi: stampa “Nessuna visita da visualizzare.”  
  3. Esito: consultazione vuota.

- **A2 — Tipo visita non trovato**:  
  1. Condizione: `trovaTipoVisita()` restituisce `null`.  
  2. Passi: l’istanza viene saltata o stampata con dettagli ridotti (dipende dal metodo).  
  3. Esito: informazioni incomplete.

### Regole di business / vincoli
- RB1: aggiornamento stati automatico: a chiusura iscrizioni (3 giorni prima) e dopo la data visita (effettuata/archiviazione o rimozione cancellate).

### Dati coinvolti
- Input: scelta menu  
- Output: elenco visite  
- Entità/modelli: `VisitaIstanza`, `GestoreDati.aggiornaStatiIscrizioni()`

### Tracciabilità
- **Codice:** `MenuFruitore.menu()` case 1 e 3, `GestoreDati.stampaVisitePerFruitore()`, `GestoreDati.aggiornaStatiIscrizioni()`  
- **Requisiti/slide:**  
  > Nota: non forniti.

---

## UC14 — Iscriversi a una visita proposta

**Attori primari:** Fruitore  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** prenotare posti su una visita proposta, ottenendo un codice prenotazione.  
**Precondizioni:** fruitore autenticato; dati generali presenti; esistono visite iscrivibili.  
**Postcondizioni (successo):** iscrizione salvata nell’istanza; restituito codice; possibile cambio stato a `completa`.  
**Postcondizioni (fallimento):** nessuna iscrizione creata; messaggio d’errore.  
**Trigger:** menu fruitore → “Iscriviti a una visita proposta”.

### Flusso principale
1. Il sistema calcola le **visite iscrivibili**:
   - solo stato `proposta`,
   - iscrizioni ancora aperte (oggi < dataVisita-3),
   - non piene rispetto a max partecipanti del tipo visita.
2. Il sistema mostra la lista e l’attore seleziona una visita.
3. Il sistema legge `DatiGenerali` e chiede `numPersone` nel range `1..maxPersoneIscrivibiliDaFruitore`.
4. Il sistema valida:
   - iscrizioni aperte (regola dei 3 giorni),
   - stato dell’istanza ancora `proposta`,
   - il fruitore non sia già iscritto alla stessa istanza,
   - capienza sufficiente.
5. Il sistema genera un codice prenotazione univoco e salva l’iscrizione.
6. Se la capienza diventa piena, il sistema imposta lo stato istanza a `completa`.
7. Il sistema salva il piano e stampa il codice.

### Flussi alternativi / eccezioni
- **A1 — Nessuna visita disponibile**:  
  1. Condizione: lista iscrivibili vuota.  
  2. Passi: messaggio informativo.  
  3. Esito: iscrizione non possibile.

- **A2 — Dati generali mancanti**:  
  1. Condizione: `leggiDatiGenerali()` restituisce `null`.  
  2. Passi: messaggio “Dati generali mancanti…”.  
  3. Esito: iscrizione negata.

- **A3 — Iscrizioni chiuse (3 giorni prima)**:  
  1. Condizione: oggi non è prima della data di chiusura (dataVisita-3).  
  2. Passi: errore con data di chiusura.  
  3. Esito: iscrizione negata.

- **A4 — Fruitore già iscritto alla stessa visita**:  
  1. Condizione: esiste già iscrizione con `usernameFruitore` sull’istanza.  
  2. Passi: errore “Sei già iscritto…”.  
  3. Esito: iscrizione negata.

- **A5 — Posti insufficienti**:  
  1. Condizione: `numPersone > postiRimasti`.  
  2. Passi: errore con posti disponibili rimasti.  
  3. Esito: iscrizione negata.

### Regole di business / vincoli
- RB1: iscrizioni chiudono **3 giorni prima** della data visita.
- RB2: iscrizione solo su stato `proposta` (non su `confermata`, `cancellata`, ecc.).
- RB3: una sola iscrizione per fruitore per istanza.
- RB4: se posti occupati raggiungono il massimo, lo stato diventa `completa`.

### Dati coinvolti
- Input: selezione visita; `numPersone`  
- Output: codice prenotazione; messaggi errore  
- Entità/modelli: `Iscrizione`, `VisitaIstanza`, `DatiGenerali`

### Tracciabilità
- **Codice:** `MenuFruitore.menuIscrizione()`, `GestoreDati.visiteIscrivibili()`, `GestoreDati.iscriviFruitoreAVisita()`, `GestoreDati.generaCodicePrenotazioneUnico()`  
- **Requisiti/slide:**  
  > Nota: non forniti.

---

## UC15 — Disdire iscrizione tramite codice

**Attori primari:** Fruitore  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** cancellare una prenotazione prima della chiusura iscrizioni, liberando posti.  
**Precondizioni:** fruitore autenticato; possiede un codice prenotazione valido e “suo”; iscrizioni ancora aperte.  
**Postcondizioni (successo):** iscrizione rimossa; se visita era `completa` e ora non più piena, torna `proposta`; piano salvato.  
**Postcondizioni (fallimento):** nulla cambia; messaggio di errore/non consentito.  
**Trigger:** menu fruitore → “Disdici iscrizione…”.

### Flusso principale
1. L’attore inserisce il codice prenotazione.
2. Il sistema normalizza il codice (trim + uppercase).
3. Il sistema scorre il piano e considera solo istanze in stato `proposta` o `completa`.
4. Per ciascuna, verifica che oggi sia prima della chiusura (dataVisita-3).
5. Se trova un’iscrizione con quel codice:
   1. verifica che lo username del fruitore coincida col proprietario del codice,
   2. rimuove l’iscrizione,
   3. se lo stato era `completa` e ora ci sono posti, imposta lo stato a `proposta`,
   4. salva il piano.
6. Il sistema conferma la disdetta.

### Flussi alternativi / eccezioni
- **A1 — Codice non trovato / non valido**:  
  1. Condizione: nessuna iscrizione col codice.  
  2. Passi: messaggio “Codice non valido, non trovato…”.  
  3. Esito: nessuna modifica.

- **A2 — Disdetta non consentita (troppo tardi)**:  
  1. Condizione: oggi non è prima di (dataVisita-3).  
  2. Passi: il sistema non consente e continua a cercare altre (di fatto esclude).  
  3. Esito: disdetta negata.

- **A3 — Codice appartenente a un altro fruitore**:  
  1. Condizione: codice trovato ma username diverso.  
  2. Passi: il sistema solleva errore “Operazione negata…”.  
  3. Esito: nessuna modifica.

### Regole di business / vincoli
- RB1: disdetta possibile solo prima della chiusura iscrizioni (3 giorni prima).
- RB2: disdetta possibile solo se l’istanza è `proposta` o `completa`.
- RB3: il codice è “segreto” e vincolato al fruitore proprietario.

### Dati coinvolti
- Input: codice prenotazione  
- Output: conferma o errore  
- Entità/modelli: `Iscrizione`, `VisitaIstanza`

### Tracciabilità
- **Codice:** `MenuFruitore.menuDisdetta()`, `GestoreDati.disdiciIscrizioneByCodice()`  
- **Requisiti/slide:**  
  > Nota: non forniti.

---

## UC16 — Consultare stato visite e piano prodotto

**Attori primari:** Configuratore  
**Attori secondari:** —  
**Scopo:** avere una vista sintetica degli stati delle istanze e/o stampare il piano.  
**Precondizioni:** configuratore autenticato; piano leggibile (anche vuoto).  
**Postcondizioni (successo):** stato/piano stampati.  
**Postcondizioni (fallimento):** messaggi informativi o errore lettura.  
**Trigger:** menu configuratore → “Menù generale” → “Visualizzare stato delle visite” / “Visualizzare piano visite prodotto”.

### Flusso principale (stato)
1. L’attore seleziona “Visualizzare stato delle visite”.
2. Il sistema legge le istanze del piano.
3. Il sistema conta quante istanze ci sono per ciascuno stato (`proposta`, `completa`, `confermata`, `cancellata`, `effettuata`).
4. Il sistema stampa i conteggi.

### Flusso principale (piano)
1. L’attore seleziona “Visualizzare piano visite prodotto”.
2. Il sistema legge e ordina le istanze per data e id tipo visita.
3. Il sistema stampa per ogni istanza: data, idTipoVisita, volontario, stato.

### Flussi alternativi / eccezioni
- **A1 — Piano non prodotto / vuoto**:  
  1. Condizione: lista istanze vuota.  
  2. Passi: messaggio “piano vuoto” o “stato non disponibile…”.  
  3. Esito: consultazione vuota.

- **A2 — Errore I/O**:  
  1. Condizione: fallisce `leggiPianoVisite()`.  
  2. Passi: errore.  
  3. Esito: consultazione fallita.

### Regole di business / vincoli
- RB1: gli stati sono quelli definiti da `StatiVisita`.

### Dati coinvolti
- Input: —  
- Output: report su console  
- Entità/modelli: `VisitaIstanza`, `StatiVisita`

### Tracciabilità
- **Codice:** `MenuConfiguratore.mostraStatoVisite()`, `MenuConfiguratore.visualizzaPianoProdotto()`, `GestoreDati.leggiPianoVisite()`  
- **Requisiti/slide:**  
  > Nota: non forniti.

---

## UC17 — Rimuovere luogo/tipo visita/volontario con cascata

**Attori primari:** Configuratore  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** gestire richieste di rimozione che possono eliminare entità orfane (tipi visita senza volontari, luoghi senza visite, volontari senza associazioni).  
**Precondizioni:** configuratore autenticato; **piano già prodotto** (vincolo imposto dal menu modifiche).  
**Postcondizioni (successo):** entità rimosse; persistenza aggiornata; stati/attività sincronizzati.  
**Postcondizioni (fallimento):** nessuna modifica oppure rimozione parziale se errore I/O.  
**Trigger:** menu configuratore → “Attività” → “Gestisci richieste di aggiunta/rimozione dati”.

### Flusso principale (rimozione luogo)
1. Il sistema verifica che `pianoProdotto=true`.
2. L’attore seleziona “Rimuovi luogo (con cascata)”.
3. Il sistema mostra luoghi attivi e l’attore ne seleziona uno.
4. Il sistema richiede conferma esplicita (warning su cascata).
5. Il sistema rimuove il luogo e tutte le visite associate, poi applica la chiusura transitiva:
   - rimuove tipi visita senza volontari,
   - rimuove luoghi senza visite,
   - rimuove volontari non più associati a nessun tipo visita.
6. Il sistema salva utenti/luoghi/visite e sincronizza attività/archivio.

### Flussi alternativi / eccezioni
- **A1 — Piano non prodotto**:  
  1. Condizione: `pianoProdotto=false`.  
  2. Passi: messaggio “Prima devi produrre il piano…”.  
  3. Esito: rimozione negata.

- **A2 — Nessun elemento disponibile** (es. nessun luogo attivo / nessun tipo visita attivo / nessun volontario):  
  1. Condizione: liste vuote.  
  2. Passi: messaggio informativo.  
  3. Esito: nessuna operazione.

- **A3 — Conferma negata**:  
  1. Condizione: l’attore risponde “no”.  
  2. Passi: operazione annullata.  
  3. Esito: nessuna modifica.

- **A4 — Errore I/O**:  
  1. Condizione: falliscono salvataggi.  
  2. Passi: errore.  
  3. Esito: rischio incoerenza; non garantito.

### Regole di business / vincoli
- RB1: rimozioni con cascata possono eliminare ulteriori entità rese “orfane”.
- RB2: dopo rimozioni si ricalcolano `attivo` su luoghi e volontari in base alle visite attive.

### Dati coinvolti
- Input: selezione entità; conferma  
- Output: messaggi di completamento/errore  
- Entità/modelli: `Luogo`, `Visita`, `Utente`, `GestoreDati.rimuovi*ConCascata()`

### Tracciabilità
- **Codice:** `MenuConfiguratore.menuModifiche()`, `MenuConfiguratore.richiestaRimozioneLuogo()`, `MenuConfiguratore.richiestaRimozioneTipoVisita()`, `MenuConfiguratore.richiestaRimozioneVolontario()`, `GestoreDati.rimuoviLuogoConCascata()`, `GestoreDati.chiusuraTransitiva()`  
- **Requisiti/slide:**  
  > Nota: non forniti.

---

## UC18 — Visualizzare piano visite associate al volontario

**Attori primari:** Volontario  
**Attori secondari:** —  
**Scopo:** mostrare le istanze del piano assegnate al volontario (anche proposte/complete/confermate).  
**Precondizioni:** volontario autenticato; piano leggibile.  
**Postcondizioni (successo):** elenco stampato con dettagli e iscritti (codice → persone).  
**Postcondizioni (fallimento):** messaggi informativi (piano non prodotto o nessuna visita).  
**Trigger:** menu volontario → “Visualizza il piano delle visite associate”.

### Flusso principale
1. L’attore seleziona la voce.
2. Il sistema carica il piano e aggiorna automaticamente gli stati.
3. Il sistema filtra le istanze con `nomeVolontario == nicknameVolontario` e stato in `{confermata, proposta, completa}`.
4. Il sistema stampa dettagli per ciascuna istanza, includendo elenco iscritti (codice → numero persone).

### Flussi alternativi / eccezioni
- **A1 — Piano non prodotto**:  
  1. Condizione: `StatoSistema.pianoProdotto=false` (controllato con logica a messaggi).  
  2. Passi: stampa “Piano non prodotto.”  
  3. Esito: consultazione non disponibile.

- **A2 — Nessuna visita assegnata**:  
  1. Condizione: piano prodotto ma nessuna istanza per il volontario.  
  2. Passi: stampa “Non hai visite come guida…”.  
  3. Esito: consultazione vuota.

### Regole di business / vincoli
- RB1: la stampa mostra anche iscrizioni per trasparenza operativa.

### Dati coinvolti
- Input: —  
- Output: elenco visite del volontario  
- Entità/modelli: `VisitaIstanza.toStringPerVolontario()`

### Tracciabilità
- **Codice:** `MenuVolontario.visualizzaPianoComeVolontario()`, `GestoreDati.stampaVisiteConfermatePerVolontario()`  
- **Requisiti/slide:**  
  > Nota: non forniti.

---

## UC19 — Visualizzare archivio storico

**Attori primari:** Configuratore  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** consultare i dati storici persistiti.  
**Precondizioni:** configuratore autenticato; archivio leggibile.  
**Postcondizioni (successo):** archivio stampato su console.  
**Postcondizioni (fallimento):** errore I/O.  
**Trigger:** menu configuratore → “Menù generale” → “visualizza archivio storico”.

### Flusso principale
1. L’attore seleziona la voce.
2. Il sistema legge `archivioStorico.json`.
3. Il sistema stampa l’oggetto archivio (tramite `toString()`).

### Flussi alternativi / eccezioni
- **A1 — Errore I/O**:  
  1. Condizione: fallisce la lettura del file.  
  2. Passi: stack trace.  
  3. Esito: consultazione fallita.

### Regole di business / vincoli
- RB1: l’archivio contiene almeno “tipi visita scaduti” e “visite effettuate” (persistite).

### Dati coinvolti
- Input: —  
- Output: testo archivio  
- Entità/modelli: `ArchivioStorico`

### Tracciabilità
- **Codice:** `MenuConfiguratore.stampaArchivio()`, `FileIO.leggiArchivioStorico()`, `GestoreDati.sincronizzaStatiAttiviEArchivio()`, `GestoreDati.archiviaVisitaEffettuata()`  
- **Requisiti/slide:**  
  > Nota: non forniti.

---

# Scostamenti e ambiguità

## 1) “Riapri raccolta disponibilità (mese successivo)” sembra non funzionare come etichetta promette
- Nel menu configuratore attività compare la voce: **“Riapri raccolta disponibilità (MESE SUCCESSIVO)”** (`MenuConfiguratore.menuAttivita()`).
- Ma `MenuConfiguratore.riapriRaccoltaMeseSuccessivo()`:
  - richiede **prima** che `stato.isPianoProdotto()` sia `true`,
  - poi chiama `GestoreDati.riapriRaccoltaMeseSuccessivo()`.
- Tuttavia `GestoreDati.riapriRaccoltaMeseSuccessivo()` **rifiuta** se `stato.isPianoProdotto()` è `true` (“Piano già prodotto… la raccolta per questo mese non si riapre.”) e **non sposta** `meseRaccolta` al mese successivo.
- Effetto pratico: con i controlli attuali, la voce rischia di essere un “vicolo cieco” (non apre nulla).

## 2) Archivio storico stampato incompleto rispetto ai dati memorizzati
- `ArchivioStorico` include sia `tipiVisitaScaduti` sia `istanzeEffettuate`.
- Però `ArchivioStorico.toString()` stampa solo i **tipi visita scaduti**, e `MenuConfiguratore.stampaArchivio()` usa `toString()`.
- Effetto: l’utente non vede le **visite effettuate** anche se vengono archiviate tramite `GestoreDati.archiviaVisitaEffettuata()`.

## 3) Ramo “visita == null” nella creazione luogo+visita non attivabile dai flussi visibili
- In `MenuConfiguratore.creaLuogoConVisitaObbligatoria()` esiste gestione di `visita == null` che porta a rimuovere il luogo.
- Ma `creaVisita()` nel codice mostrato **non ritorna mai `null`**.
- Interpretazione: codice difensivo o residuo di versioni precedenti.