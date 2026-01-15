# Attori e confini del sistema

## Attori (osservabili nel codice)
- **Configuratore** (attore umano): effettua login e usa i menu da console.
- **File System / Persistenza su file** (attore secondario): lettura/scrittura su directory dati (es. `./DATA`).
- **Orologio di sistema** (attore secondario): data “oggi” per calcoli su scadenze e per la scelta del mese delle date precluse.

## Confini del sistema
- **Dentro**: applicazione Java stand-alone da console; gestione luoghi; gestione tipi visita (creazione + vincolo di sovrapposizione); gestione date precluse; gestione dato generale “max iscrivibili da fruitore”; consultazioni; archiviazione dei **tipi visita scaduti**; persistenza su file.
- **Fuori**: console (I/O utente), file system, clock di sistema.

---

# Elenco casi d’uso (tabella)

| ID   | Titolo | Attore primario | Priorità | Stato |
|------|--------|------------------|----------|-------|
| UC01 | Avviare applicazione e caricare dati persistenti | Sistema 
| UC02 | Autenticarsi come configuratore | Configuratore 
| UC03 | Cambiare credenziali al primo accesso | Configuratore 
| UC04 | Inserire o visualizzare dati generali | Configuratore 
| UC05 | Navigare il menù del configuratore | Configuratore 
| UC06 | Creare luogo con prima visita obbligatoria | Configuratore 
| UC07 | Creare visita e associarla obbligatoriamente a un luogo | Configuratore 
| UC08 | Gestire sovrapposizione (overlap) in creazione visita | Configuratore 
| UC09 | Aggiungere una data preclusa (mese target) | Configuratore 
| UC10 | Visualizzare date precluse (mese target) | Configuratore 
| UC11 | Modificare “max iscrivibili da fruitore” | Configuratore 
| UC12 | Visualizzare volontari attivi e tipi visita associati | Configuratore 
| UC13 | Visualizzare luoghi attivi (visitabili) | Configuratore 
| UC14 | Visualizzare tipi visita attivi (per luogo) e consultare archivio storico | Configuratore 

---

## UC01 — Avviare applicazione e caricare dati persistenti
**Attori primari:** Sistema  
**Attori secondari:** File System, Orologio di sistema  
**Scopo:** Caricare i dati da file, sincronizzare i tipi visita scaduti e poi avviare login e menu.  
**Precondizioni:** Applicazione avviata; directory dati accessibile.  
**Postcondizioni (successo):** Dati in memoria; eventuali tipi visita scaduti rimossi dagli attivi e salvati in archivio; si passa al login.  
**Postcondizioni (fallimento):** Errore di I/O non gestito a livello utente (stack trace); comportamento successivo non garantito.  
**Trigger:** Esecuzione di `Main.main(...)`.

### Flusso principale
1. Il sistema avvia `Main.main(...)`.
2. Il sistema invoca `GestoreDati.caricaDaDirectory("./DATA")`.
3. Il sistema legge da file: utenti, luoghi, visite, date precluse, dati generali/archivio (se presenti).
4. Il sistema ricostruisce le visite dentro i luoghi (in base a `idLuogo`).
5. Il sistema esegue una sincronizzazione rispetto alla data odierna:
   1. Per ogni tipo visita, aggiorna `attivo` in base alla scadenza.
   2. Se un tipo visita risulta non attivo, lo sposta nell’archivio storico e lo rimuove dalla lista delle visite attive.
6. Il sistema prosegue con UC02 (login).

### Regole di business / vincoli
- RB1: Un tipo visita è “scaduto” se la sua `dataFine` è **prima** di “oggi” (quindi non è più attivo).

### Dati coinvolti
- Input: file su `./DATA` (utenti, luoghi, visite, date precluse, archivio).  
- Output: strutture dati in memoria; eventuali file aggiornati (visite e archivio).  
- Entità/modelli: `GestoreDati`, `FileIO`, `Luogo`, `Visita`, `ArchivioStorico`.

---

