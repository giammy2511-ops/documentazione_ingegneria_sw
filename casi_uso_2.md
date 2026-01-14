# Attori e confini del sistema

## Attori (ruoli reali nel codice)
- **Configuratore**: utente autenticato con ruolo `"configuratore"`, accede a `MenuConfiguratore`.
- **Volontario**: utente autenticato con ruolo `"volontario"`, accede a `MenuVolontario`.
- **File system (directory `./DATA`)**: “sistema esterno” per la persistenza (JSON/TXT) gestita da `FileIO`.

> Nota: nei file compare anche il ruolo `"fruitore"` (es. in `Utente`, `DatiGenerali`), ma **non esiste un flusso di login/menu** per questo ruolo in `Main`. Quindi non è possibile derivare casi d’uso operativi del fruitore “da codice”.

## Confini del sistema
**Dentro**:
- Applicazione Java stand-alone da console (menu CLI).
- Gestione dati in memoria + persistenza su file in `./DATA` tramite `FileIO`.
- Logica applicativa: autenticazione, setup luoghi/visite (tipi visita), date precluse, disponibilità volontari, archiviazione tipi visita scaduti.

**Fuori**:
- Utente umano (configuratore/volontario) che inserisce input da console.
- File system (lettura/scrittura file).

## Lista candidata dei casi d’uso (estratta da menu/metodi di alto livello)
- Login e cambio credenziali primo accesso (`Main`).
- Inizializzazione “dati generali” (`Main` + `FileIO`).
- Setup iniziale: creazione luogo + prima visita; creazione visita associata a luogo (`MenuConfiguratore`).
- Gestione date precluse (inserimento/visualizzazione) (`MenuConfiguratore`).
- Modifica parametro “max iscrivibili” (`MenuConfiguratore` + `FileIO`).
- Consultazioni (volontari associati, luoghi, tipi visita, archivio storico) (`MenuConfiguratore`).
- Volontario: vedere tipi visita associati; inserire/consultare disponibilità (entro il 15) (`MenuVolontario`).
- Processo interno: sincronizzazione “attivo/scaduto” e archiviazione tipi visita scaduti (`GestoreDati.sincronizzaStatiAttiviEArchivio`).

---

# Elenco casi d’uso (tabella)

| ID | Titolo | Attore primario | Priorità | Stato |
|---|---|---|---|---|
| UC01 | Autenticarsi al sistema | Configuratore / Volontario | Alta | da codice |
| UC02 | Cambiare credenziali al primo accesso | Configuratore / Volontario | Alta | da codice |
| UC03 | Inizializzare i dati generali | Configuratore | Media | da codice |
| UC04 | Creare un luogo con prima visita obbligatoria | Configuratore | Alta | da codice |
| UC05 | Creare una visita e associarla a un luogo | Configuratore | Alta | da codice |
| UC06 | Gestire volontari durante la creazione visita | Configuratore | Media | da codice |
| UC07 | Inserire e consultare date precluse | Configuratore | Media | da codice |
| UC08 | Modificare numero massimo iscrivibili da fruitore | Configuratore | Bassa | da codice |
| UC09 | Visualizzare volontari con tipi visita associati | Configuratore | Bassa | da codice |
| UC10 | Visualizzare elenco luoghi visitabili (attivi) | Configuratore | Bassa | da codice |
| UC11 | Visualizzare elenco tipi di visita (attivi) | Configuratore | Bassa | da codice |
| UC12 | Consultare archivio storico dei tipi visita scaduti | Configuratore | Bassa | da codice |
| UC13 | Visualizzare i tipi di visita associati | Volontario | Media | da codice |
| UC14 | Esprimere disponibilità per il mese entrante | Volontario | Alta | da codice |
| UC15 | Sincronizzare scadenze e archiviazione tipi visita | Sistema (interno) | Media | da codice |

---

## UC01 — Autenticarsi al sistema
**Attori primari:** Configuratore, Volontario  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** Accedere al sistema con credenziali valide e ruolo supportato.  
**Precondizioni:** Esiste `./DATA` (o è creabile). Esistono utenti persistiti (o lista vuota).  
**Postcondizioni (successo):** `utenteCorrente` è impostato e il menu del ruolo viene avviato.  
**Postcondizioni (fallimento):** Nessun menu avviato; viene richiesto di reinserire credenziali.  
**Trigger:** Avvio applicazione (`Main.main`).  

### Flusso principale
1. L’attore avvia l’applicazione.
2. Il sistema carica i dati da directory `./DATA`.
3. Il sistema richiede **nome utente** e **password**.
4. Il sistema valida le credenziali cercando un utente con:
   - username e password corrispondenti
   - ruolo `"configuratore"` oppure `"volontario"`
   - se volontario: utente **attivo**
5. Se la validazione ha esito positivo, il sistema imposta l’utente corrente.

### Flussi alternativi / eccezioni
- **A1 — Credenziali errate o utente non attivo**:
  1. Condizione: username/password non corrispondono a nessun utente valido **oppure** volontario `attivo=false`.
  2. Passi: il sistema stampa “Utente non attivo o credenziali errate.” e torna al passo di inserimento.
  3. Esito: autenticazione non completata.
