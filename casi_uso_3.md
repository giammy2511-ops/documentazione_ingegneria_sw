# Attori e confini del sistema

## Attori (ruoli reali)
- **Configuratore**: utente autenticato con ruolo `"configuratore"`. Gestisce setup, dati e ciclo mensile (raccolta/piano).
- **Volontario**: utente autenticato con ruolo `"volontario"`. Inserisce disponibilità nel mese di raccolta e consulta i tipi visita associati.
- **Fruitore** *(solo modellato, non esposto da menu)*: ruolo `"fruitore"` esiste nel modello utente, ma **non c’è alcun menu/flow attivabile** dall’app console.
- **Sistema di persistenza su file (File system)**: lettura/scrittura dati applicativi nella directory `./DATA`.
- **Orologio di sistema**: la logica di finestra temporale dipende da `LocalDate.now()` (aperture/chiusure e mese target).

## Confini del sistema
**Dentro il sistema**
- Applicazione Java **stand-alone da console** (CLI).
- Autenticazione semplice (username/password su file).
- Gestione luoghi e tipi visita.
- Raccolta disponibilità volontari.
- Produzione e consultazione del piano visite (istanze).
- Archiviazione “storica” dei tipi visita scaduti (archivio).

**Fuori dal sistema**
- Nessuna GUI, nessun DB, nessun servizio esterno.
- Nessuna gestione “fruitore” operativa da menu (anche se alcuni dati sono presenti come concetto).

---

# Elenco casi d’uso

| ID   | Titolo | Attore primario | Priorità | Stato |
|------|--------|------------------|----------|-------|
| UC01 | Autenticarsi | Configuratore / Volontario | Alta | da codice |
| UC02 | Cambiare credenziali al primo accesso | Configuratore / Volontario | Alta | da codice |
| UC03 | Inizializzare dati generali del sistema | Configuratore | Alta | da codice |
| UC04 | Avviare setup iniziale | Configuratore | Alta | da codice |
| UC05 | Creare luogo con prima visita obbligatoria | Configuratore | Alta | da codice |
| UC06 | Creare tipo visita associandolo a un luogo esistente | Configuratore | Alta | da codice |
| UC07 | Inserire (o creare) volontari associati a un tipo visita | Configuratore | Media | da codice |
| UC08 | Modificare numero massimo iscrivibili per fruitore | Configuratore | Media | da codice |
| UC09 | Consultare elenchi (luoghi / tipi visita / volontari associati) | Configuratore | Media | da codice |
| UC10 | Chiudere raccolta disponibilità del mese corrente di raccolta | Configuratore | Alta | da codice |
| UC11 | Produrre piano visite del mese di raccolta | Configuratore | Alta | da codice |
| UC12 | Gestire richieste di aggiunta/rimozione (dopo piano prodotto) | Configuratore | Media | da codice |
| UC13 | Inserire e consultare date precluse | Configuratore | Media | da codice |
| UC14 | Consultare piano visite prodotto | Configuratore | Media | da codice |
| UC15 | Inserire disponibilità come volontario | Volontario | Alta | da codice |
| UC16 | Consultare tipi visita associati come volontario | Volontario | Media | da codice |

> Nota: non compare alcun UC “Fruitore” perché nel codice fornito non esiste un menu/flow attivabile per quel ruolo.

---

## UC01 — Autenticarsi

**Attori primari:** Configuratore, Volontario  
**Attori secondari:** Sistema di persistenza su file  
**Scopo:** accedere al sistema con credenziali valide e ruolo supportato.  
**Precondizioni:** esiste un archivio utenti leggibile dai file dati.  
**Postcondizioni (successo):** l’utente è autenticato e viene avviato il menu del ruolo (configuratore o volontario).  
**Postcondizioni (fallimento):** nessun accesso; l’utente resta non autenticato.  
**Trigger:** avvio dell’applicazione.

### Flusso principale
1. L’attore avvia il programma.
2. Il sistema carica i dati dalla directory `./DATA`.
3. Il sistema richiede **nome utente** e **password**.
4. L’attore inserisce le credenziali.
5. Il sistema valida le credenziali contro l’archivio utenti.
6. Il sistema verifica che l’utente sia **attivo** (in particolare per i volontari).
7. Il sistema imposta l’utente come “corrente”.
8. Il sistema avvia il menu in base al ruolo:
   - se ruolo = configuratore → menu configuratore
   - se ruolo = volontario → menu volontario

### Flussi alternativi / eccezioni
- **A1 — Credenziali errate o utente non attivo**
  1. Condizione: autenticazione fallisce oppure l’utente risulta non attivo.
  2. Passi: il sistema mostra un messaggio di errore e richiede nuovamente le credenziali.
  3. Esito: si torna al passo 3 del flusso principale.
- **A2 — Ruolo non supportato dalla UI**
  1. Condizione: l’utente ha un ruolo diverso da `configuratore` o `volontario`.
  2. Passi: il sistema segnala “Ruolo non supportato”.
  3. Esito: l’app non avvia un menu operativo per quell’utente.

### Regole di business / vincoli
- RB1: configuratore e fruitore sono sempre considerati “attivi” nel modello; per il volontario l’attività dipende dalle visite attive a cui è associato.

### Dati coinvolti
- Input: username, password.
- Output: sessione applicativa (utente corrente), menu del ruolo.
- Entità/modelli: `Utente`, archivio utenti.

### Tracciabilità
- **Codice:** `Main.login()`, `Main.verificaCredenziali()`, `GestoreDati.autentica()`, `Main.avviaMenu()`, `Utente.aggiornaAttivoDaVisite(...)`
- **Requisiti/slide:** > Nota: non forniti in questa chat.