## UC02 — Autenticarsi come configuratore
**Attori primari:** Configuratore  
**Attori secondari:** —  
**Scopo:** Consentire l’accesso ai menu solo a un utente con ruolo `"configuratore"`.  
**Precondizioni:** UC01 completato; lista utenti caricata.  
**Postcondizioni (successo):** Sessione attiva con `utenteCorrente` valorizzato.  
**Postcondizioni (fallimento):** Accesso negato, ripetizione richiesta credenziali.  
**Trigger:** Termine di UC01.

### Flusso principale
1. Il configuratore inserisce username e password via console.
2. Il sistema valida le credenziali confrontandole con gli utenti caricati.
3. Il sistema accetta l’accesso solo se l’utente ha ruolo `"configuratore"`.
4. Se l’utente è al primo accesso, il sistema avvia UC03, altrimenti passa a UC04.


### Regole di business / vincoli
- RB1: Solo ruolo `"configuratore"` può autenticarsi in questa versione.

### Dati coinvolti
- Input: username, password.  
- Output: utente autenticato in sessione.  
- Entità/modelli: `Utente`.

### Tracciabilità
- **Codice:** `Main.login`, `Main.verificaCredenziali`, `GestoreDati.autentica`  
- **Requisiti/slide:** non forniti

---

## UC03 — Cambiare credenziali al primo accesso
**Attori primari:** Configuratore  
**Attori secondari:** File System  
**Scopo:** Aggiornare username/password se l’utente ha `primoAccesso=true`.  
**Precondizioni:** UC02 completato; `utenteCorrente.isPrimoAccesso()==true`.  
**Postcondizioni (successo):** Credenziali aggiornate e salvate; `primoAccesso` impostato a `false`.  
**Postcondizioni (fallimento):** Credenziali non cambiate; richiesta ripetuta.  
**Trigger:** Rilevazione “primo accesso” dopo UC02.

### Flusso principale
1. Il sistema chiede nuovo username e nuova password.
2. Il sistema valida:
   1. username non vuoto;
   2. password non vuota;
   3. password diversa da quella precedente;
   4. se lo username cambia, verifica che non esista già.
3. Il sistema aggiorna l’utente corrente (`setNomeUtente`, `setPassword`, `setPrimoAccesso(false)`).
4. Il sistema salva su file l’elenco utenti.
5. Il sistema stampa conferma di avvenuto aggiornamento.

### Regole di business / vincoli
- RB1: Username univoco nel sistema.  
- RB2: Password nuova diversa dalla precedente.

### Dati coinvolti
- Input: nuovo username, nuova password.  
- Output: file utenti aggiornato.  
- Entità/modelli: `Utente`, `FileIO`.

### Tracciabilità
- **Codice:** `Main.cambioCredenziali`, `GestoreDati.aggiornaCredenziali`, `FileIO.salvaUtenti`  
- **Requisiti/slide:** non forniti

---

## UC04 — Inserire o visualizzare dati generali
**Attori primari:** Configuratore  
**Attori secondari:** File System  
**Scopo:** Gestire i dati generali (ambito territoriale e max iscrivibili da fruitore).  
**Precondizioni:** UC02 completato.  
**Postcondizioni (successo):** Se mancanti, dati creati e salvati; se presenti, dati mostrati.  
**Postcondizioni (fallimento):** Nessuna modifica garantita; stack trace su errore I/O.  
**Trigger:** Dopo UC02/UC03.

### Flusso principale
1. Il sistema prova a leggere i dati generali da file.
2. Se presenti:
   1. il sistema stampa ambito territoriale;
   2. il sistema stampa “max persone iscrivibili da un fruitore”.
3. Se assenti:
   1. il configuratore inserisce ambito territoriale;
   2. il configuratore inserisce il valore max;
   3. il sistema salva i dati generali.


### Regole di business / vincoli
- RB1: Non risultano vincoli applicati sul valore max (es. > 0): viene accettato qualunque intero.