- **A2 — Errore di I/O in caricamento dati**:
  1. Condizione: eccezione `IOException` durante `GestoreDati.caricaDaDirectory`.
  2. Passi: il sistema stampa stack trace e termina.
  3. Esito: autenticazione non disponibile.

### Regole di business / vincoli
- RB1: Sono supportati **solo** i ruoli `"configuratore"` e `"volontario"` per l’accesso ai menu.
- RB2: Un volontario non attivo non può autenticarsi (`GestoreDati.autentica`).

### Dati coinvolti
- Input: username, password  
- Output: messaggi di esito, avvio menu ruolo  
- Entità/modelli: `Utente`, `GestoreDati`, `FileIO`

### Tracciabilità
- **Codice:** `Main.login`, `Main.verificaCredenziali`, `GestoreDati.autentica`, `GestoreDati.caricaDaDirectory`, `FileIO.leggiUtenti`  
- **Requisiti/slide:** riferimenti non forniti in questa chat.

---

## UC02 — Cambiare credenziali al primo accesso
**Attori primari:** Configuratore, Volontario  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** Forzare il cambio password al primo accesso (e cambio username solo per configuratore).  
**Precondizioni:** Utente autenticato; `utente.isPrimoAccesso() == true`.  
**Postcondizioni (successo):** Credenziali aggiornate; `primoAccesso=false` salvato su file.  
**Postcondizioni (fallimento):** Credenziali non modificate; l’utente resta in primo accesso.  
**Trigger:** Dopo UC01, se `utenteCorrente.isPrimoAccesso()`.

### Flusso principale
1. Il sistema verifica che l’utente sia al primo accesso.
2. Se l’utente è configuratore, il sistema richiede **nuovo nome utente**.
3. Il sistema richiede **nuova password**.
4. Il sistema valida che la nuova password:
   - non sia vuota
   - sia diversa dalla password attuale
5. Il sistema aggiorna le credenziali dell’utente e salva l’elenco utenti su file.

### Flussi alternativi / eccezioni
- **A1 — Volontario: username non modificabile**:
  1. Condizione: ruolo `"volontario"`.
  2. Passi: il sistema imposta `nuovoNome = nomeUtente corrente` senza chiederlo.
  3. Esito: viene cambiata solo la password.
- **A2 — Username già esistente (solo configuratore)**:
  1. Condizione: nuovo username diverso dal precedente e già presente.
  2. Passi: il sistema mostra errore e richiede di riprovare.
  3. Esito: nessuna modifica salvata.
- **A3 — Password uguale alla precedente / vuota**:
  1. Condizione: password nuova non valida.
  2. Passi: il sistema stampa messaggio e richiede reinserimento.
  3. Esito: finché non valida, non salva.
- **A4 — Errore di I/O nel salvataggio utenti**:
  1. Condizione: `IOException` durante `fileIO.salvaUtenti`.
  2. Passi: stack trace; operazione fallisce.
  3. Esito: credenziali potrebbero essere mutate in memoria ma non persistite (rischio incoerenza).

### Regole di business / vincoli
- RB1: Volontario non può cambiare username (`GestoreDati.aggiornaCredenziali` forza il vecchio).
- RB2: Password nuova deve essere diversa dalla corrente.

### Dati coinvolti
- Input: nuovo username (solo configuratore), nuova password  
- Output: conferma o errore  
- Entità/modelli: `Utente`, `GestoreDati`, `FileIO`

### Tracciabilità
- **Codice:** `Main.cambioCredenziali`, `GestoreDati.aggiornaCredenziali`, `FileIO.salvaUtenti`  
- **Requisiti/slide:** riferimenti non forniti in questa chat.

---

## UC03 — Inizializzare i dati generali
**Attori primari:** Configuratore  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** Creare e salvare i parametri generali se non presenti.  
**Precondizioni:** Autenticazione avvenuta; `FileIO.leggiDatiGenerali()` restituisce `null`.  
**Postcondizioni (successo):** `generale.txt` popolato con ambito territoriale e max.  
**Postcondizioni (fallimento):** Dati generali assenti o non validi; nessun salvataggio.  
**Trigger:** Dopo UC01/UC02, chiamata a `Main.chiediDatiGenerali`.

### Flusso principale
1. Il sistema legge i dati generali da file.
2. Se i dati sono assenti e l’utente è configuratore, il sistema richiede:
   - ambito territoriale (stringa non vuota)
   - numero massimo iscrivibili (intero)
3. Il sistema salva i dati generali su file.
4. Il sistema stampa conferma di inizializzazione.

### Flussi alternativi / eccezioni
- **A1 — Dati già presenti**:
  1. Condizione: `leggiDatiGenerali()` restituisce un oggetto valido.
  2. Passi: il sistema stampa ambito e massimo e non richiede input.
  3. Esito: nessuna modifica.
- **A2 — Utente non configuratore e dati assenti**:
  1. Condizione: ruolo `"volontario"` e `dati == null`.
  2. Passi: il sistema stampa messaggio “Accedi come configuratore…” e termina la procedura.
  3. Esito: dati restano assenti.
- **A3 — Errore I/O in lettura/scrittura**:
  1. Condizione: `IOException`.
  2. Passi: stack trace.
  3. Esito: inizializzazione fallita.

