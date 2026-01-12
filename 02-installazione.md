# 2. Manuale di installazione

## 2.1 Requisiti software
- **JDK**: Java 17 (consigliato) o comunque una versione compatibile con `java.time` e record (il progetto usa `record`).
- **Sistema operativo**: Windows / macOS / Linux (qualsiasi, purché supporti la JVM).
- **Dipendenze** (necessarie a compilare/eseguire):
  - `it.unibs.fp.mylib` (classi `InputDati`, `MyMenu`) – libreria usata per input e menù console;
  - Jackson:
    - `jackson-core`, `jackson-databind`, `jackson-annotations`,
    - `jackson-datatype-jsr310` (supporto a `LocalDate`, `YearMonth`, ecc.).

> Se il progetto è consegnato con una cartella `lib/` contenente i `.jar`, usarla nel classpath. In alternativa, usare Maven/Gradle (se presenti i file di build).

## 2.2 Struttura consigliata del progetto
Esempio (adattare ai vostri nomi cartelle):
```
project/
  src/
    ingegneriaSoftware/
      Main.java
      ...
  lib/
    mylib.jar
    jackson-*.jar
  DATA/            # creato automaticamente se mancante
```
L’applicazione cerca la cartella dati **relativa alla directory di esecuzione**: `./DATA`.

## 2.3 Compilazione (senza tool di build)
Da cartella `project/`:

### Linux/macOS (bash)
```bash
mkdir -p out
javac -cp "lib/*" -d out $(find src -name "*.java")
```
Esecuzione:
```bash
java -cp "out:lib/*" ingegneriaSoftware.Main
```

### Windows (PowerShell)
```powershell
mkdir out
javac -cp "lib/*" -d out (Get-ChildItem -Recurse -Filter *.java src | % FullName)
java -cp "out;lib/*" ingegneriaSoftware.Main
```

## 2.4 Esecuzione tramite JAR (se previsto)
Se avete un JAR eseguibile (fat‑jar o con manifest):
```bash
java -jar nomefile.jar
```
**Importante**: eseguire sempre dalla directory che contiene (o può creare) la cartella `DATA`, oppure modificare il path nel codice (`Main` → `caricaDaDirectory`).

## 2.5 Struttura dei file e dati persistenti
La cartella dati è creata/gestita da `FileIO`. I nomi file sono fissi:

- `credenziali.json` – utenti (configuratore/volontario/fruitore)
- `luoghi.json` – luoghi
- `visite.json` – tipi di visita (catalogo attivo)
- `generale.txt` – dati generali (ambito territoriale, max prenotabili per fruitore)
- `datePrecluse.txt` – date precluse globali
- `disponibilitaVolontari.txt` – disponibilità (mappa volontario → date)
- `statoSistema.json` – mese target e fase (raccolta aperta/chiusa, piano prodotto)
- `visiteIstanze.json` – piano mensile (istanze proposte/confermate/cancellate…)
- `archivioStorico.json` – storico (tipi visita scaduti + istanze effettuate)

Dettagli su formato e significato in [A1 – Persistenza](A1-persistenza.md).

## 2.6 Prima inizializzazione del sistema (cosa serve per partire)
Al primo avvio, se `DATA/` è vuota:
1. la cartella `DATA/` viene creata;
2. **non** vengono creati utenti automaticamente dal codice: serve che esista almeno un **configuratore** in `credenziali.json` (di solito fornito dai dati iniziali di consegna).
3. al primo login da configuratore, se `generale.txt` è mancante, il sistema richiede:
   - ambito territoriale;
   - numero massimo di persone prenotabili da un fruitore per singola iscrizione.

Se provi ad accedere come volontario/fruitore senza `generale.txt`, il sistema blocca alcune operazioni e (per i ruoli diversi dal configuratore) non permette di procedere con l’applicazione dei dati generali.