### Dati coinvolti
- Input: ambito territoriale, max.  
- Output: dati generali su file / stampa console.  
- Entità/modelli: `DatiGenerali`, `FileIO`.

### Tracciabilità
- **Codice:** `Main.chiediDatiGenerali`, `FileIO.leggiDatiGenerali`, `FileIO.salvaDatiGenerali`  
- **Requisiti/slide:** non forniti

---

## UC05 — Navigare il menù del configuratore
**Attori primari:** Configuratore  
**Attori secondari:** —  
**Scopo:** Accedere alle funzioni di setup iniziale e al menu generale.  
**Precondizioni:** UC02 completato; `GestoreDati` inizializzato; menu creato.  
**Postcondizioni (successo):** Funzione selezionata eseguita oppure uscita dal menu.  
**Postcondizioni (fallimento):** Messaggio “Scelta non disponibile” e ritorno al menu.  
**Trigger:** Dopo UC04.

### Flusso principale
1. Il sistema calcola se il sistema è “inizializzato”:
   - vero se esiste almeno un luogo con almeno una visita associata.
2. Il sistema mostra il “Menù configuratore” con:
   - “Menu generale” (sempre),
   - “Setup iniziale” (solo se non inizializzato).
3. Il configuratore seleziona una voce:
   - se “Menu generale” → usa UC09–UC14 (in base alla singola scelta),
   - se “Setup iniziale” → usa UC06 o UC07 (in base alla scelta successiva).
4. Il configuratore può uscire con scelta `0`.


### Regole di business / vincoli
- RB1: “Setup iniziale” appare solo se non ci sono luoghi con visite associate.

### Dati coinvolti
- Input: scelta numerica del menu.  
- Output: navigazione e invocazione funzioni.  
- Entità/modelli: `MenuConfiguratore`.

### Tracciabilità
- **Codice:** `MenuConfiguratore.menu`, `MenuConfiguratore.sistemaInizializzato`, `MenuConfiguratore.menuGenerale`, `MenuConfiguratore.menuSetupIniziale`  
- **Requisiti/slide:** non forniti

---

## UC06 — Creare luogo con prima visita obbligatoria
**Attori primari:** Configuratore  
**Attori secondari:** File System  
**Scopo:** Creare un luogo e garantire che abbia almeno un tipo visita associato.  
**Precondizioni:** UC05; “Setup iniziale” disponibile oppure selezionato.  
**Postcondizioni (successo):** Luogo salvato; visita salvata e associata; vincolo overlap rispettato.  
**Postcondizioni (fallimento):** Se l’operazione viene annullata dopo la creazione del luogo, il luogo viene rimosso e risalvato l’elenco luoghi.  
**Trigger:** Scelta “Crea luogo + prima visita (obbligatoria)”.

### Flusso principale
1. Il configuratore inserisce i dati del luogo (nome, descrizione, latitudine, longitudine).
2. Il sistema genera l’ID del luogo e verifica che non esista già un luogo con lo stesso ID.
3. Il sistema aggiunge il luogo e salva la lista luoghi su file.
4. Il sistema avvia la creazione della visita associata (dati visita + volontari + scheduling).
5. Il sistema tenta di aggiungere la visita al luogo verificando il vincolo di non sovrapposizione.
6. Se la visita viene aggiunta correttamente, il sistema salva e conferma l’inserimento.

### Regole di business / vincoli
- RB1: Un luogo è duplicato se ha lo stesso ID (ID derivato da nome e coordinate).  
- RB2: Per uno stesso luogo non devono esistere visite che si sovrappongono temporalmente in un giorno comune del periodo (vincolo overlap).

### Dati coinvolti
- Input: dati luogo; dati visita; selezione/creazione volontari; scheduling; min/max partecipanti; biglietto.  
- Output: file luoghi e file visite aggiornati (e utenti se vengono creati/riattivati volontari).  
- Entità/modelli: `Luogo`, `Visita`, `PuntoIncontro`, `SchedulingVisita`, `Utente`, `GestoreCalendario`.