### Regole di business / vincoli
- RB1: Solo il configuratore può inizializzare i dati generali se assenti.

### Dati coinvolti
- Input: ambito territoriale, max iscrivibili  
- Output: conferma o messaggio informativo  
- Entità/modelli: `DatiGenerali`, `FileIO`

### Tracciabilità
- **Codice:** `Main.chiediDatiGenerali`, `FileIO.leggiDatiGenerali`, `FileIO.salvaDatiGenerali`, `DatiGenerali`  
- **Requisiti/slide:** riferimenti non forniti in questa chat.

---

## UC04 — Creare un luogo con prima visita obbligatoria
**Attori primari:** Configuratore  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** Inserire un nuovo luogo e almeno un tipo di visita associato (setup iniziale).  
**Precondizioni:** Utente configuratore autenticato.  
**Postcondizioni (successo):** Luogo persistito e almeno una visita associata salvata.  
**Postcondizioni (fallimento):** Se la visita non viene creata/validata, il luogo viene rimosso e non resta persistito.  
**Trigger:** Dal menu configuratore → (se non inizializzato) “Setup iniziale” → “Crea luogo + prima visita”.

### Flusso principale
1. L’attore seleziona “Crea luogo + prima visita (obbligatoria)”.
2. Il sistema richiede i dati del luogo (nome, descrizione, latitudine, longitudine).
3. Il sistema verifica che non esista già un luogo con **stesso ID** (derivato da nome+coordinate).
4. Il sistema salva il nuovo luogo.
5. Il sistema richiede i dati della visita (titolo, descrizione, punto incontro, min/max partecipanti, biglietto, volontari, scheduling).
6. Il sistema tenta di aggiungere la visita al calendario verificando sovrapposizioni nel luogo.
7. Se l’aggiunta riesce, il sistema salva visite e luoghi e conferma la creazione.

### Flussi alternativi / eccezioni
- **A1 — Luogo duplicato**:
  1. Condizione: esiste già un luogo con stesso ID (nome+coordinate).
  2. Passi: il sistema chiede di reinserire i dati del luogo.
  3. Esito: nessun luogo creato finché non unico.
- **A2 — Overlap visite nello stesso luogo**:
  1. Condizione: `GestoreCalendario.aggiungiVisita` restituisce `false`.
  2. Passi: il sistema mostra menu “OVERLAP rilevato…”, l’attore può scegliere di reinserire solo lo scheduling.
  3. Esito: visita creata solo quando non sovrapposta.
- **A3 — Annullamento dopo overlap**:
  1. Condizione: l’attore sceglie “0 = annulla creazione visita” nel menu overlap.
  2. Passi: il sistema rimuove il luogo appena creato e salva la lista luoghi senza di esso.
  3. Esito: nessun luogo/visita creati.
- **A4 — Errori I/O in salvataggio**:
  1. Condizione: `IOException` durante `salvaLuoghi`/`salvaVisite`.
  2. Passi: stack trace.
  3. Esito: possibile creazione parziale (dipende dal punto di errore).

### Regole di business / vincoli
- RB1: ID luogo = `nome-lat-lon` (`Luogo.creaID`); deve essere unico.
- RB2: Ogni luogo creato in setup deve avere almeno una visita associata (se no viene rimosso).
- RB3: Nel medesimo luogo, due visite non possono sovrapporsi temporalmente in alcun giorno comune (`GestoreCalendario`).
- RB4: Almeno un volontario deve essere selezionato per la visita (`menuSceltaVolontari` impone “almenoUno”).
- RB5: Scheduling valido: data fine ≥ data inizio; data inizio ≥ oggi; selezione di almeno un giorno (`menuSceltaGiorniSettimana`).

### Dati coinvolti
- Input: dati luogo; dati visita; scheduling (date, ora, durata, giorni)  
- Output: conferma o errori/ri-tentativi  
- Entità/modelli: `Luogo`, `Visita`, `PuntoIncontro`, `SchedulingVisita`, `GestoreCalendario`, `FileIO`

### Tracciabilità
- **Codice:** `MenuConfiguratore.menuSetupIniziale`, `MenuConfiguratore.creaLuogoConVisitaObbligatoria`, `MenuConfiguratore.creaLuogo`, `MenuConfiguratore.creaVisita`, `MenuConfiguratore.menuOverlap`, `GestoreDati.aggiungiLuogo`, `GestoreDati.aggiungiVisitaALuogo`, `GestoreCalendario.aggiungiVisita`, `FileIO.salvaLuoghi`, `FileIO.salvaVisite`  
- **Requisiti/slide:** riferimenti non forniti in questa chat.

---

## UC05 — Creare una visita e associarla a un luogo
**Attori primari:** Configuratore  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** Inserire un tipo di visita e associarlo a un luogo esistente o a un nuovo luogo.  
**Precondizioni:** Utente configuratore autenticato.  
**Postcondizioni (successo):** Visita salvata e associata a un luogo; sincronizzazione stati/archivio eseguita.  
**Postcondizioni (fallimento):** Visita non aggiunta (overlap o annullamento) e nessuna persistenza della visita.  
**Trigger:** Menu configuratore → Setup iniziale → “Crea visita (associa subito a luogo…)”.

