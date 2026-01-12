# 6. Utilizzo da parte del Fruitore

## 6.1 Registrazione e login

### 6.1.1 Registrazione
Dal menù iniziale selezionare **Fruitore** → “Registrati”.
- inserire username;
- se già esistente, appare un sotto‑menù per reinserire o tornare indietro;
- inserire password;
- viene creato un utente con ruolo `fruitore` e `primoAccesso=false`.

### 6.1.2 Login
“Accedi” richiede username e password. L’accesso è permesso solo se il ruolo dell’utente trovato è effettivamente `fruitore`.

## 6.2 Menù del fruitore
Voci principali:
1. **Visualizza visite (proposte / confermate / cancellate)**
2. **Iscriviti a una visita proposta**
3. **Le mie visite (proposte / complete / confermate / cancellate)**
4. **Disdici iscrizione (prima della chiusura iscrizioni)**

## 6.3 Consultazione delle visite
- Se si sceglie “Visualizza visite”, il sistema mostra le visite nel piano filtrate per stato:
  - `proposta`, `confermata`, `cancellata`.
- In “Le mie visite”, oltre a quelle sopra può comparire anche `completa`, ma solo se il fruitore risulta iscritto.

Per ogni visita viene stampato un dettaglio (data, tipo visita, luogo, ecc.) tramite `toStringDettagliato`, con alcune varianti se si passa o meno lo username fruitore.

## 6.4 Iscrizione a una visita (anche per più partecipanti)

### 6.4.1 Scelta della visita
“Iscriviti a una visita proposta” mostra la lista delle **visite iscrivibili**, che sono:
- solo in stato `proposta`;
- con data di visita tale che **oggi è prima** della data di chiusura iscrizioni (chiusura = dataVisita − 3 giorni);
- con posti ancora disponibili rispetto al max del tipo visita.

La lista è ordinata per data.

### 6.4.2 Numero partecipanti e vincoli
Dopo aver scelto la visita, il fruitore inserisce il numero di persone:
- minimo 1;
- massimo = `maxPersoneIscrivibiliDaFruitore` dai dati generali (`generale.txt`).

Poi `GestoreDati.iscriviFruitoreAVisita` applica ulteriori vincoli:
- iscrizione consentita **solo** su visite `proposta`;
- lo stesso fruitore non può iscriversi due volte alla stessa visita;
- capienza residua sufficiente (posti rimasti >= numPersone).

Se dopo l’iscrizione la capienza diventa piena, lo stato dell’istanza passa a `completa`.

### 6.4.3 Codice di prenotazione
Se l’iscrizione è accettata, viene generato un codice alfanumerico univoco (uppercase) e mostrato a schermo:
```
Iscrizione accettata! Codice prenotazione: X7Q2B9
```
Il codice viene salvato con l’iscrizione nella visita istanza.

## 6.5 Disdetta di una prenotazione

### 6.5.1 Quando è consentita
La disdetta è consentita **solo** se:
- la visita è ancora in stato `proposta` o `completa`;
- oggi è **prima** della data di chiusura iscrizioni (dataVisita − 3 giorni);
- il codice appartiene a un’iscrizione del fruitore corrente.

Se la disdetta avviene su una visita `completa` e libera posti, lo stato torna a `proposta`.

### 6.5.2 Flusso utente
- scegliere “Disdici iscrizione”
- inserire codice
- il sistema risponde con successo o fallimento (codice non trovato / non valido / disdetta non consentita).

## 6.6 Stati delle visite e significato (in applicazione)

- **proposta**: istanza generata nel piano, iscrizioni aperte fino a 3 giorni prima.
- **completa**: capienza raggiunta; iscrizioni ulteriori rifiutate; disdetta ancora possibile fino alla chiusura.
- **confermata**: assegnata automaticamente 3 giorni prima se iscritti >= minimo del tipo visita.
- **cancellata**: assegnata automaticamente 3 giorni prima se iscritti < minimo; viene rimossa dal piano dopo la data visita.
- **effettuata**: assegnata dopo la data visita per visite confermate; viene spostata in archivio storico e rimossa dal piano corrente.

> Nota: gli stati vengono aggiornati quando l’applicazione esegue operazioni che richiamano l’aggiornamento (non esiste un job schedulato).