### Tracciabilità
- **Codice:** `MenuConfiguratore.creaLuogoConVisitaObbligatoria`, `MenuConfiguratore.creaLuogo`, `MenuConfiguratore.creaVisita`, `GestoreDati.aggiungiLuogo`, `GestoreDati.aggiungiVisitaALuogo`, `GestoreCalendario.aggiungiVisita`  
- **Requisiti/slide:** non forniti

---

## UC07 — Creare visita e associarla obbligatoriamente a un luogo
**Attori primari:** Configuratore  
**Attori secondari:** File System  
**Scopo:** Creare un tipo visita e associarlo subito a un luogo esistente o a un nuovo luogo.  
**Precondizioni:** UC05; “Setup iniziale” selezionato.  
**Postcondizioni (successo):** Visita creata e associata; dati salvati.  
**Postcondizioni (fallimento):** Nessuna visita aggiunta; eventuali rollback se si crea un luogo nuovo e poi si annulla tramite UC06/UC08.  
**Trigger:** Scelta “Crea visita (associa subito a luogo esistente o nuovo)”.

### Flusso principale
1. Il sistema mostra le opzioni:
   1. associare a luogo esistente;
   2. creare nuovo luogo e associare la visita.
2. Se il configuratore sceglie “luogo esistente”:
   1. il sistema mostra l’elenco luoghi;
   2. il configuratore seleziona un luogo (o torna indietro con `0`);
   3. se il luogo selezionato è inattivo, il sistema lo riattiva e salva i luoghi.
3. Il configuratore inserisce i dati della visita:
   - titolo (con verifica unicità nel luogo),
   - descrizione,
   - punto di incontro,
   - min/max partecipanti,
   - flag biglietto,
   - selezione volontari (almeno uno; con possibilità di creare un nuovo volontario),
   - scheduling (date/ora/durata/giorni).
4. Il sistema tenta di aggiungere la visita al luogo verificando il vincolo overlap.
5. Se l’aggiunta è valida, il sistema salva e conferma.

### Regole di business / vincoli
- RB1: Titolo visita univoco all’interno dello stesso luogo.  
- RB2: In fase di selezione volontari è obbligatorio selezionare almeno un volontario.  
- RB3: Se un volontario selezionato è inattivo, viene riattivato e salvato.  
- RB4: Vincolo overlap come in UC06.

### Dati coinvolti
- Input: scelta luogo; dati visita; scelta/creazione volontari; scheduling.  
- Output: file visite e luoghi aggiornati; file utenti aggiornato se si crea/riattiva volontario.  
- Entità/modelli: `Luogo`, `Visita`, `Utente`, `SchedulingVisita`, `PuntoIncontro`.

### Tracciabilità
- **Codice:** `MenuConfiguratore.creaVisitaConAssociazioneObbligatoria`, `MenuConfiguratore.scegliLuogoEsistente`, `MenuConfiguratore.creaVisita`, `MenuConfiguratore.menuSceltaVolontari`, `MenuConfiguratore.inserimentoCredenzialiNuovoVolontario`, `GestoreDati.creaVolontario`, `GestoreDati.aggiungiVisitaALuogo`, `GestoreCalendario.aggiungiVisita`  
- **Requisiti/slide:** non forniti

---

## UC08 — Gestire sovrapposizione (overlap) in creazione visita
**Attori primari:** Configuratore  
**Attori secondari:** —  
**Scopo:** Decidere come procedere quando una visita si sovrappone a un’altra nello stesso luogo.  
**Precondizioni:** UC06 o UC07 in corso; overlap rilevato dal controllo di calendario.  
**Postcondizioni (successo):** Scheduling modificato e aggiunta visita riprovata.  
**Postcondizioni (fallimento):** Creazione visita annullata (e, in UC06, può avvenire rollback del luogo).  
**Trigger:** `aggiungiVisitaALuogo(...)` fallisce perché `GestoreCalendario.aggiungiVisita(...)` restituisce `false`.