### Flusso principale
1. L’attore seleziona “Crea visita (associa…)”.
2. Il sistema chiede se associare a luogo esistente o crearne uno nuovo.
3. Se luogo esistente:
   1. Il sistema mostra l’elenco luoghi.
   2. L’attore seleziona un luogo.
   3. Se il luogo era inattivo, il sistema lo riattiva e salva.
4. Il sistema raccoglie i dati della visita e verifica che il titolo non sia già usato nello stesso luogo.
5. Il sistema tenta di aggiungere la visita verificando sovrapposizioni temporali.
6. In caso di successo, il sistema salva visite e luoghi e conferma.

### Flussi alternativi / eccezioni
- **A1 — Nessun luogo esistente**:
  1. Condizione: lista luoghi vuota e l’attore sceglie “associa a luogo esistente”.
  2. Passi: il sistema informa e forza la creazione “luogo + visita” (rimando a UC04).
  3. Esito: si procede tramite UC04.
- **A2 — Titolo visita duplicato nel luogo**:
  1. Condizione: `GestoreDati.esisteVisitaConTitoloNelLuogo(idLuogo, titolo) == true`.
  2. Passi: il sistema richiede reinserimento del titolo.
  3. Esito: visita creata solo con titolo univoco nel luogo.
- **A3 — Overlap**: come UC04/A2 (menu overlap + reinserimento scheduling).
- **A4 — Annullamento**:
  1. Condizione: l’attore termina il menu o annulla al menu overlap.
  2. Passi: il sistema non salva la visita.
  3. Esito: nessuna modifica persistita sulla visita.

### Regole di business / vincoli
- RB1: Titolo visita deve essere univoco **per luogo** (check su `idLuogo` + `titolo`).
- RB2: No sovrapposizioni per luogo su giorni comuni (`GestoreCalendario`).
- RB3: Riattivazione luogo se selezionato e inattivo (`setAttivo(true)` + `salvaLuoghi`).

### Dati coinvolti
- Input: scelta luogo; dati visita; scheduling  
- Output: conferma/errore  
- Entità/modelli: `Luogo`, `Visita`, `GestoreDati`, `GestoreCalendario`, `FileIO`

### Tracciabilità
- **Codice:** `MenuConfiguratore.creaVisitaConAssociazioneObbligatoria`, `MenuConfiguratore.scegliLuogoEsistente`, `MenuConfiguratore.creaVisita`, `GestoreDati.esisteVisitaConTitoloNelLuogo`, `GestoreCalendario.aggiungiVisita`  
- **Requisiti/slide:** riferimenti non forniti in questa chat.

---

## UC06 — Gestire volontari durante la creazione visita
**Attori primari:** Configuratore  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** Selezionare volontari per una visita, con possibilità di creare un nuovo volontario o riattivare uno esistente.  
**Precondizioni:** Configuratore sta creando una visita (UC04/UC05).  
**Postcondizioni (successo):** Lista volontari associati alla visita valorizzata; eventuali volontari creati/riattivati salvati.  
**Postcondizioni (fallimento):** Se non si seleziona alcun volontario, il sistema blocca la chiusura della selezione.  
**Trigger:** Durante `MenuConfiguratore.creaVisita` → `menuSceltaVolontari`.

### Flusso principale
1. Il sistema mostra la lista dei volontari selezionabili + opzione “Aggiungi nuovo volontario”.
2. L’attore seleziona uno o più volontari.
3. Il sistema impedisce di terminare la selezione finché non è stato scelto almeno un volontario.
4. Il sistema restituisce l’elenco dei nickname scelti da associare alla visita.

### Flussi alternativi / eccezioni
- **A1 — Creare nuovo volontario**:
  1. Condizione: l’attore seleziona “Aggiungi nuovo volontario”.
  2. Passi: il sistema richiede username e password predefinita; verifica username univoco; crea l’utente con `primoAccesso=true`.
  3. Esito: nuovo volontario salvato e automaticamente selezionato.
- **A2 — Volontario selezionato ma inattivo**:
  1. Condizione: l’attore seleziona un volontario con `attivo=false`.
  2. Passi: il sistema lo riattiva (`setAttivo(true)`) e salva utenti.
  3. Esito: volontario attivo e selezionato.
- **A3 — Username già esistente (creazione volontario)**:
  1. Condizione: `GestoreDati.esisteNomeUtente(username) == true`.
  2. Passi: il sistema chiede di inserirne uno diverso.
  3. Esito: creazione completata solo con username unico.
- **A4 — Errore I/O nel salvataggio**:
  1. Condizione: `IOException` durante `salvaUtenti`.
  2. Passi: stampa errore.
  3. Esito: volontario potrebbe non risultare persistito.

### Regole di business / vincoli
- RB1: Almeno un volontario deve essere associato a ogni visita creata.
- RB2: Username dei volontari deve essere univoco.
- RB3: I nuovi volontari vengono creati con `primoAccesso=true` (cambio credenziali obbligatorio al primo login).

### Dati coinvolti
- Input: selezione volontari; (eventuale) username/password predefinita  
- Output: lista nickname volontari per la visita; messaggi di conferma  
- Entità/modelli: `Utente`, `Visita`, `GestoreDati`, `FileIO`