---

## UC02 — Cambiare credenziali al primo accesso

**Attori primari:** Configuratore, Volontario  
**Attori secondari:** Sistema di persistenza su file  
**Scopo:** sostituire le credenziali predefinite al primo accesso.  
**Precondizioni:** l’utente è autenticato e ha flag `primoAccesso = true`.  
**Postcondizioni (successo):** credenziali aggiornate e `primoAccesso` disattivato.  
**Postcondizioni (fallimento):** credenziali non aggiornate (resta necessario riprovare).  
**Trigger:** autenticazione riuscita di un utente al primo accesso.

### Flusso principale
1. Il sistema rileva che l’utente autenticato è al primo accesso.
2. Il sistema richiede le nuove credenziali.
3. L’attore inserisce la nuova password (e, se non volontario, anche il nuovo username).
4. Il sistema valida che la password sia confermata correttamente.
5. Il sistema salva le nuove credenziali su file.
6. Il sistema conferma l’aggiornamento e prosegue con l’avvio del menu.

### Flussi alternativi / eccezioni
- **A1 — Conferma password non corrispondente**
  1. Condizione: password e conferma differiscono.
  2. Passi: il sistema segnala l’errore e ripete la richiesta.
  3. Esito: ritorno al passo 2.
- **A2 — Errore di persistenza**
  1. Condizione: eccezione I/O durante il salvataggio.
  2. Passi: il sistema segnala errore e mantiene lo stato non aggiornato.
  3. Esito: l’attore deve riprovare (loop previsto nel chiamante).

### Regole di business / vincoli
- RB1: per i volontari, il nuovo username **non viene richiesto** (resta quello esistente); si cambia solo la password.

### Dati coinvolti
- Input: nuovo username (non volontario), nuova password, conferma password.
- Output: credenziali aggiornate su file.
- Entità/modelli: `Utente`.

### Tracciabilità
- **Codice:** `Main.cambioCredenziali()`, `GestoreDati.aggiornaCredenziali(...)`, `FileIO` (salvataggio utenti)
- **Requisiti/slide:** > Nota: non forniti in questa chat.

---

## UC03 — Inizializzare dati generali del sistema

**Attori primari:** Configuratore  
**Attori secondari:** Sistema di persistenza su file  
**Scopo:** definire i parametri generali necessari al funzionamento (ambito territoriale, max iscrizioni fruitore).  
**Precondizioni:** accesso come configuratore; dati generali assenti o non leggibili.  
**Postcondizioni (successo):** dati generali salvati e disponibili per esecuzioni successive.  
**Postcondizioni (fallimento):** dati generali non salvati.  
**Trigger:** avvio post-login quando i dati generali mancano.

### Flusso principale
1. Il sistema tenta di leggere i dati generali da file.
2. Il sistema rileva che i dati generali non sono presenti.
3. Il sistema verifica che l’utente corrente sia un configuratore.
4. L’attore inserisce:
   - ambito territoriale
   - numero massimo di persone iscrivibili da un fruitore
5. Il sistema salva i dati generali su file.

### Flussi alternativi / eccezioni
- **A1 — Utente non configuratore**
  1. Condizione: dati generali mancanti e ruolo ≠ configuratore.
  2. Passi: il sistema informa che serve il configuratore per inizializzare e termina l’esecuzione.
  3. Esito: uscita dal programma.
- **A2 — Errore di scrittura**
  1. Condizione: eccezione I/O durante il salvataggio.
  2. Passi: il sistema segnala errore (stack trace in console).
  3. Esito: dati non inizializzati.

### Regole di business / vincoli
- RB1: il “max iscrivibili” è un dato globale; nel codice fornito è **solo configurabile e persistito** (non risulta poi usato in flussi fruitore, perché non implementati).

### Dati coinvolti
- Input: `ambito`, `max`.
- Output: `DatiGenerali` persistito.
- Entità/modelli: `DatiGenerali`.

### Tracciabilità
- **Codice:** `Main.chiediDatiGenerali()`, `FileIO.leggiDatiGenerali()`, `FileIO.salvaDatiGenerali(...)`
- **Requisiti/slide:** > Nota: non forniti in questa chat.

---

## UC04 — Avviare setup iniziale

**Attori primari:** Configuratore  
**Attori secondari:** Sistema di persistenza su file  
**Scopo:** portare il sistema da “vuoto/non inizializzato” a “operativo” creando i primi dati minimi.  
**Precondizioni:** nessun luogo con almeno un tipo visita attivo (sistema non inizializzato).  
**Postcondizioni (successo):** esiste almeno un luogo e almeno un tipo visita associato.  
**Postcondizioni (fallimento):** dati minimi non garantiti (setup interrotto/annullato).  
**Trigger:** scelta “Setup iniziale” nel menu configuratore quando il sistema non è inizializzato.

### Flusso principale
1. Il configuratore apre il menu configuratore.
2. Il sistema rileva che il sistema non è inizializzato.
3. L’attore seleziona “Setup iniziale”.
4. Il sistema mostra le opzioni di setup:
   - creare un luogo con una visita obbligatoria
   - creare una visita con associazione obbligatoria a un luogo esistente
5. L’attore seleziona una delle opzioni.
6. Il sistema esegue il caso d’uso specifico (UC05 o UC06).

### Flussi alternativi / eccezioni
- **A1 — Errore I/O**
  1. Condizione: problemi di lettura/scrittura file durante setup.
  2. Passi: il sistema segnala errore.
  3. Esito: setup non completato.