### Flusso principale
1. Il sistema mostra un menu: “OVERLAP rilevato… (0 = annulla creazione visita)” con l’opzione di modificare parametri.
2. Il configuratore sceglie di modificare lo scheduling.
3. Il sistema richiede di reinserire solo: date/ora/durata/giorni.
4. Il sistema riprova l’aggiunta della visita al luogo.

### Regole di business / vincoli
- RB1: Il sistema non consente overlap tra visite dello stesso luogo nei giorni in comune del periodo.

### Dati coinvolti
- Input: scelta annulla/modifica; nuovi parametri di scheduling.  
- Output: eventuale nuova visita salvata, o annullamento.  
- Entità/modelli: `GestoreCalendario`, `SchedulingVisita`, `Visita`.

### Tracciabilità
- **Codice:** `MenuConfiguratore.menuOverlap`, `MenuConfiguratore.aggiornaScheduling`, `MenuConfiguratore.chiediScheduling`, `GestoreCalendario.aggiungiVisita`  
- **Requisiti/slide:** non forniti

---

## UC09 — Aggiungere una data preclusa (mese target)
**Attori primari:** Configuratore  
**Attori secondari:** File System, Orologio di sistema  
**Scopo:** Inserire una data preclusa all’interno del mese target calcolato dal sistema e salvarla su file.  
**Precondizioni:** UC05; accesso a “Menù generale”; insieme date precluse caricato.  
**Postcondizioni (successo):** Data aggiunta in memoria e append su file.  
**Postcondizioni (fallimento):** Data non salvata su file se errore I/O (ma potrebbe essere stata aggiunta in memoria).  
**Trigger:** Menù generale → “Inserire date precluse” → “aggiunta data preclusa”.

### Flusso principale
1. Il sistema calcola il mese target:
   1. se oggi è dal giorno 16 in poi: base = mese corrente, altrimenti base = mese precedente;
   2. mese target = base + 3 mesi.
2. Il sistema mostra il menu delle date precluse per il mese target.
3. Il configuratore inserisce un giorno (1..ultimo giorno del mese target).
4. Il sistema costruisce la data completa (anno-mese del target + giorno).
5. Il sistema valida che la data non sia già presente tra le date precluse.
6. Il sistema aggiunge la data all’insieme in memoria.
7. Il sistema salva la data su file in append.

### Regole di business / vincoli
- RB1: Le date precluse gestite da questo menu sono limitate al **mese target** calcolato (non è un inserimento “libero” su qualunque data).  
- RB2: Non sono ammessi duplicati nello stesso insieme.

### Dati coinvolti
- Input: giorno del mese.  
- Output: nuova riga/record in file date precluse; aggiornamento in memoria.  
- Entità/modelli: `Set<LocalDate>`.

### Tracciabilità
- **Codice:** `MenuConfiguratore.inserimentoDatePrecluse`, `FileIO.appendDataPreclusa`  
- **Requisiti/slide:** non forniti

---

## UC10 — Visualizzare date precluse (mese target)
**Attori primari:** Configuratore  
**Attori secondari:** —  
**Scopo:** Visualizzare le date precluse del mese target, ordinate.  
**Precondizioni:** UC05; accesso a “Menù generale”; insieme date precluse disponibile in memoria.  
**Postcondizioni (successo):** Elenco stampato oppure messaggio “nessuna data”.  
**Postcondizioni (fallimento):** —  
**Trigger:** Menù generale → “Inserire date precluse” → “visualizza date precluse”.

### Flusso principale
1. Il sistema calcola lo stesso mese target di UC09.
2. Il sistema filtra le date precluse mantenendo solo quelle del mese target.
3. Il sistema ordina le date filtrate.
4. Il sistema stampa:
   - l’elenco delle date, oppure
   - un messaggio che indica che non ci sono date precluse per quel mese.

### Regole di business / vincoli
- RB1: La visualizzazione è limitata al mese target calcolato dal sistema.