### Tracciabilità
- **Codice:** `MenuConfiguratore.menuSceltaVolontari`, `MenuConfiguratore.inserimentoCredenzialiNuovoVolontario`, `GestoreDati.creaVolontario`, `GestoreDati.esisteNomeUtente`, `FileIO.salvaUtenti`  
- **Requisiti/slide:** riferimenti non forniti in questa chat.

---

## UC07 — Inserire e consultare date precluse
**Attori primari:** Configuratore  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** Gestire le date non selezionabili (precluse) per uno specifico mese futuro.  
**Precondizioni:** Configuratore autenticato.  
**Postcondizioni (successo):** Date aggiunte persistite su `datePrecluse.txt`.  
**Postcondizioni (fallimento):** Nessuna data aggiunta se duplicata o se errore I/O.  
**Trigger:** Menu configuratore → “Inserire date precluse”.

### Flusso principale
1. L’attore entra nella funzione date precluse.
2. Il sistema determina il mese obiettivo:
   - se oggi è **dal 16 in poi**: mese obiettivo = **tra 3 mesi** (rispetto al mese corrente)
   - se oggi è **entro il 15**: mese obiettivo = **tra 2 mesi** (perché usa il mese precedente come base)
3. Il sistema mostra un menu: aggiungere data / visualizzare date precluse.
4. Se aggiunta:
   1. L’attore inserisce il giorno (1..fine mese obiettivo).
   2. Il sistema verifica che la data non sia già preclusa.
   3. Il sistema aggiunge la data all’insieme e la appende al file.
5. Se visualizzazione:
   1. Il sistema filtra le date precluse del mese obiettivo e le stampa (ordinate).

### Flussi alternativi / eccezioni
- **A1 — Data già preclusa**:
  1. Condizione: la data è già presente nel set.
  2. Passi: il sistema stampa “Data già preclusa.” e non salva.
  3. Esito: nessun cambiamento.
- **A2 — Nessuna data preclusa nel mese**:
  1. Condizione: filtro vuoto.
  2. Passi: il sistema stampa messaggio “Non ci sono date…”.
  3. Esito: sola consultazione.
- **A3 — Errore I/O in append**:
  1. Condizione: `IOException` su `appendDataPreclusa`.
  2. Passi: stack trace.
  3. Esito: la data potrebbe essere in memoria ma non persistita (incoerenza potenziale).

### Regole di business / vincoli
- RB1: Non si possono inserire duplicati nello stesso insieme di date precluse.
- RB2: L’ambito temporale gestito qui è un **mese specifico futuro** calcolato dalla data corrente.

### Dati coinvolti
- Input: giorno del mese obiettivo  
- Output: elenco date, conferme  
- Entità/modelli: `GestoreDati.datePrecluse`, `FileIO.appendDataPreclusa`

### Tracciabilità
- **Codice:** `MenuConfiguratore.inserimentoDatePrecluse`, `GestoreDati.getDatePrecluse`, `FileIO.appendDataPreclusa`  
- **Requisiti/slide:** riferimenti non forniti in questa chat.

---

## UC08 — Modificare numero massimo iscrivibili da fruitore
**Attori primari:** Configuratore  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** Aggiornare il parametro `maxPersoneIscrivibiliDaFruitore` nei dati generali.  
**Precondizioni:** File `generale.txt` leggibile (idealmente già inizializzato).  
**Postcondizioni (successo):** Dati generali salvati con nuovo massimo.  
**Postcondizioni (fallimento):** Nessuna modifica persistita se errore I/O.  
**Trigger:** Menu configuratore → “Modificare numero max iscrivibili da fruitore”.

### Flusso principale
1. L’attore seleziona la voce di modifica.
2. Il sistema legge i dati generali correnti.
3. Il sistema richiede il nuovo valore massimo (intero).
4. Il sistema salva i dati generali aggiornati su file.

### Flussi alternativi / eccezioni
- **A1 — Dati generali non presenti**:
  1. Condizione: `leggiDatiGenerali()` restituisce `null`.
  2. Passi: il sistema andrebbe in errore (`dati.ambitoTerritoriale()`), quindi l’UC è fragile.
  3. Esito: possibile eccezione runtime.
- **A2 — Errore I/O**:
  1. Condizione: `IOException`.
  2. Passi: stack trace.
  3. Esito: nessun salvataggio.

### Regole di business / vincoli
- RB1: Il valore è acquisito come intero senza vincoli ulteriori (nessun range esplicito nel menu).

### Dati coinvolti
- Input: nuovo massimo  
- Output: nessun output strutturato oltre eventuali stack trace  
- Entità/modelli: `DatiGenerali`, `FileIO`

### Tracciabilità
- **Codice:** `MenuConfiguratore.modificaNumeroMaxIscrivibili`, `FileIO.leggiDatiGenerali`, `FileIO.salvaDatiGenerali`, `DatiGenerali`  
- **Requisiti/slide:** riferimenti non forniti in questa chat.

---

## UC09 — Visualizzare volontari con tipi visita associati
**Attori primari:** Configuratore  
**Scopo:** Consultare, per ogni volontario attivo, i tipi visita attivi a cui risulta associato.  
**Precondizioni:** Nessuna (ma devono esistere utenti/visite per avere risultati).  
**Postcondizioni (successo):** Stampa a console della mappa volontario → titoli visite.  
**Trigger:** Menu configuratore → “Visualizzare elenco volontari con visite associate”.

