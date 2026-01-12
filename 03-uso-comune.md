# 3. Manuale di utilizzo – struttura comune

## 3.1 Modalità di avvio
Eseguire `Main`. All’avvio vengono caricati i file da `./DATA` e si apre il menù:

- **Configuratore**
- **Volontario**
- **Fruitore**

Uscire scegliendo `0` (voce di ritorno/uscita standard dei menù).

## 3.2 Login e gestione credenziali

### Configuratore / Volontario
Login con:
- username
- password

La verifica è eseguita da `GestoreDati.autentica(username, password)` e fallisce se:
- username/password non corrispondono;
- l’utente (solo se volontario) risulta **non attivo**.

**Primo accesso del volontario**: se `utente.isPrimoAccesso()` è `true`, il sistema forza il cambio password durante il login.

> Nota implementativa: il cambio credenziali dei volontari permette solo il cambio password; per gli altri ruoli può essere previsto anche il cambio username (dipende dal ramo eseguito nel menu di cambio credenziali).

### Fruitore
Il fruitore usa un sotto‑menù dedicato:
- **Accedi**
- **Registrati**

In registrazione:
- lo username deve essere unico (controllo su `GestoreDati.esisteNomeUtente`);
- in caso di conflitto viene proposto un menu di scelta (reinserire username o tornare indietro).

## 3.3 Struttura dei menù e regole di navigazione
- I menù sono costruiti con `MyMenu`.
- Convenzione: `0` = *indietro* / *esci* (dipende dal contesto).
- Ogni ruolo ha un menù principale e alcuni sotto‑menù (es. scelta mese disponibilità per volontario; gestione modifiche per configuratore).

## 3.4 Aggiornamenti “a runtime” di stati e dati
Molte operazioni (visualizzazioni, iscrizioni, disdette) invocano internamente:
- aggiornamento degli **stati delle visite istanza** (proposta/completa → confermata/cancellata 3 giorni prima; confermata → effettuata dopo la data);
- sincronizzazione di attivi/inattivi e archivio per i **tipi visita**.

Conseguenza pratica:
- gli stati **non** cambiano “da soli” se l’applicazione non viene eseguita; si riallineano alla successiva azione/avvio che richiama gli aggiornamenti.
