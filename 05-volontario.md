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

### 5.5.1 Vincoli sulle date selezi
Una data può essere aggiunta come disponibilità solo se:
- appartiene al mese selezionato;
- **non** è una data preclusa globale (`datePrecluse.txt`);
- è “compatibile” con almeno un tipo visita associato al volontario (controllo basato su programmabilità: periodo + giorno della settimana);
- non è già presente tra le disponibilità registrate.

Le disponibilità sono salvate su file in `disponibilitaVolontari.txt` e rilette ad ogni avvio/operazione.

### 5.5.3 Esempio di interazione (semplificato)
```
Disponibilità FEBBRAIO 2026
--------------------------------
1	Aggiungi una data di disponibilità
2	Visualizza disponibilità inserite
3	Visualizza date precluse del mese

0	Esci

Digita il numero dell'opzione desiderata > 1
--------------------------------
Scegli una data di disponibilità (0 = termina)
--------------------------------
1	2026-02-02
2	2026-02-04
3	2026-02-07
4	2026-02-09
5	2026-02-11
6	2026-02-13
7	2026-02-14
8	2026-02-15
9	2026-02-16
10	2026-02-17
11	2026-02-18
12	2026-02-20
13	2026-02-21
14	2026-02-22
15	2026-02-23
16	2026-02-24
17	2026-02-25
18	2026-02-27
19	2026-02-28

0	Esci

Digita il numero dell'opzione desiderata > 4
Disponibilità aggiunta per: 2026-02-09
```

## 5.6 Visualizzazione visite e codici di prenotazione
Il volontario ha una voce “visualizza piano”. Il metodo invocato stampa un riepilogo della visita istanza usando `toStringPerVolontario`.