### Flusso principale
1. L’attore seleziona la voce di consultazione.
2. Il sistema costruisce una mappa dei soli utenti con ruolo volontario e `attivo=true`.
3. Il sistema scorre i tipi visita presenti nel calendario e, per ciascuna visita attiva, aggiunge il titolo ai volontari indicati.
4. Il sistema stampa:
   - “nessun tipo visita attivo” se il volontario non ha associazioni attive
   - oppure la lista dei titoli.

### Flussi alternativi / eccezioni
- **A1 — Nessun tipo visita nel calendario**:
  1. Condizione: `calendario().getVisite() == null`.
  2. Passi: stampa “Non ci sono tipi visita.”
  3. Esito: consultazione termina.
- **A2 — Nessun volontario attivo**:
  1. Condizione: mappa vuota.
  2. Passi: stampa “Non ci sono volontari attivi.”
  3. Esito: consultazione termina.

### Regole di business / vincoli
- RB1: Considera solo volontari **attivi**.
- RB2: Considera solo visite **attive**.

### Dati coinvolti
- Input: nessuno  
- Output: righe testuali per volontario  
- Entità/modelli: `Utente`, `Visita`, `GestoreCalendario`

### Tracciabilità
- **Codice:** `MenuConfiguratore.mostraElencoVolontariAssociatiVisita`, `GestoreDati.getUtenti`, `GestoreDati.getGestoreCalendario`  
- **Requisiti/slide:** riferimenti non forniti in questa chat.

---

## UC10 — Visualizzare elenco luoghi visitabili (attivi)
**Attori primari:** Configuratore  
**Scopo:** Consultare l’elenco dei luoghi attivi (con almeno una visita attiva).  
**Precondizioni:** Nessuna.  
**Postcondizioni (successo):** Stampa elenco nomi luoghi attivi.  
**Trigger:** Menu configuratore → “Visualizzare elenco luoghi visitabili”.

### Flusso principale
1. L’attore seleziona la voce.
2. Il sistema filtra i luoghi con `attivo=true`.
3. Il sistema stampa i nomi dei luoghi attivi.

### Flussi alternativi / eccezioni
- **A1 — Nessun luogo attivo**:
  1. Condizione: lista filtrata vuota.
  2. Passi: stampa “Non esistono luoghi attivi.”
  3. Esito: termina.

### Regole di business / vincoli
- RB1: Un luogo è attivo se ha almeno una visita attiva (`Luogo.aggiornaAttivoDaVisite`).

### Dati coinvolti
- Input: nessuno  
- Output: elenco luoghi  
- Entità/modelli: `Luogo`

### Tracciabilità
- **Codice:** `MenuConfiguratore.mostraElencoLuoghi`, `Luogo.isAttivo`  
- **Requisiti/slide:** riferimenti non forniti in questa chat.

---

## UC11 — Visualizzare elenco tipi di visita (attivi)
**Attori primari:** Configuratore  
**Scopo:** Consultare i tipi visita attivi, raggruppati per luogo attivo.  
**Precondizioni:** Nessuna.  
**Postcondizioni (successo):** Stampa dei tipi visita attivi con descrizione semplificata.  
**Trigger:** Menu configuratore → “Visualizzare l’elenco dei tipi di visita”.

### Flusso principale
1. L’attore seleziona la voce.
2. Il sistema scorre i luoghi attivi.
3. Per ogni luogo, filtra le visite attive.
4. Il sistema stampa:
   - nome luogo
   - per ogni visita attiva: `toStringSemplificato()`.

### Flussi alternativi / eccezioni
- **A1 — Nessun tipo visita attivo**:
  1. Condizione: non viene stampato alcun elemento.
  2. Passi: stampa “Non esistono tipi di visita attivi.”
  3. Esito: termina.

### Regole di business / vincoli
- RB1: Mostra solo visite con `attivo=true`.

### Dati coinvolti
- Input: nessuno  
- Output: dettagli semplificati visite  
- Entità/modelli: `Luogo`, `Visita`

### Tracciabilità
- **Codice:** `MenuConfiguratore.mostraElencoTipiVisita`, `Visita.toStringSemplificato`  
- **Requisiti/slide:** riferimenti non forniti in questa chat.

---

## UC12 — Consultare archivio storico dei tipi visita scaduti
**Attori primari:** Configuratore  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** Visualizzare i tipi visita scaduti archiviati.  
**Precondizioni:** File archivio esistente oppure inizializzabile vuoto.  
**Postcondizioni (successo):** Stampa `toString()` dell’archivio storico.  
**Trigger:** Menu configuratore → “visualizza archivio storico”.

### Flusso principale
1. L’attore seleziona la voce.
2. Il sistema legge `archivioStorico.json`.
3. Il sistema stampa la rappresentazione testuale dell’archivio.

### Flussi alternativi / eccezioni
- **A1 — Archivio non presente/vuoto**:
  1. Condizione: file non esiste o è vuoto.
  2. Passi: `FileIO.leggiArchivioStorico()` restituisce un `ArchivioStorico` vuoto.
  3. Esito: stampa di un archivio senza elementi.