### Regole di business / vincoli
- RB1: il sistema è considerato “inizializzato” se esiste almeno un luogo con lista visite non vuota.

### Dati coinvolti
- Input: selezione menu.
- Output: creazione dati minimi.
- Entità/modelli: `Luogo`, `Visita`.

### Tracciabilità
- **Codice:** `MenuConfiguratore.menu()`, `MenuConfiguratore.sistemaInizializzato()`, `MenuConfiguratore.menuSetupIniziale()`
- **Requisiti/slide:** > Nota: non forniti in questa chat.

---

## UC05 — Creare luogo con prima visita obbligatoria

**Attori primari:** Configuratore  
**Attori secondari:** Sistema di persistenza su file  
**Scopo:** inserire un nuovo luogo e garantirne l’utilizzabilità creando almeno un tipo visita associato.  
**Precondizioni:** autenticazione configuratore; dati luoghi caricati.  
**Postcondizioni (successo):** luogo creato e salvato; visita creata e associata al luogo; dati persistiti.  
**Postcondizioni (fallimento):** se la visita non viene creata, il luogo viene rimosso (rollback logico) e persistito senza il nuovo luogo.  
**Trigger:** scelta “Crea luogo con visita obbligatoria” nel setup o nelle modifiche.

### Flusso principale
1. L’attore seleziona l’operazione di creazione luogo.
2. Il sistema richiede i dati del luogo.
3. L’attore inserisce i dati del luogo.
4. Il sistema crea l’oggetto `Luogo` e lo aggiunge alla lista luoghi.
5. Il sistema richiede la creazione del **primo tipo visita** per quel luogo.
6. L’attore inserisce i dati della visita (titolo, scheduling, vincoli partecipanti, punto d’incontro, volontari…).
7. Il sistema valida vincoli (es. non duplicazione titolo nel luogo; coerenza date; overlap sullo stesso luogo).
8. Il sistema associa la visita al luogo e salva su file.
9. Il sistema conferma l’inserimento.

### Flussi alternativi / eccezioni
- **A1 — Annullamento/visita non creata**
  1. Condizione: la visita restituisce `null` (interruzione del flusso di creazione).
  2. Passi: il sistema rimuove il luogo appena creato e salva l’elenco luoghi aggiornato.
  3. Esito: nessun nuovo luogo persistito.
- **A2 — Sovrapposizione temporale (overlap) con altra visita dello stesso luogo**
  1. Condizione: il gestore calendario rileva una sovrapposizione su almeno un giorno comune.
  2. Passi: il sistema propone un menu “OVERLAP rilevato” con opzione di modifica parametri e riprova.
  3. Esito: l’attore modifica parametri della visita e il sistema ripete la validazione/associazione.
- **A3 — Errore di persistenza**
  1. Condizione: eccezione I/O nel salvataggio luoghi/visite.
  2. Passi: il sistema segnala errore.
  3. Esito: inserimento non garantito.

### Regole di business / vincoli
- RB1: un luogo “nuovo” deve avere almeno un tipo visita associato (vincolo imposto dal flusso).
- RB2: nello stesso luogo non possono coesistere tipi visita che risultano sovrapposti temporalmente in giorni comuni.

### Dati coinvolti
- Input: dati `Luogo`; dati `Visita` (incluso `SchedulingVisita`, punto incontro, vincoli partecipanti, volontari).
- Output: luogo e visita persistiti.
- Entità/modelli: `Luogo`, `Visita`, `SchedulingVisita`, `PuntoIncontro`, `GestoreCalendario`.

### Tracciabilità
- **Codice:** `MenuConfiguratore.creaLuogoConVisitaObbligatoria()`, `MenuConfiguratore.creaLuogo()`, `MenuConfiguratore.creaVisita(...)`, `GestoreDati.aggiungiLuogo(...)`, `GestoreDati.aggiungiVisitaALuogo(...)`, `GestoreCalendario.aggiungiVisitaSeNonOverlap(...)`, `FileIO.salvaLuoghi(...)`
- **Requisiti/slide:** > Nota: non forniti in questa chat.

---

## UC06 — Creare tipo visita associandolo a un luogo esistente

**Attori primari:** Configuratore  
**Attori secondari:** Sistema di persistenza su file  
**Scopo:** aggiungere un tipo visita a un luogo già presente.  
**Precondizioni:** esiste almeno un luogo; autenticazione configuratore.  
**Postcondizioni (successo):** tipo visita creato, associato e persistito; sincronizzazioni eventuali applicate.  
**Postcondizioni (fallimento):** nessuna nuova visita associata.  
**Trigger:** scelta “Crea visita con associazione obbligatoria” (setup o modifiche).

### Flusso principale
1. L’attore avvia la creazione visita.
2. Il sistema mostra la lista dei luoghi disponibili.
3. L’attore seleziona un luogo.
4. Il sistema richiede i dati del tipo visita (titolo, scheduling, vincoli, punto d’incontro, volontari).
5. L’attore inserisce i dati.
6. Il sistema valida:
   - coerenza date/ora/durata/giorni programmabili
   - assenza di duplicazione del titolo nello stesso luogo
   - assenza di overlap con altri tipi visita dello stesso luogo nei giorni comuni
7. Il sistema aggiunge la visita al luogo e salva su file.
8. Il sistema conferma l’inserimento.

### Flussi alternativi / eccezioni
- **A1 — Nessun luogo disponibile**
  1. Condizione: lista luoghi vuota.
  2. Passi: il sistema informa l’attore e termina l’operazione.
  3. Esito: nessuna visita creata.