### Dati coinvolti
- Input: —  
- Output: stampa console delle date.  
- Entità/modelli: `Set<LocalDate>`.

### Tracciabilità
- **Codice:** `MenuConfiguratore.inserimentoDatePrecluse` (ramo “visualizza”)  
- **Requisiti/slide:** non forniti

---

## UC11 — Modificare “max iscrivibili da fruitore”
**Attori primari:** Configuratore  
**Attori secondari:** File System  
**Scopo:** Aggiornare il valore massimo di persone iscrivibili “da un fruitore” nei dati generali.  
**Precondizioni:** UC05; accesso a “Menù generale”; dati generali leggibili.  
**Postcondizioni (successo):** File dati generali aggiornato.  
**Postcondizioni (fallimento):** Nessun aggiornamento garantito; stack trace su errore I/O.  
**Trigger:** Menù generale → “Modificare numero max iscrivibili da fruitore”.

### Flusso principale
1. Il sistema legge i dati generali correnti.
2. Il configuratore inserisce un nuovo valore intero per il max.
3. Il sistema salva i dati generali con lo stesso ambito territoriale e il nuovo max.

### Regole di business / vincoli
- RB1: Non risulta una validazione sul nuovo max (può essere anche non positivo).

### Dati coinvolti
- Input: nuovo max (intero).  
- Output: `DatiGenerali` salvato.  
- Entità/modelli: `DatiGenerali`, `FileIO`.

### Tracciabilità
- **Codice:** `MenuConfiguratore.modificaNumeroMaxIscrivibili`, `FileIO.leggiDatiGenerali`, `FileIO.salvaDatiGenerali`  
- **Requisiti/slide:** non forniti

---

## UC12 — Visualizzare volontari attivi e tipi visita associati
**Attori primari:** Configuratore  
**Attori secondari:** —  
**Scopo:** Mostrare per ogni volontario attivo l’elenco dei tipi visita attivi a cui è associato (oppure “nessuno”).  
**Precondizioni:** UC05; accesso a “Menù generale”.  
**Postcondizioni (successo):** Stampa mappa volontario → visite attive.  
**Postcondizioni (fallimento):** —  
**Trigger:** Menù generale → “Visualizzare elenco volontari con visite associate”.

### Flusso principale
1. Il sistema crea una mappa con tutti gli utenti ruolo `"volontario"` e `attivo=true`.
2. Se non esistono tipi visita (lista visite `null`), il sistema stampa “Non ci sono tipi visita.” e termina.
3. Il sistema scorre tutti i tipi visita nel calendario:
   1. ignora visite `null` o non attive;
   2. per ogni volontario associato alla visita, aggiunge il titolo visita alla lista nella mappa.
4. Se non esistono volontari attivi, il sistema stampa “Non ci sono volontari attivi.” e termina.
5. Il sistema stampa, per ogni volontario:
   - “(nessun tipo visita attivo)” se lista vuota, altrimenti la lista titoli.

### Regole di business / vincoli
- RB1: Vengono considerati solo volontari `attivo=true`.  
- RB2: Vengono considerati solo tipi visita `attivo=true`.

### Dati coinvolti
- Input: —  
- Output: stampa console.  
- Entità/modelli: `Utente`, `Visita`, `GestoreCalendario`.

### Tracciabilità
- **Codice:** `MenuConfiguratore.mostraElencoVolontariAssociatiVisita`  
- **Requisiti/slide:** non forniti

---

## UC13 — Visualizzare luoghi attivi (visitabili)
**Attori primari:** Configuratore  
**Attori secondari:** —  
**Scopo:** Stampare l’elenco dei luoghi con `attivo=true`.  
**Precondizioni:** UC05; accesso a “Menù generale”.  
**Postcondizioni (successo):** Elenco luoghi stampato o messaggio “nessuno”.  
**Postcondizioni (fallimento):** —  
**Trigger:** Menù generale → “Visualizzare elenco luoghi visitabili”.