- **A2 — Errore I/O**:
  1. Condizione: `IOException`.
  2. Passi: stack trace.
  3. Esito: consultazione fallita.

### Regole di business / vincoli
- RB1: L’archivio contiene “tipi visita scaduti” (non più attivi) come da logica di sincronizzazione.

### Dati coinvolti
- Input: nessuno  
- Output: stampa archivio  
- Entità/modelli: `ArchivioStorico`, `Visita`

### Tracciabilità
- **Codice:** `MenuConfiguratore.stampaArchivio`, `FileIO.leggiArchivioStorico`, `ArchivioStorico`  
- **Requisiti/slide:** riferimenti non forniti in questa chat.

---

## UC13 — Visualizzare i tipi di visita associati
**Attori primari:** Volontario  
**Scopo:** Consultare i tipi visita in cui il volontario è indicato come guida.  
**Precondizioni:** Volontario autenticato.  
**Postcondizioni (successo):** Stampa dei dettagli semplificati delle visite associate.  
**Trigger:** Menu volontario → “Visualizza i tipi di visita associati”.

### Flusso principale
1. L’attore seleziona “Visualizza i tipi di visita associati”.
2. Il sistema scorre tutti i luoghi e le loro visite.
3. Il sistema seleziona le visite che contengono il nickname del volontario nella lista volontari.
4. Il sistema stampa per ogni visita: `toStringSemplificato()`.

### Flussi alternativi / eccezioni
- **A1 — Nessuna visita associata**:
  1. Condizione: lista risultante vuota.
  2. Passi: stampa “Non risulti associato ad alcun tipo di visita.”
  3. Esito: termina.

### Regole di business / vincoli
- RB1: L’associazione è basata su match esatto del nickname in `Visita.volontari`.

### Dati coinvolti
- Input: nessuno  
- Output: dettagli visite  
- Entità/modelli: `Luogo`, `Visita`

### Tracciabilità
- **Codice:** `MenuVolontario.visualizzaTipiVisitaAssociati`, `MenuVolontario.getTipiVisitaAssociati`, `Visita.toStringSemplificato`  
- **Requisiti/slide:** riferimenti non forniti in questa chat.

---

## UC14 — Esprimere disponibilità per il mese entrante
**Attori primari:** Volontario  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** Inserire date di disponibilità per il mese successivo, entro il giorno 15.  
**Precondizioni:** Volontario autenticato; data odierna con giorno ≤ 15; esiste almeno un tipo visita programmabile nel mese entrante.  
**Postcondizioni (successo):** Una o più date salvate in `disponibilitaVolontari.txt` per il volontario.  
**Postcondizioni (fallimento):** Nessuna data salvata se raccolta chiusa o vincoli non soddisfatti.  
**Trigger:** Menu volontario → “Esprimi disponibilità (entro il 15)”.

### Flusso principale
1. L’attore seleziona “Esprimi disponibilità”.
2. Il sistema verifica che oggi sia entro il 15.
3. Il sistema calcola il **mese entrante** (primo giorno del mese successivo).
4. Il sistema verifica che esista almeno un giorno del mese entrante programmabile da almeno un tipo visita associato al volontario.
5. Il sistema presenta un menu:
   - aggiungi data
   - visualizza disponibilità inserite
   - visualizza date precluse del mese entrante
6. Se “aggiungi data”:
   1. Il sistema costruisce l’elenco di date selezionabili del mese entrante, escludendo:
      - date precluse
      - date già inserite dal volontario
      - date non compatibili con nessun tipo visita associato (`isProgrammabile`)
   2. L’attore seleziona una data.
   3. Il sistema salva la data nella disponibilità del volontario e persiste su file.
   4. Il sistema torna alla selezione (per aggiungerne altre).

### Flussi alternativi / eccezioni
- **A1 — Raccolta chiusa**:
  1. Condizione: oggi > 15.
  2. Passi: stampa “Raccolta disponibilità chiusa…”.
  3. Esito: termina senza modifiche.
- **A2 — Nessun tipo visita programmabile nel mese**:
  1. Condizione: `haAlmenoUnTipoProgrammabileNelMese` è falso.
  2. Passi: stampa “Non risulti associato… in questo periodo.”
  3. Esito: termina senza modifiche.
- **A3 — Nessuna data selezionabile**:
  1. Condizione: tutte le date sono precluse/già scelte/non compatibili.
  2. Passi: stampa “Non ci sono date selezionabili…”.
  3. Esito: termina inserimento.
- **A4 — Errore I/O nel salvataggio disponibilità**:
  1. Condizione: `IOException` in `salvaDisponibilitaVolontari`.
  2. Passi: stampa “Errore nel salvataggio…” + stack trace.
  3. Esito: inserimento fallito (in memoria potrebbe essere stata aggiunta la data).

### Regole di business / vincoli
- RB1: Inserimento consentito solo entro il 15 del mese corrente.
- RB2: Le date disponibili devono essere compatibili con almeno un tipo visita associato e programmabile in quel giorno.
- RB3: Non si possono inserire duplicati per lo stesso volontario.