- **A2 — Overlap rilevato**
  1. Condizione: sovrapposizione temporale con altra visita del luogo.
  2. Passi: il sistema offre modifica parametri e riprova.
  3. Esito: ripetizione dal passo 4.
- **A3 — Input non valido**
  1. Condizione: date non parseabili, violazione di range, ecc.
  2. Passi: il sistema ripete la richiesta (loop di input su data).
  3. Esito: ritorno al passo 4.

### Regole di business / vincoli
- RB1: vincolo anti-overlap per visite dello stesso luogo (su periodo comune).
- RB2: giorni programmabili non vuoti; durata positiva; dataInizio ≤ dataFine.

### Dati coinvolti
- Input: selezione luogo; dati del tipo visita.
- Output: tipo visita persistito.
- Entità/modelli: `Luogo`, `Visita`, `SchedulingVisita`, `GestoreCalendario`.

### Tracciabilità
- **Codice:** `MenuConfiguratore.creaVisitaConAssociazioneObbligatoria()`, `MenuConfiguratore.scegliLuogoEsistente()`, `MenuConfiguratore.creaVisita(...)`, `GestoreDati.aggiungiVisitaALuogo(...)`
- **Requisiti/slide:** > Nota: non forniti in questa chat.

---

## UC07 — Inserire (o creare) volontari associati a un tipo visita

**Attori primari:** Configuratore  
**Attori secondari:** Sistema di persistenza su file  
**Scopo:** associare almeno un volontario a un tipo visita (e, se serve, creare nuove credenziali volontario).  
**Precondizioni:** durante creazione/modifica di un tipo visita; autenticazione configuratore.  
**Postcondizioni (successo):** lista volontari del tipo visita valorizzata con almeno un nome utente.  
**Postcondizioni (fallimento):** la visita non supera la validazione “almeno un volontario”.  
**Trigger:** fase di scelta volontari dentro la creazione di un tipo visita.

### Flusso principale
1. Il sistema mostra i volontari esistenti selezionabili (escludendo quelli già scelti).
2. L’attore seleziona uno o più volontari da associare.
3. Il sistema impedisce la conclusione se non è stato selezionato almeno un volontario.
4. Il sistema salva l’associazione dentro l’oggetto visita (che poi verrà persistito con la visita).

### Flussi alternativi / eccezioni
- **A1 — Creazione nuovo volontario “al volo”**
  1. Condizione: l’attore seleziona l’opzione “Aggiungi nuovo volontario”.
  2. Passi: il sistema richiede username e password predefinita, crea il volontario e lo aggiunge alla selezione.
  3. Esito: nuovo volontario associato alla visita; al primo accesso dovrà cambiare credenziali.
- **A2 — Nessun volontario selezionato**
  1. Condizione: l’attore tenta di terminare la selezione senza aver scelto alcun volontario.
  2. Passi: il sistema blocca la chiusura e segnala “Devi selezionare almeno un volontario”.
  3. Esito: ritorno alla selezione.

### Regole di business / vincoli
- RB1: un tipo visita deve avere **almeno un volontario** associato (vincolo applicato nel menu di selezione).
- RB2: un volontario nuovo nasce con `primoAccesso=true` e dovrà cambiare credenziali al primo login.

### Dati coinvolti
- Input: selezione volontari; eventualmente username/password predefinita.
- Output: associazioni volontari nella visita; eventuale nuovo `Utente` volontario persistito.
- Entità/modelli: `Utente`, `Visita`.

### Tracciabilità
- **Codice:** `MenuConfiguratore` (selezione volontari + `inserimentoCredenzialiNuovoVolontario()`), `GestoreDati.creaVolontario(...)`, `Main.cambioCredenziali()` (obbligo al primo accesso)
- **Requisiti/slide:** > Nota: non forniti in questa chat.

---

## UC08 — Modificare numero massimo iscrivibili per fruitore

**Attori primari:** Configuratore  
**Attori secondari:** Sistema di persistenza su file  
**Scopo:** aggiornare il limite globale “max iscrivibili” per fruitore.  
**Precondizioni:** dati generali presenti; autenticazione configuratore.  
**Postcondizioni (successo):** nuovo valore salvato su file.  
**Postcondizioni (fallimento):** valore non aggiornato.  
**Trigger:** scelta dal “Menù generale” configuratore.

### Flusso principale
1. L’attore seleziona “Modificare numero max iscrivibili da fruitore”.
2. Il sistema legge i dati generali correnti.
3. L’attore inserisce il nuovo valore intero.
4. Il sistema salva i dati generali aggiornati.

### Flussi alternativi / eccezioni
- **A1 — Errore di persistenza**
  1. Condizione: eccezione I/O in lettura/scrittura.
  2. Passi: il sistema segnala errore.
  3. Esito: nessun aggiornamento garantito.

### Regole di business / vincoli
- RB1: il valore è un intero; non sono visibili vincoli di range nel codice mostrato per questo input (dipende da `InputDati.leggiIntero(...)`).

### Dati coinvolti
- Input: nuovo max.
- Output: `DatiGenerali` aggiornato.
- Entità/modelli: `DatiGenerali`.

### Tracciabilità
- **Codice:** `MenuConfiguratore.modificaNumeroMaxIscrivibili()`, `FileIO.leggiDatiGenerali()`, `FileIO.salvaDatiGenerali(...)`
- **Requisiti/slide:** > Nota: non forniti in questa chat.

---

## UC09 — Consultare elenchi (luoghi / tipi visita / volontari associati)

