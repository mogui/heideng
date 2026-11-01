# Stage 5 - Zoom Out iniziale

**Target:** `v2/SPEC.md` (righe 1-698)
**Revisore:** `egm-zoom-out` - completed
**Standard:** `zoom-out-standard.md`
**Contesto Goldfish:** nessuno

## Verdetto

- Supera i controlli su contesto e problema radice. Nessun Blocker.
- Obiettivo chiaro: pipeline lights-out da spec approvata a `main`, giudicata da holdout check.
- Riuso degli strumenti esistenti: sandbox di Claude Code, `srt`, skill vendorizzate.
- Ticket sequenziali e verdetti da exit code tengono il core semplice.
- Problema principale: troppi meccanismi secondari costruiti sopra il core.

## Riepilogo

| Severità | Conteggio |
|---|---|
| 🔴 Blocker | 0 |
| 🟡 Major | 5 |
| 🟢 Nit | 7 |

## Findings

### Z5-M1 - Il nodo docs è una seconda pipeline dopo il merge - Major
- **Ambito:** righe 95-97, 119, 344-352, 506, 601, 612
- **Problema:** dopo il merge una sessione LLM separata lavora su un altro clone, con smoke proprio e un secondo fast-forward/CAS di `main`. Risultato: due scritture su `main` per run, due punti di race, modifiche alla documentazione che non passano da review né da verify.
- **Raccomandazione:** portare la documentazione nel flusso prima del merge: un ticket docs finale nel plan, oppure docs sul branch prima di regression. Si eliminano la seconda scrittura su `main`, il clone `docs/` e la semantica di fallimento dedicata.

### Z5-M2 - `test-host` ha tre percorsi sovrapposti - Major
- **Ambito:** righe 84, 154-164, 258, 294-299
- **Problema:** `test-host` gira nel test step del ticket (campo `checks`), nel nodo host a fine round, e con regole per ricordare quale commit è già verde. È un'ottimizzazione introdotta senza un collo di bottiglia misurato. Costa un nodo, un campo ticket, una sorgente di ticket F e stato aggiuntivo.
- **Raccomandazione:** un solo punto. Il più semplice: solo nel nodo host, eliminando `checks` e la regola "counts as green".

### Z5-M3 - Troppi modi per indebolire l'isolamento nel config - Major
- **Ambito:** righe 366, 593-599, 616, 652
- **Problema:** sei chiavi `sandbox`, quattro delle quali (`write: $NAME`, `excluded`, `extra`, `unsandboxed`) indeboliscono l'isolamento in modi diversi, ognuna con regole di risoluzione, voci di report e messaggi doctor propri. `extra` è un deep-merge generico che aggira il modello definito dalla spec.
- **Raccomandazione:** tenere `allow`, `read`, `write`, `unsandboxed`. Eliminare `extra` o restringerlo al caso toolchain macOS. Valutare di unire `excluded` a `unsandboxed`.

### Z5-M4 - Part I scende in dettagli di implementazione e di prompt - Major
- **Ambito:** righe 150, 271, 277-282, 291, 305, 312, 492, 586
- **Problema:** precedenze ponytail/tdd, algoritmo di compressione output, `git clean -fdx`, `tmp + fsync + rename`, polling "every second", scrubbing dell'output di build, raggruppamento per prima riga d'errore, TUI con spinner. Part I dichiara di cambiare col design; questi dettagli cambiano più spesso.
- **Raccomandazione:** spostare le precedenze nel prompt vendorizzato o in Part II. In Part I tenere i contratti: cosa deve preservare la compressione, scritture di stato atomiche, contenuto overlay mai al worker. Ridurre la TUI a "progress su stderr".

### Z5-M5 - Doctor verifica le skill due volte a ogni run - Major
- **Ambito:** righe 400-405, 677, 692-693
- **Problema:** a ogni `run`, `accept` e `init` parte una sessione probe per ognuno dei 9 nodi LLM più i probe di isolamento, oltre al controllo hash delle skill. Il caricamento è già fissato da file vendorizzati e hash del manifest.
- **Raccomandazione:** probe per nodo solo da `heideng doctor` e quando cambiano versione heideng, versione Claude Code o manifest. Tenere a ogni run i controlli economici (Requires, isolamento).

### Z5-N1 - I profili prodotto sono materiale di riferimento nel core - Nit
- **Ambito:** righe 176-189
- **Raccomandazione:** l'engine non legge i profili. In Part I una frase con il significato delle colonne; la tabella in Part II o in una risorsa del prompt di accept.

### Z5-N2 - Resolve multi-escalation - Nit
- **Ambito:** righe 455, 473, 580
- **Raccomandazione:** modalità di comodo con naming di staging e rami di prompt dedicati. Una escalation per sessione basta per v2.

### Z5-N3 - Il sommario del report richiede un nodo LLM dedicato - Nit
- **Ambito:** righe 120, 362, 505, 613
- **Raccomandazione:** eliminarlo o accorparlo a docs o alla review finale.

### Z5-N4 - `heideng stats`, `--until`, riuso del plan tra run - Nit
- **Ambito:** righe 246, 569, 578, 584
- **Raccomandazione:** rimandare tutti e tre se non servono ora.

### Z5-N5 - Due difese anti-leak separate - Nit
- **Ambito:** righe 305, 390, 471
- **Raccomandazione:** dichiarare che sono backstop. Valutare di eliminare `leak_min_chars` facendo passare al resolve agent solo il testo Intent.

### Z5-N6 - Riferimenti skill incompleti - Nit
- **Ambito:** righe 15-18 vs 681-682
- **Raccomandazione:** `codebase-design` e `domain-modeling` sono assegnate a plan e implement ma mancano in References. Indicarne la sorgente.

### Z5-N7 - Product sandbox descritta sia in Part I sia in Part II - Nit
- **Ambito:** righe 392-396, 655-672
- **Raccomandazione:** un solo set di regole in Part I e un solo mapping in Part II, per evitare divergenze.