### Dati coinvolti
- Input: selezione data dal calendario del mese entrante  
- Output: conferme, liste  
- Entità/modelli: `Visita` (per compatibilità), `GestoreDati.disponibilitaVolontari`, `FileIO.salvaDisponibilitaVolontari`

### Tracciabilità
- **Codice:** `MenuVolontario.inserisciDisponibilita`, `MenuVolontario.menuSceltaDisponibilitaMese`, `MenuVolontario.buildDateSelezionabili`, `Visita.isProgrammabile`, `GestoreDati.aggiungiDisponibilitaVolontario`, `FileIO.salvaDisponibilitaVolontari`  
- **Requisiti/slide:** riferimenti non forniti in questa chat.

---

## UC15 — Sincronizzare scadenze e archiviazione tipi visita
**Attori primari:** Sistema (interno)  
**Attori secondari:** File system (`./DATA`)  
**Scopo:** Rimuovere dal “calendario attivo” i tipi visita scaduti e archiviare quelli scaduti in archivio storico; aggiornare attivazione luoghi e volontari.  
**Precondizioni:** Dati caricati; visite e luoghi in memoria.  
**Postcondizioni (successo):** Tipi visita scaduti rimossi da `visite.json`, aggiunti a `archivioStorico.json`; luoghi/utenti aggiornati e salvati.  
**Postcondizioni (fallimento):** In caso di errori I/O, persistenza potrebbe essere parziale.  
**Trigger:** 
- Dopo il caricamento dati (`GestoreDati.caricaDaDirectory`), oppure
- Dopo l’aggiunta di una visita (`GestoreDati.aggiungiVisitaALuogo`).

### Flusso principale
1. Il sistema legge l’archivio storico (o ne crea uno vuoto).
2. Il sistema scorre la lista dei tipi visita nel calendario.
3. Per ciascuna visita, calcola se è scaduta (data fine < oggi) e aggiorna `attivo`.
4. Se una visita non è più attiva:
   1. Il sistema la aggiunge all’archivio storico **solo se** non è già presente (per `idTipoVisita`).
   2. Il sistema la rimuove dalla lista visite attive.
5. Se ci sono state modifiche:
   1. Il sistema salva `visite.json` e `archivioStorico.json`.
6. Il sistema ricostruisce le visite dentro i luoghi partendo dal calendario.
7. Il sistema aggiorna `attivo` dei luoghi in base alle visite attive.
8. Il sistema aggiorna `attivo` dei volontari in base alle visite attive associate.
9. Il sistema salva utenti e luoghi.

### Flussi alternativi / eccezioni
- **A1 — Liste null/assenza dati**:
  1. Condizione: visite nulle o archivio nullo.
  2. Passi: il sistema inizializza strutture vuote.
  3. Esito: procedura procede senza NPE.
- **A2 — Errore I/O in salvataggi multipli**:
  1. Condizione: `IOException` su uno dei salvataggi.
  2. Passi: propagazione/stack trace.
  3. Esito: rischio di persistenza incoerente tra file diversi.

### Regole di business / vincoli
- RB1: Una visita è “scaduta” se `dataFine < oggi` (`Visita.isScaduta`).
- RB2: L’archivio storico non deve contenere duplicati di tipi visita (controllo su `idTipoVisita`).
- RB3: Volontari senza alcuna visita attiva associata diventano `attivo=false`.

### Dati coinvolti
- Input: data odierna; visite attive; archivio storico  
- Output: file aggiornati; stati `attivo` aggiornati  
- Entità/modelli: `Visita`, `ArchivioStorico`, `Luogo`, `Utente`, `FileIO`

### Tracciabilità
- **Codice:** `GestoreDati.sincronizzaStatiAttiviEArchivio`, `Visita.aggiornaAttivoDaOggi`, `Visita.isScaduta`, `FileIO.leggiArchivioStorico`, `FileIO.salvaVisite`, `FileIO.salvaArchivioStorico`, `GestoreDati.aggiornaAttivoVolontariDaVisiteAttive`, `Luogo.aggiornaAttivoDaVisite`  
- **Requisiti/slide:** riferimenti non forniti in questa chat.

---

# Scostamenti e ambiguità

1. **Ruolo “fruitore” presente nei modelli ma non nei flussi reali**
   - In `Utente` si cita `"fruitore"` e `DatiGenerali` contiene `maxPersoneIscrivibiliDaFruitore`, ma `Main.avviaMenu()` gestisce solo `"configuratore"` e `"volontario"`.  
   - Impatto: eventuali UC del fruitore **non sono derivabili da codice** e andrebbero marcati “da requisiti” solo se mi fornisci documenti che li descrivono.

2. **“Garantire che i DatiGenerali siano presenti prima di avviare i menu” non è realmente garantito**
   - Commento in `Main` dichiara questa intenzione, ma il codice: se i dati generali mancano e accede un volontario, stampa un messaggio e **prosegue comunque** ad avviare `MenuVolontario`.  
   - Impatto: la precondizione “dati generali presenti” non è enforced.

3. **Stati visita (proposta/completa/confermata/cancellata/effettuata) non implementati nei menu**
   - Esiste l’enum `StatiVisita`, ma `MenuConfiguratore.mostraStatoVisite()` stampa “Versione 2: stati non ancora gestiti” e non ci sono UC operativi di transizione/gestione stati.