**Attori primari:** Configuratore  
**Scopo:** consultare dati anagrafici/di configurazione caricati nel sistema.  
**Precondizioni:** autenticazione configuratore; dati caricati.  
**Postcondizioni (successo):** elenchi stampati in console.  
**Postcondizioni (fallimento):** messaggio “nessun dato” o errore I/O.  
**Trigger:** scelta dal “Menù generale” configuratore.

### Flusso principale
1. L’attore seleziona una delle voci di consultazione:
   - elenco volontari con visite associate
   - elenco luoghi visitabili
   - elenco tipi di visita
2. Il sistema recupera i dati dal gestore.
3. Il sistema stampa l’elenco in console (formattazione testuale).

### Flussi alternativi / eccezioni
- **A1 — Liste vuote**
  1. Condizione: non esistono elementi da mostrare.
  2. Passi: il sistema informa che non ci sono dati disponibili.
  3. Esito: ritorno al menu.

### Regole di business / vincoli
- RB1: “volontario attivo” dipende dalle visite attive associate; la consultazione potrebbe includere anche volontari non attivi (dipende dal filtro usato nel metodo di stampa).
  > Nota: senza requisiti non è chiaro se l’elenco debba includere anche inattivi; va verificato sul metodo specifico di stampa nel menu.

### Dati coinvolti
- Input: selezione voce.
- Output: stampa su console.
- Entità/modelli: `Luogo`, `Visita`, `Utente`.

### Tracciabilità
- **Codice:** `MenuConfiguratore.mostraElencoVolontariAssociatiVisita()`, `MenuConfiguratore.mostraElencoLuoghi()`, `MenuConfiguratore.mostraElencoTipiVisita()`
- **Requisiti/slide:** > Nota: non forniti in questa chat.

---

## UC10 — Chiudere raccolta disponibilità del mese corrente di raccolta

**Attori primari:** Configuratore  
**Attori secondari:** Orologio di sistema, Sistema di persistenza su file  
**Scopo:** impedire ulteriori inserimenti di disponibilità e preparare la produzione del piano.  
**Precondizioni:** autenticazione configuratore; raccolta aperta; piano non prodotto; oggi dentro finestra.  
**Postcondizioni (successo):** `raccoltaAperta=false` persistito nello stato sistema.  
**Postcondizioni (fallimento):** raccolta invariata.  
**Trigger:** scelta “Chiudi raccolta disponibilità (mese)” dal menu attività.

### Flusso principale
1. L’attore seleziona “Chiudi raccolta disponibilità”.
2. Il sistema carica lo stato del sistema.
3. Il sistema verifica che **oggi** sia nella finestra ammessa per quel mese di raccolta.
4. Il sistema verifica che la raccolta sia aperta e che il piano non sia già prodotto.
5. Il sistema chiude la raccolta e salva lo stato.

### Flussi alternativi / eccezioni
- **A1 — Fuori finestra temporale**
  1. Condizione: oggi non appartiene alla finestra `target-2/16` → `target-1/15`.
  2. Passi: il sistema rifiuta l’operazione con errore.
  3. Esito: nessuna modifica.
- **A2 — Raccolta già chiusa / piano già prodotto**
  1. Condizione: `raccoltaAperta=false` oppure `pianoProdotto=true`.
  2. Passi: il sistema rifiuta l’operazione.
  3. Esito: nessuna modifica.

### Regole di business / vincoli
- RB1: la finestra di gestione è definita rispetto a `meseRaccolta`: dal giorno 16 di due mesi prima al giorno 15 del mese precedente.

### Dati coinvolti
- Input: comando di chiusura.
- Output: stato sistema aggiornato.
- Entità/modelli: `StatoSistema`.

### Tracciabilità
- **Codice:** `MenuConfiguratore.chiudiRaccoltaDisponibilita()`, `GestoreDati.chiudiRaccoltaDisponibilita()`, `StatoSistema.chiudiRaccolta()`, `FileIO.salvaStatoSistema(...)`
- **Requisiti/slide:** > Nota: non forniti in questa chat.

---

## UC11 — Produrre piano visite del mese di raccolta

**Attori primari:** Configuratore  
**Attori secondari:** Orologio di sistema, Sistema di persistenza su file  
**Scopo:** generare le istanze pianificate (`VisitaIstanza`) assegnando volontari disponibili alle visite programmabili giorno per giorno.  
**Precondizioni:** raccolta chiusa; piano non prodotto; oggi dentro finestra.  
**Postcondizioni (successo):** lista `VisitaIstanza` salvata; `pianoProdotto=true`; dati “dimenticati” fino a un certo mese.  
**Postcondizioni (fallimento):** piano non aggiornato (oppure letto da file se già prodotto).  
**Trigger:** scelta “Produci piano visite (mese)” dal menu attività.

### Flusso principale
1. L’attore seleziona “Produci piano visite”.
2. Il sistema carica lo stato sistema.
3. Il sistema verifica:
   - oggi dentro finestra temporale del mese di raccolta
   - raccolta chiusa
   - piano non ancora prodotto
4. Il sistema legge:
   - disponibilità volontari
   - date precluse
   - luoghi e tipi visita
5. Per ogni giorno del mese di raccolta:
   1. il sistema salta il giorno se è precluso
   2. per ogni luogo e per ogni tipo visita del luogo:
      - se il tipo visita è programmabile in quel giorno
      - il sistema sceglie un volontario associato che:
        - è disponibile in quel giorno
        - non è già occupato in quel giorno (vincolo “1 visita/giorno per volontario”)
      - se trova un volontario, crea una `VisitaIstanza` in stato `proposta`
6. Il sistema salva l’elenco istanze su file.
7. Il sistema marca il piano come prodotto e salva lo stato sistema.
8. Il sistema conferma quante visite “proposte” sono state create.

