# 4. Utilizzo da parte del Configuratore

## 4.1 Accesso
Dal menù iniziale selezionare **Configuratore** → inserire credenziali.

Se i dati generali non sono presenti (`generale.txt` vuoto/assente), al primo accesso il sistema chiede:
- *Ambito territoriale*
- *Numero max persone iscrivibili da un fruitore*

Senza questi dati, gli utenti non configuratori vengono bloccati (in pratica il sistema richiede prima la configurazione globale).

## 4.2 Struttura del menù del Configuratore
Il menù principale del configuratore include (ordine conforme al codice):

1. **Setup iniziale (crea luogo/visita)**
2. **Attività a regime (piano, modifiche, riapertura, ecc.)**
3. **Modifica numero massimo iscrivibili da fruitore**
4. **Mostra elenco volontari e visite associate**
5. **Inserimento / visualizzazione date precluse**
6. **Visualizza luoghi e visite**
7. **Visualizza stato sistema e piano corrente**
8. **Visualizza archivio storico** (tipi visita scaduti / istanze effettuate, se previsto nel menù)

> Alcune voci possono variare leggermente se il menù viene composto in base allo stato (es. “setup iniziale” se mancano dati). La logica di base resta quella sopra.

## 4.3 Setup iniziale: creazione luogo e prima visita (obbligatoria)
Flusso: **Setup iniziale** → “Crea luogo con prima visita”

1. Inserimento luogo:
   - Nome luogo (non vuoto)
   - Descrizione (non vuota)
   - Latitudine `[-90, 90]`
   - Longitudine `[-180, 180]`
   - L’ID del luogo è derivato dai dati; se esiste già un luogo con stesso ID (nome+coordinate), viene richiesto di reinserire.

2. Inserimento visita (obbligatoria, altrimenti il luogo viene annullato e rimosso):
   - Titolo (unico **nel luogo**; se duplicato, reinserimento)
   - Descrizione
   - Punto di incontro (via/piazza, civico opzionale, descrizione opzionale)
   - Min partecipanti
   - Max partecipanti (vincolo: `max >= min`)
   - Flag “biglietto acquistabile?” (Sì/No)
   - Selezione volontari (almeno 1):
     - scegliere tra volontari esistenti;
     - oppure “Aggiungi nuovo volontario” (creazione credenziali).
     - se un volontario selezionato è inattivo, viene riattivato e salvato.
   - Scheduling (periodo e giorni programmabili):
     - Data inizio (>= oggi)
     - Data fine (>= data inizio)
     - Durata in minuti
     - Ora inizio
     - Giorni della settimana programmabili (almeno 1, calcolati sul periodo)

3. Controllo overlap (vincolo temporale nello stesso luogo)
L’inserimento del tipo visita passa da `GestoreCalendario.aggiungiVisita(visita)`.
Se rileva sovrapposizioni temporali con altri tipi visita **dello stesso luogo**, l’operazione fallisce e viene mostrato un menù:
- modificare scheduling (date/ora/durata/giorni) e riprovare
- oppure annullare (in setup iniziale: annulla anche il luogo appena creato).

### Interpretazione implementativa del vincolo overlap
Il controllo non è “solo sul giorno”, ma valuta l’intersezione di giorni del periodo comune e, per ciascun giorno, la sovrapposizione degli intervalli orari.

## 4.4 Creazione visita associandola a un luogo esistente
Flusso: **Setup iniziale** → “Crea visita con associazione obbligatoria”

- Opzione A: associa a luogo esistente → scelta luogo → crea visita (come sopra) → controllo overlap → salvataggio.
- Opzione B: crea nuovo luogo e associa la visita → equivalente a §4.3.

Nota: se scegli un luogo inattivo, il sistema lo riattiva e salva.

## 4.5 Gestione volontari e associazioni
Il configuratore può:
- creare nuovi volontari (username+password predefinita);
- associare volontari ai tipi visita in fase di creazione (obbligatorio almeno 1);
- visualizzare mappa volontario → titoli visite associate (solo per tipi visita attivi).

### Stato “attivo” dei volontari
Nel codice, un volontario può risultare inattivo se non è associato ad alcun tipo visita attivo.
- un volontario inattivo **non** riesce ad autenticarsi;
- viene riattivato automaticamente se il configuratore lo seleziona in un’associazione.

## 4.6 Date precluse (globali)
Voce: **Inserimento / visualizzazione date precluse**.

