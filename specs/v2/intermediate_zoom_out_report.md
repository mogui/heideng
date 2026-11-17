# Stage 8 - Zoom Out intermedio

**Target:** `v2/SPEC.md` (righe 1-698), invariato rispetto allo Stage 5
**Revisore:** `egm-zoom-out` - completed
**Standard:** `zoom-out-standard.md`
**Input:** `zoom_out_report.md` e le lacune emerse negli Stage 6-7, date al revisore come affermazioni da valutare

## Verdetto

- Context sufficiency superata: la riga 3 e i Principles dicono cosa fa il tool e perché.
- Quasi tutte le lacune sollevate sono reali. Per la maggior parte la correzione più semplice toglie un meccanismo o aggiunge una frase che dichiara un limite accettato.
- Due lacune richiedono un piccolo meccanismo nuovo: script fuori sandbox (profilo `srt` rilassato) e `.git` (una regola di deny).
- Chiudere ogni lacuna di lifecycle con nuova gestione renderebbe la spec più complessa senza necessità.

| Severità | Conteggio |
|---|---|
| 🔴 Blocker | 1 |
| 🟡 Major | 7 |
| 🟢 Nit | 4 |

## Findings

### Z8-B1 - Ogni via di fuga dalla sandbox annulla il Principle 3 - Blocker
- **Ambito:** righe 28, 141, 163-164, 183-189, 597-599, 636
- **Problema:** `test-host`, `services`, `checks` in `unsandboxed` e `sandbox.excluded` eseguono sull'host, con l'utente della persona, codice scritto dal worker. Quel codice può leggere `~/.heideng/holdout/` e copiare i check nel work clone, dove la sessione implement successiva li legge. L'audit (riga 390) non lo vede. CLI, macOS, iOS e Desktop richiedono una di queste vie: la garanzia centrale non vale per metà dei profili. "Isolation weakened" non dice che la garanzia holdout è annullata.
- **Raccomandazione:** sostituire l'interruttore tutto-o-niente con un unico profilo `srt` rilassato: rete, scritture e device aperti, negazioni di zona attive (holdout e state negati agli script build, work e state negati agli script di check). Un profilo al posto di `unsandboxed` + `excluded` + parte di `extra` (riduce anche Z5-M3). Dove neanche così funziona, dichiararlo in Limits e stampare nel report "verdicts are not holdout-grade".

### Z8-M1 - Il worker può scrivere `.git/` e l'engine esegue git sull'host - Major
- **Ambito:** righe 269-271, 640, 664
- **Problema:** `allowWrite` include il cwd, quindi `.git/`. Hook o config (`core.hooksPath`, `core.fsmonitor`, filter driver) vengono eseguiti dai `git commit`/`reset`/`push` dell'engine fuori sandbox. Il worker può anche committare da sé, contro "only the engine runs git commands that write".
- **Raccomandazione:** `denyWrite: ["<cwd>/.git"]` in entrambi i template, oppure git dir nella zona `state` (`--git-dir`/`--work-tree`), già negata a ogni sandbox. Contratto in Part I: "git metadata is never writable by a scope". Vale anche per i clone di plan e review.

### Z8-M2 - Manca una sezione Non-Goals / Limits - Major
- **Ambito:** righe 1-10, struttura
- **Problema:** il problem statement c'è già (riga 3), la motivazione della scelta principale è nel Principle 6. Manca Non-Goals / Limits: senza, ogni passata di review legge un compromesso accettato come una lacuna.
- **Raccomandazione:** 6-10 bullet. È il modo più economico per chiudere Z8-B1 (fallback), Z8-M3 e Z8-M7.