### Flussi alternativi / eccezioni
- **A1 — Raccolta ancora aperta**
  1. Condizione: `raccoltaAperta=true`.
  2. Passi: il sistema rifiuta la produzione e richiede prima la chiusura.
  3. Esito: nessun piano nuovo.
- **A2 — Piano già prodotto**
  1. Condizione: `pianoProdotto=true`.
  2. Passi: il sistema legge e restituisce il piano già salvato (senza rigenerarlo).
  3. Esito: consultazione implicita del piano esistente.
- **A3 — Nessun volontario disponibile per un tipo visita in un giorno**
  1. Condizione: non esiste volontario associato che soddisfi disponibilità e non-occupazione.
  2. Passi: il sistema non crea l’istanza per quel tipo visita in quel giorno.
  3. Esito: piano parziale (non tutte le visite programmabili generano istanze).
- **A4 — Errore I/O**
  1. Condizione: errore in lettura/scrittura file.
  2. Passi: il sistema segnala errore.
  3. Esito: piano non garantito.

### Regole di business / vincoli
- RB1: una visita istanza nasce in stato `proposta`.
- RB2: un volontario non può essere assegnato a più visite nello stesso giorno (vincolo implementato tramite “occupatoVolontarioGiorno”).
- RB3: l’algoritmo è **greedy e dipende dall’ordine** (giorni → luoghi → tipi visita → volontari). Questo può introdurre bias di assegnazione (i primi volontari in lista vengono scelti più spesso).

### Dati coinvolti
- Input: comando di produzione; disponibilità; date precluse; tipi visita.
- Output: elenco `VisitaIstanza` persistito; stato sistema aggiornato.
- Entità/modelli: `StatoSistema`, `Visita`, `Luogo`, `VisitaIstanza`, `StatiVisita`.

### Tracciabilità
- **Codice:** `MenuConfiguratore.produciPianoVisite()`, `GestoreDati.produciPianoVisite()`, `GestoreDati.scegliVolontarioDisponibile(...)`, `FileIO.salvaVisiteIstanze(...)`, `FileIO.salvaStatoSistema(...)`, `Visita.isProgrammabile(...)`
- **Requisiti/slide:** > Nota: non forniti in questa chat.

---

## UC12 — Gestire richieste di aggiunta/rimozione (dopo piano prodotto)

**Attori primari:** Configuratore  
**Attori secondari:** Sistema di persistenza su file  
**Scopo:** applicare modifiche “strutturali” (aggiunte/rimozioni) quando il piano del mese è già stato prodotto.  
**Precondizioni:** `pianoProdotto=true` nello stato sistema.  
**Postcondizioni (successo):** dati modificati e persistiti; eventuali rimozioni applicate “con cascata”.  
**Postcondizioni (fallimento):** dati non aggiornati.  
**Trigger:** scelta “Gestisci richieste di aggiunta/rimozione dati” dal menu attività.

### Flusso principale
1. L’attore seleziona la gestione modifiche.
2. Il sistema carica lo stato sistema e verifica che il piano sia già prodotto.
3. Il sistema mostra il menu modifiche:
   - aggiungi luogo
   - aggiungi tipo visita
   - rimuovi luogo (con cascata)
   - rimuovi tipo visita (con cascata)
   - rimuovi volontario (con cascata)
4. L’attore seleziona un’operazione.
5. Il sistema applica l’operazione scelta e salva i dati.

### Flussi alternativi / eccezioni
- **A1 — Piano non prodotto**
  1. Condizione: `pianoProdotto=false`.
  2. Passi: il sistema blocca l’accesso al menu modifiche e informa l’attore.
  3. Esito: nessuna modifica.
- **A2 — Cascata su dati correlati**
  1. Condizione: rimozione di luogo/tipo/volontario.
  2. Passi: il sistema rimuove anche i riferimenti correlati (es. visite del luogo, volontari associati, ecc.).
  3. Esito: chiusura “transitiva” e persistenza aggiornata.
- **A3 — Liste vuote**
  1. Condizione: non esistono elementi rimovibili (es. nessun luogo attivo).
  2. Passi: il sistema informa e interrompe l’operazione di rimozione.
  3. Esito: nessuna modifica.

### Regole di business / vincoli
- RB1: l’accesso alle modifiche è subordinato al fatto che il piano sia stato prodotto (vincolo esplicito nel menu).
- RB2: le rimozioni avvengono “con cascata” (chiusura transitiva) per evitare riferimenti orfani.

### Dati coinvolti
- Input: selezione operazione; selezione elemento da rimuovere; dati per nuove entità.
- Output: dati aggiornati.
- Entità/modelli: `Luogo`, `Visita`, `Utente`, `GestoreCalendario`.

### Tracciabilità
- **Codice:** `MenuConfiguratore.menuModifiche()`, `MenuConfiguratore.richiestaRimozioneLuogo()`, `MenuConfiguratore.richiestaRimozioneTipoVisita()`, `MenuConfiguratore.richiestaRimozioneVolontario()`, `GestoreDati.rimuoviLuogoConCascata(...)`, `GestoreDati.rimuoviTipoVisitaConCascata(...)`, `GestoreDati.rimuoviVolontarioConCascata(...)`
- **Requisiti/slide:** > Nota: non forniti in questa chat.

---

## UC13 — Inserire e consultare date precluse