La logica implementata sceglie un mese “target” per le date precluse:
- calcola un “mese base” in funzione del giorno corrente:
  - se oggi è dal 16 in poi → base = mese corrente
  - altrimenti → base = mese precedente
- mese precluse = base + 3 mesi

Operazioni:
- aggiungere una data del mese target (scegliendo un giorno 1..fine mese); se già presente → messaggio;
- visualizzare le date precluse del mese target (filtrate e ordinate).

> Effetto: le date precluse vengono ignorate dal generatore del piano (nessuna istanza viene creata su date precluse).

## 4.7 Attività a regime: stato sistema, raccolta disponibilità e piano

### 4.7.1 StatoSistema: mese target e finestra temporale
Lo stato del sistema include:
- `meseRaccolta` (YearMonth target)
- `raccoltaAperta` (boolean)
- `pianoProdotto` (boolean)

La finestra temporale **in cui è consentito operare** (chiusura raccolta / produzione piano / riapertura) è:
- dal **16** del mese `(meseRaccolta - 2)`
- al **15** del mese `(meseRaccolta - 1)`

In più, il codice sincronizza automaticamente:
- al giorno 15: se raccolta aperta → viene chiusa;
- al giorno ≥ 16: se raccolta chiusa e piano non prodotto → viene riaperta.

### 4.7.2 Chiudere la raccolta disponibilità
Voce: “Chiudi raccolta disponibilità”.
- consentita solo dentro la finestra;
- fallisce se già chiusa o se il piano è già prodotto.

### 4.7.3 Produrre il piano delle visite
Voce: “Produci piano visite”.
Prerequisiti implementativi:
- raccolta deve essere chiusa;
- piano non deve essere già prodotto;
- data corrente deve essere dentro la finestra.

Algoritmo effettivo (greedy):
1. legge disponibilità dei volontari e date precluse;
2. per ogni giorno del mese `meseRaccolta`, e per ogni luogo, e per ogni tipo visita associato al luogo:
   - se il tipo visita è programmabile quel giorno (periodo + giorno settimana);
   - cerca **il primo volontario** associato al tipo che sia disponibile quel giorno e non già assegnato a un’altra visita nello stesso giorno;
   - crea un’istanza (`VisitaIstanza`) con stato iniziale `proposta`.
3. salva `visiteIstanze.json`;
4. marca `pianoProdotto=true` e salva lo stato;
5. ripulisce disponibilità e date precluse fino al mese incluso (dati “vecchi” non più necessari).

> Conseguenza: un volontario può essere assegnato **al massimo a una visita per giorno**, anche se in teoria gli orari non si sovrappongono (scelta semplificativa dell’implementazione).

### 4.7.4 Riaprire la raccolta per il mese successivo
Voce: “Riapri raccolta mese successivo”.
Condizione implementata:
- **richiede** che il piano del mese corrente sia già prodotto.

Effetto:
- aggiorna `meseRaccolta` al mese successivo e riapre la raccolta (se in finestra).

## 4.8 Gestione aggiunte e rimozioni con effetti collaterali

### 4.8.1 Quando è possibile modificare
Nel menù “Attività” è presente una voce dedicata alle richieste di modifica.
Nel codice, la modifica è considerata parte delle attività da eseguire quando il piano è stato prodotto (coerenza con “a valere dal mese successivo”).

### 4.8.2 Rimozioni a cascata (chiusura transitiva)
Le rimozioni sono implementate con metodi “ConCascata” che:
1. eliminano direttamente l’elemento richiesto (luogo / tipo visita / volontario);
2. applicano una **chiusura transitiva** (`chiusuraTransitiva`) che ripete finché ci sono cambiamenti:
   - rimuove tipi visita rimasti **senza volontari**;
   - rimuove luoghi rimasti **senza tipi visita**;
   - rimuove volontari rimasti **non associati** ad alcun tipo visita attivo (e li elimina anche da disponibilità persistite).

Poi salva il corpo dati e riallinea attivi/archivi.

#### Effetti pratici (da sapere prima di confermare una rimozione)
- Eliminare un **luogo** può portare a rimuovere i suoi tipi visita e, indirettamente, volontari che erano associati solo a quelle visite.
- Eliminare un **tipo visita** può rendere inattivo e poi eliminabile un luogo (se resta senza visite) e può eliminare volontari “orfani”.
- Eliminare un **volontario** può portare alla rimozione dei tipi visita che restano senza guide.

