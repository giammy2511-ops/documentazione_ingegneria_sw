# 5. Utilizzo da parte del Volontario

## 5.1 Accesso
Dal menù iniziale selezionare **Volontario** → inserire credenziali.

Vincolo implementativo importante:
- il volontario deve risultare **attivo**, altrimenti l’autenticazione fallisce anche con password corretta.
- l’attività dipende dall’associazione ad almeno un tipo visita attivo (aggiornata automaticamente da `GestoreDati`).

## 5.2 Primo accesso e cambio password
Se l’account è stato creato dal configuratore, viene salvato con `primoAccesso=true`.
Al primo login, il sistema impone il cambio password (non è opzionale).

## 5.3 Menù del volontario
Voci principali:
1. **Visualizza tipi visita associati**
2. **Inserisci disponibilità per un mese**
3. **Visualizza piano visite del mese (come volontario)**

## 5.4 Consultazione dei tipi di visita associati
L’applicazione scorre i luoghi e i tipi visita e mostra quelli in cui il nickname del volontario compare nella lista volontari del tipo visita.

## 5.5 Inserimento disponibilità mensili

### 5.5.1 Scelta del mese
Il volontario sceglie fra:
- mese “target” corrente del sistema;
- (eventualmente) mese successivo (dipende dalle opzioni mostrate dal menù di scelta mese).

Nel codice il mese è costruito a partire da `StatoSistema.meseRaccolta` e dal calendario corrente.

### 5.5.2 Vincoli sulle date selezionabili
Una data può essere aggiunta come disponibilità solo se:
- appartiene al mese selezionato;
- **non** è una data preclusa globale (`datePrecluse.txt`);
- è “compatibile” con almeno un tipo visita associato al volontario (controllo basato su programmabilità: periodo + giorno della settimana);
- non è già presente tra le disponibilità registrate.

Le disponibilità sono salvate su file in `disponibilitaVolontari.txt` e rilette ad ogni avvio/operazione.

### 5.5.3 Esempio di interazione (semplificato)
```
Menù volontario
1) Visualizza tipi visita associati
2) Inserisci disponibilità per un mese
...

> 2
Scegli mese: 1) FEBBRAIO 2026  2) MARZO 2026
> 1
Inserire giorno (1..28): > 12
Disponibilità aggiunta: 2026-02-12
```

## 5.6 Visualizzazione visite e codici di prenotazione
Il volontario ha una voce “visualizza piano”. Il metodo invocato stampa un riepilogo della visita istanza usando `toStringPerVolontario`.

⚠️ **Limite/bug rilevato nel codice**  
La funzione è denominata come se mostrasse “visite confermate”, ma il filtro sugli stati include solo `proposta` e `completa` (e ripete `proposta` due volte), escludendo `confermata`. Risultato plausibile:
- quando una visita passa a `confermata` (3 giorni prima), potrebbe **sparire** dalla vista volontario.

Se la consegna richiede la vista delle confermate con codici, questa parte va considerata **non pienamente implementata** oppure affetta da un errore nel filtro stati.