**Attori primari:** Configuratore  
**Attori secondari:** Sistema di persistenza su file, Orologio di sistema  
**Scopo:** gestire le date in cui non è possibile pianificare visite.  
**Precondizioni:** autenticazione configuratore.  
**Postcondizioni (successo):** data preclusa aggiunta e persistita; oppure elenco stampato.  
**Postcondizioni (fallimento):** nessuna modifica.  
**Trigger:** scelta “Inserimento date precluse” nel menu attività.

### Flusso principale
1. L’attore seleziona “Inserimento date precluse”.
2. Il sistema calcola il mese target per le precluse (logica basata su “oggi”).
3. Il sistema mostra un menu:
   - aggiunta data preclusa
   - visualizza date precluse
4. Se l’attore sceglie “aggiunta”:
   1. il sistema richiede il giorno del mese (range 1..lengthOfMonth)
   2. l’attore inserisce il giorno
   3. il sistema costruisce la data completa (YYYY-MM-targetMonth)
   4. il sistema salva la data tra le precluse e persiste
5. Se l’attore sceglie “visualizza”:
   1. il sistema filtra e ordina le precluse del mese target
   2. il sistema stampa l’elenco

### Flussi alternativi / eccezioni
- **A1 — Nessuna data preclusa nel mese**
  1. Condizione: filtro vuoto.
  2. Passi: il sistema informa che non ci sono date precluse per quel mese.
  3. Esito: ritorno al menu date precluse.
- **A2 — Errore I/O**
  1. Condizione: errori in lettura/scrittura delle precluse.
  2. Passi: il sistema segnala errore.
  3. Esito: nessuna modifica garantita.

### Regole di business / vincoli
- RB1: il mese target per le precluse è calcolato come “mese base + 3”, dove il mese base dipende dal giorno corrente (≥16 oppure no).
  > Nota: questa regola non è esplicitamente collegata a `StatoSistema.meseRaccolta`; va verificato se è una scelta voluta o un disallineamento.

### Dati coinvolti
- Input: giorno del mese; comando di visualizzazione.
- Output: set di `LocalDate` precluse aggiornato / stampa.
- Entità/modelli: insieme `datePrecluse`.

### Tracciabilità
- **Codice:** `MenuConfiguratore.inserimentoDatePrecluse()`, `GestoreDati.getDatePrecluse()/setDatePrecluse(...)`, `FileIO.leggiDatePrecluse()`, `FileIO.salvaDatePrecluse(...)`
- **Requisiti/slide:** > Nota: non forniti in questa chat.

---

## UC14 — Consultare piano visite prodotto

**Attori primari:** Configuratore  
**Attori secondari:** Sistema di persistenza su file  
**Scopo:** visualizzare il piano visite (istanze) generato per il mese di raccolta.  
**Precondizioni:** esiste un piano salvato (o lista istanze leggibile).  
**Postcondizioni (successo):** piano stampato in console.  
**Postcondizioni (fallimento):** messaggio di assenza dati o errore I/O.  
**Trigger:** voce “Visualizzare piano visite prodotto” nel menu generale.

### Flusso principale
1. L’attore seleziona “Visualizzare piano visite prodotto”.
2. Il sistema legge le `VisitaIstanza` dal file del piano.
3. Il sistema stampa l’elenco.

### Flussi alternativi / eccezioni
- **A1 — Piano assente/vuoto**
  1. Condizione: lista nulla o vuota.
  2. Passi: il sistema informa che non ci sono istanze.
  3. Esito: ritorno al menu.
- **A2 — Errore I/O**
  1. Condizione: eccezione in lettura file.
  2. Passi: il sistema segnala errore.
  3. Esito: consultazione fallita.

### Regole di business / vincoli
- RB1: le istanze hanno uno stato (`proposta`, `completa`, `confermata`, `cancellata`, `effettuata`) ma nel codice fornito non emergono flussi per farle avanzare di stato.
  > Nota: per UC di avanzamento stati servirebbe un menu dedicato non presente nei file caricati.

### Dati coinvolti
- Input: comando di consultazione.
- Output: stampa del piano.
- Entità/modelli: `VisitaIstanza`, `StatiVisita`.

### Tracciabilità
- **Codice:** `MenuConfiguratore.visualizzaPianoProdotto()`, `GestoreDati.leggiPianoVisite()`, `FileIO.leggiVisiteIstanze()`
- **Requisiti/slide:** > Nota: non forniti in questa chat.

---

## UC15 — Inserire disponibilità come volontario

**Attori primari:** Volontario  
**Attori secondari:** Sistema di persistenza su file, Orologio di sistema  
**Scopo:** inserire date di disponibilità nel mese di raccolta corrente, rispettando vincoli di preclusione e compatibilità coi tipi visita associati.  
**Precondizioni:** autenticazione volontario; raccolta aperta; il volontario ha almeno un tipo visita programmabile nel mese.  
**Postcondizioni (successo):** una o più date aggiunte alle disponibilità del volontario e persistite.  
**Postcondizioni (fallimento):** nessuna nuova disponibilità aggiunta.  
**Trigger:** scelta “Esprimi disponibilità (mese)” nel menu volontario.

### Flusso principale
1. Il volontario apre il menu volontario.
2. Il sistema carica lo stato del sistema e identifica il mese di raccolta.
3. Il volontario seleziona “Esprimi disponibilità”.
4. Il sistema verifica che la raccolta sia aperta.
5. Il sistema verifica che il volontario abbia almeno un tipo visita programmabile nel mese.
6. Il sistema mostra un sottomenu:
   - scegli una data di disponibilità
   - stampa disponibilità del mese
   - stampa date precluse del mese
7. Se l’attore sceglie “scegli data”:
   1. il sistema costruisce l’elenco delle date selezionabili (non precluse, non già scelte, compatibili coi tipi associati)
   2. l’attore seleziona una data
   3. il sistema salva la data nelle disponibilità del volontario su file