### Flusso principale
1. Il sistema filtra i luoghi mantenendo solo quelli attivi.
2. Se la lista è vuota, il sistema stampa “Non esistono luoghi attivi.”.
3. Altrimenti il sistema stampa il nome di ciascun luogo attivo.


### Regole di business / vincoli
- RB1: Solo luoghi `attivo=true` vengono mostrati.

### Dati coinvolti
- Input: —  
- Output: stampa console.  
- Entità/modelli: `Luogo`.

### Tracciabilità
- **Codice:** `MenuConfiguratore.mostraElencoLuoghi`  
- **Requisiti/slide:** non forniti

---

## UC14 — Visualizzare tipi visita attivi (per luogo) e consultare archivio storico
**Attori primari:** Configuratore  
**Attori secondari:** File System (solo per archivio storico)  
**Scopo:** Consultare l’elenco dei tipi visita attivi raggruppati per luogo e consultare l’archivio storico dei tipi visita scaduti.  
**Precondizioni:** UC05; accesso a “Menù generale”.  
**Postcondizioni (successo):** Stampa elenco tipi visita attivi oppure archivio storico.  
**Postcondizioni (fallimento):** Stack trace su errore I/O (archivio).  
**Trigger:**  
- Menù generale → “Visualizzare l’elenco dei tipi di visita” **oppure**  
- Menù generale → “visualizza archivio storico”.

### Flusso principale (visualizzare tipi visita attivi)
1. Il sistema scorre i luoghi attivi.
2. Per ogni luogo, filtra le visite del luogo mantenendo solo quelle attive.
3. Se per un luogo esistono visite attive, stampa:
   1. nome del luogo,
   2. per ogni visita attiva, una riga con versione “semplificata” dei dati (`toStringSemplificato`).
4. Se non è stata trovata alcuna visita attiva in alcun luogo, stampa “Non esistono tipi di visita attivi.”.


### Regole di business / vincoli
- RB1: In elenco tipi visita vengono mostrate solo visite `attivo=true` e luoghi `attivo=true`.  
- RB2: L’archivio storico contiene tipi visita scaduti rimossi dagli attivi durante la sincronizzazione (UC01).

### Dati coinvolti
- Input: —  
- Output: stampa console (elenchi) o stampa `ArchivioStorico.toString()`.  
- Entità/modelli: `Luogo`, `Visita`, `ArchivioStorico`, `FileIO`.

### Tracciabilità
- **Codice:**  
  - Elenco tipi visita: `MenuConfiguratore.mostraElencoTipiVisita`, `Visita.toStringSemplificato`  
  - Archivio storico: `MenuConfiguratore.stampaArchivio`, `FileIO.leggiArchivioStorico`  
- **Requisiti/slide:** non forniti

---

# Scostamenti e ambiguità

1. **Messaggio “Utente non attivo” nel login, ma nessun controllo `attivo` sul configuratore**
   - In login si stampa “Utente non attivo o credenziali errate.”, però `GestoreDati.autentica(...)` verifica solo username/password e ruolo `"configuratore"`, senza usare un flag “attivo”.

2. **Voce “Visualizzare stato delle visite” non implementata**
   - Il menu generale contiene “Visualizzare stato delle visite”, ma l’azione stampa solo “(Versione 1: stati non ancora gestiti)”.
   - Esiste un enum `StatiVisita`, ma nella classe `Visita` non risulta un campo stato né logica di transizione.

3. **Rollback luogo in UC06: ramo `visita == null` improbabile con il codice attuale**
   - `creaVisita(...)` restituisce sempre una nuova `Visita` (non ha percorsi di annullamento), ma i chiamanti gestiscono il caso `null` come annullamento: oggi sembra un residuo o una funzionalità prevista ma non presente.

4. **“Max iscrivibili da fruitore” gestito come dato generale, ma senza vincoli di validazione**
   - Il valore viene letto/scritto come intero senza controlli (es. non negatività). Se i requisiti prevedono vincoli, non sono applicati qui.