### Z8-M3 - Contenuto holdout nell'output dei check: dichiarare la garanzia, niente filtri - Major
- **Ambito:** righe 28, 305, 310, 471
- **Problema:** il Principle 3 passa deliberatamente l'output dei check al worker; "expected X, got Y" espone i valori attesi. Trattamento disomogeneo: solo path sostituiti (riga 310), scrub a denylist fragile (riga 305), filtro a substring (riga 471). Un terzo filtro indebolirebbe i ticket F.
- **Raccomandazione:** riformulare il Principle 3 (il worker non legge né modifica il codice dei check, riceve l'output del verdetto); esposizione dei valori attesi in Limits; a riga 305 allowlist (tenere solo le righe che citano file prodotto). Supporta l'eliminazione di `leak_min_chars` (Z5-N5).

### Z8-M4 - Lacune di lifecycle: chiuderle togliendo stati e percorsi - Major
- **Ambito:** righe 210, 236, 260, 284, 292, 338-342, 346-352, 455-479, 522-523, 581-582
- **Correzioni proposte:**
  - **dipendenti `blocked` mai rimessi in coda:** eliminare lo stato `blocked`. Un ticket è eseguibile solo con deps `done` (riga 263): i dipendenti restano `queued` e non eseguibili; rimettere in coda il padre li sblocca da sé.
  - **cambio spec a metà run:** un cambio di spec termina il run (`stopped`, reason `spec-changed`); un nuovo `heideng run` riparte a basso costo grazie al riuso dell'holdout e alla regola "ancestor counts as done" (riga 246). L'immutabilità diventa vera per costruzione.
  - **conflitto `decisions.md` al merge:** il passo 1 committa su `main` una versione diversa allo stesso path del branch: conflitto add/add quasi a ogni run. Eliminare la copia in `state`; l'engine committa le decisioni sul branch.
  - **merge main passo 2 senza routing:** merge di `main` nel branch (conflitto = escalation `merge`), poi un nuovo round.
  - **idempotenza di merge e docs:** merge main è fatto se l'head del branch è antenato di `main`; docs sparisce se va prima del merge (Z5-M1).
  - **SIGTERM = abort non riprendibile:** ogni segnale = `interrupted`; `heideng abort` scrive `aborted` in `run.json` prima di segnalare. Valutare di unire `abort` a `close`.

### Z8-M5 - Sostituzione di AC tra spec: holdout-gen non può fare ciò che chiede la riga 232 - Major
- **Ambito:** righe 200, 204, 208, 231-232, 386
- **Problema:** holdout-gen ha cwd `holdout/<slug>/` e legge in sola lettura i check delle altre spec, ma dovrebbe riscriverli o ritirarli. Non è detto quando il ritiro ha effetto (a generazione, un run fallito lascia check altrui ritirati) né come si qualificano gli id AC.
- **Raccomandazione:** i check della nuova spec coprono il nuovo comportamento; `spec.lock` elenca `<slug>/AC-n`; baseline e regression saltano quei check nel run; al merge l'engine (tool) li sposta in `retired/`. Nessuna riscrittura di check altrui.

### Z8-M6 - Baseline è un'esecuzione completa in più per run - Major
- **Ambito:** righe 72, 108, 159, 333-334
- **Raccomandazione:** eliminare il nodo baseline. Quando un check di regression è rosso, eseguirlo una volta su `main`; se è rosso anche lì, saltarlo e riportarlo. Stesso risultato, costo limitato ai rossi, un nodo in meno.

### Z8-M7 - Il tempo di verify e regression cresce con le spec attive: accettarlo come limite - Major
- **Ambito:** righe 159, 306, 334
- **Problema:** crescita lineare con la storia del progetto, dominata dallo startup per check. Parallelismo, selezione per impatto o startup condiviso sono ottimizzazioni premature e indeboliscono il Principle 6.
- **Raccomandazione:** dichiararlo in Limits con le leve già disponibili (Z8-M6, ritiro dei check sostituiti). Rivalutare quando `wall_clock` si esaurisce davvero in regression.

### Z8-N1 - Non aggiungere Alternatives Considered né un Problem Statement separato - Nit
- **Ambito:** righe 1-10
- **Motivo:** riga 3 e Principles 3, 6, 9 contengono già problema e motivazioni. Z8-M2 copre la parte mancante.

### Z8-N2 - La accepted copy duplica il branch - Nit
- **Ambito:** righe 42, 208, 474
- **Motivo:** `git show heideng/<slug>:.heideng/specs/<slug>/spec.md` o `heideng status` danno lo stesso contenuto. Eliminarla.

### Z8-N3 - `close --delete-branch` ritira check di una spec mai attiva - Nit
- **Ambito:** riga 582
- **Motivo:** basta cancellare `holdout/<slug>/`.

### Z8-N4 - Stesso exit code per interrupted e aborted - Nit
- **Ambito:** righe 446-447
- **Motivo:** uno è riprendibile, l'altro no. Sparisce con la semplificazione di Z8-M4; altrimenti codici distinti.

## Sintesi per tipo di chiusura

| Chiusura | Lacune |
|---|---|
| Rimozione | `blocked`, cambio spec a metà run, `decisions.md`, merge main passo 2, idempotenza docs, SIGTERM, costo baseline, riscrittura check altrui |
| Un piccolo meccanismo | `.git` (deny o git dir in `state`), vie di fuga (un profilo `srt` rilassato) |
| Dichiarazione in Limits | valori attesi nell'output, crescita dei tempi della suite, prodotti non eseguibili neanche col profilo rilassato |