8. L’attore può ripetere la selezione per aggiungere altre date.

### Flussi alternativi / eccezioni
- **A1 — Raccolta chiusa**
  1. Condizione: `raccoltaAperta=false`.
  2. Passi: il sistema informa “Raccolta disponibilità chiusa”.
  3. Esito: nessuna modifica.
- **A2 — Nessun tipo visita associato programmabile nel mese**
  1. Condizione: il volontario non risulta associato a tipi visita programmabili per quel mese.
  2. Passi: il sistema informa e interrompe.
  3. Esito: nessuna modifica.
- **A3 — Nessuna data selezionabile**
  1. Condizione: tutte le date del mese sono precluse, già selezionate o incompatibili.
  2. Passi: il sistema informa che non ci sono date selezionabili.
  3. Esito: ritorno al sottomenu senza salvataggi.
- **A4 — Errore I/O**
  1. Condizione: eccezione nel salvataggio disponibilità.
  2. Passi: il sistema segnala errore.
  3. Esito: disponibilità non garantite.

### Regole di business / vincoli
- RB1: una data non è selezionabile se:
  - è preclusa
  - è già presente tra le disponibilità del volontario
  - non è compatibile con alcun tipo visita associato (vincolo “compatibilità con scheduling”)
- RB2: le disponibilità sono riferite al mese di raccolta indicato da `StatoSistema`.

### Dati coinvolti
- Input: selezione data; comandi di stampa.
- Output: set di date disponibili aggiornato e persistito; stampa liste.
- Entità/modelli: disponibilità volontario (mappa `nomeVolontario -> Set<LocalDate>`), `StatoSistema`, `Visita`.

### Tracciabilità
- **Codice:** `MenuVolontario.menu()`, `MenuVolontario.inserisciDisponibilita(...)`, `MenuVolontario.menuSceltaDisponibilitaMese(...)`, `GestoreDati.aggiungiDisponibilitaVolontario(...)`, `FileIO` (lettura/scrittura disponibilità), `Visita.isProgrammabile(...)`
- **Requisiti/slide:** > Nota: non forniti in questa chat.

---

## UC16 — Consultare tipi visita associati come volontario

**Attori primari:** Volontario  
**Scopo:** visualizzare i tipi di visita a cui il volontario è abilitato (associato).  
**Precondizioni:** autenticazione volontario.  
**Postcondizioni (successo):** elenco stampato in console.  
**Postcondizioni (fallimento):** messaggio “nessuna visita associata”.  
**Trigger:** voce “Visualizza i tipi di visita associati” nel menu volontario.

### Flusso principale
1. Il volontario seleziona “Visualizza i tipi di visita associati”.
2. Il sistema ricerca i tipi visita che contengono il nome del volontario nella lista volontari.
3. Il sistema stampa l’elenco in console.

### Flussi alternativi / eccezioni
- **A1 — Nessun tipo visita associato**
  1. Condizione: elenco vuoto.
  2. Passi: il sistema informa l’utente.
  3. Esito: ritorno al menu.

### Regole di business / vincoli
- RB1: l’associazione è per nome utente (stringa) dentro `Visita.volontari`.

### Dati coinvolti
- Input: comando di consultazione.
- Output: stampa elenco.
- Entità/modelli: `Visita`, `Luogo`.

### Tracciabilità
- **Codice:** `MenuVolontario.visualizzaTipiVisitaAssociati()`, `MenuVolontario.getTipiVisitaAssociati(...)`
- **Requisiti/slide:** > Nota: non forniti in questa chat.

---

# Scostamenti e ambiguità

1. **Ruolo “fruitore” presente ma non operativo**
   - Nel modello `Utente` esiste il ruolo `"fruitore"` ed è sempre “attivo”, ma `Main.avviaMenu()` gestisce solo `configuratore` e `volontario`.
   - Conseguenza: requisiti che prevedono casi d’uso fruitore (iscrizioni, consultazioni, ecc.) non sono verificabili “da codice” nei file caricati.

2. **Voce di menu “Riapri raccolta disponibilità (mese successivo)” vs implementazione**
   - Nel menu attività del configuratore la label suggerisce il mese successivo, ma `GestoreDati.riapriRaccoltaMeseSuccessivo()` **non cambia `meseRaccolta`**, imposta solo `raccoltaAperta=true`.
   - Il cambio mese sembra demandato alla sincronizzazione automatica (`sincronizzaStatoConCalendario()`), che dipende dalla data corrente.
   - Impatto: possibile disallineamento tra aspettativa utente e comportamento effettivo.

3. **Gestione date precluse scollegata dallo stato di raccolta**
   - `inserimentoDatePrecluse()` calcola un mese target con una regola propria (“mese base + 3”), non usando `StatoSistema.meseRaccolta`.
   - Se il requisito voleva “precluse per il mese di piano/raccolta”, qui c’è un potenziale mismatch.

4. **Ciclo di vita delle `VisitaIstanza` (stati) non gestito da menu**
   - Esistono gli stati `proposta/completa/confermata/cancellata/effettuata`, ma nei menu caricati non c’è un flusso per cambiare stato (es. confermare, cancellare, marcare effettuata).
   - > Nota: se esiste un menu “fruitore” o “volontario avanzato” in altri file non caricati, servirebbe per confermare l’implementazione.

5. **Algoritmo di piano “greedy” e potenziale bias**
   - La scelta del volontario disponibile avviene scorrendo la lista in ordine: può penalizzare volontari in fondo alla lista e privilegiare i primi.