# Stage 6 - Comprehension Test

**Target:** `v2/SPEC.md` (FeatureRange 1-698, DetailedRange N/A)
**Revisore:** `egm-comprehension` - completed
**Standard:** nessuno (test di autosufficienza)
**Contesto Goldfish:** nessuno

## What this proposes

Heideng è una CLI "lights-out factory". Una persona e un agente trasformano una spec libera in quattro file committati: `spec.md`, `acceptance.md` (AC con id, ognuno `auto` o `manual`), `contract.md` e `decisions.md`. Finiscono su un branch `heideng/<slug>` insieme a uno `spec.lock`. Poi una pipeline senza persone prende il controllo:
- doctor verifica prerequisiti e isolamento;
- holdout-gen scrive check script in una zona che gli agenti di codice non possono leggere; un tool e un oracle LLM li validano;
- baseline esegue su `main` i check delle spec già mergiate;
- plan divide la spec in ticket con `to-tickets`;
- i ticket sono implementati uno alla volta in un unico work clone, ognuno seguito da smoke in sandbox, commit e push;
- quando nulla è eseguibile, il nodo host esegue i test fuori sandbox (`test-host`), verify esegue gli holdout check su un checkout usa-e-getta con overlay, review esegue la code review LLM;
- route decide tra un altro round, regression poi merge su `main` poi aggiornamento docs, oppure stop con report.

Ogni loop ha un controllo di successo esterno al worker più stop di budget (`patience`, `max_passes`, `usd`, `wall_clock`). Ciò che l'engine non risolve diventa una escalation, chiusa con `heideng resolve` (sessione interattiva) o `heideng resume`. Part II mappa tutto su flag e settings di Claude Code, settings di `srt`, skill vendorizzate e probe del doctor.

## How the existing system works, per this document

Design nuovo, non esiste un heideng precedente. Il documento presuppone:
- un repo git con branch `main`;
- un prodotto che si espone tramite script in `.heideng/` (`build`, `test`, `test-host`, `services`, `services-ready`, `start`, `ready`);
- un `.heideng/contract.md` cumulativo e un `heideng.yaml`;
- Claude Code (`claude -p`) con sandbox OS per gli agenti, `srt` per gli script del prodotto;
- skill vendorizzate da tre repo esterni.

Lo stato vive sotto `~/.heideng/` in tre zone: `state` (solo engine), `work` (scope build), `holdout` (scope holdout e verify). Ogni scope in sandbox scrive solo nella propria zona e non può accedere alle altre.

## Verdetto

Il documento è in gran parte autosufficiente: ogni sezione è parafrasabile. Il Blocker è la sostituzione di AC di una spec già mergiata da parte di una spec successiva. La maggior parte dei Major sono percorsi di lifecycle non seguibili dall'inizio alla fine.

| Severità | Conteggio |
|---|---|
| 🔴 Blocker | 1 |
| 🟡 Major | 13 |
| 🟢 Nit | 13 |

## Findings

### C6-B1 - La sostituzione di AC tra spec non è ricostruibile - Blocker
- **Ambito:** righe 200, 204, 208, 226, 231-232, 386
- **Problema:**
  - gli id AC sono per spec (`AC-1` in ogni `acceptance.md`); non è detto come una spec citi l'AC di un'altra;
  - holdout-gen ha cwd `holdout/<slug>/` e i check delle altre spec sono solo declared reads: non può riscrivere né spostare in `retired/` check di un'altra spec. Non è detto chi lo fa, dove finiscono i check riscritti, a quale spec appartengono;
  - non è detto quando il ritiro ha effetto rispetto a baseline e regression, che eseguono ancora i vecchi check e andrebbero rossi.
- **Raccomandazione:** riferimento AC qualificato (`<slug>/AC-n`), proprietà e posizione dei check riscritti, attore e momento del ritiro, esclusione dei check sostituiti da baseline e regression.

### C6-M1 - Cambio spec a metà run vs "checks are immutable" - Major
- **Ambito:** righe 210, 236, 246, 474
- **Problema:** dopo un nuovo accept o un resolve che cambia la spec non è detto se holdout-gen, oracle e baseline ripartono, come si concilia con l'immutabilità dei check, né come si rientra in plan da un round (il diagramma mostra plan solo prima di ROUND).
- **Raccomandazione:** elencare i nodi che ripartono dopo un cambio di lock e il punto di rientro.

### C6-M2 - `decisions.md` al merge - Major
- **Ambito:** righe 208, 338, 521
- **Problema:** il branch contiene già `.heideng/specs/<slug>/decisions.md` dall'accept. Merge main committa su un clone di `main` la versione del run allo stesso path, poi merge il branch. Con contenuti diversi sembra un conflitto garantito, cioè una escalation `merge` a ogni run con decisioni.
- **Raccomandazione:** committare il `decisions.md` del run sul branch prima del merge, o risolvere quel path automaticamente.

### C6-M3 - I dipendenti `blocked` non tornano mai `queued` - Major
- **Ambito:** righe 263, 284, 292, 427, 464, 477
- **Problema:** resolve rimette in coda il ticket, ma nessuna regola riporta i dipendenti a `queued`. Non è detto se `resume` rimetta in coda ticket o dipendenti.
- **Raccomandazione:** definire le transizioni `blocked -> queued` e `failed/escalated -> queued` per resolve e resume.

### C6-M4 - Escalation `leak` sottospecificata - Major
- **Ambito:** righe 390, 437, 463-466
- **Problema:** "voids the visit (reset of the work clone)" vale solo per implement. Non è definito per plan, review, docs, report, holdout-gen, oracle; né lo stato del ticket, né se il round continua. Nessuna closure (Note, Budget, spec) affronta un leak.
- **Raccomandazione:** gestione del leak per tipo di nodo e closure attesa.

### C6-M5 - "Active run" e lifecycle del lock - Major
- **Ambito:** righe 6, 449, 485, 581-582
- **Problema:** "active run" non è definito. Non è detto se un run `stopped` è attivo, quali esiti rilasciano `run.lock`, né a quale lock si riferisce "releases the lock".
- **Raccomandazione:** definire active run ed elencare gli esiti che rilasciano `run.lock`.

### C6-M6 - Riconoscere test falliti e "first error line" in output arbitrario - Major
- **Ambito:** righe 291, 312, 417
- **Problema:** progresso, compressione e raggruppamento si basano su "failing tests" e "first error line", ma gli script hanno come unico contratto l'exit code.
- **Raccomandazione:** definire l'euristica o una convenzione di output nel contract.

### C6-M7 - Limiti per chiamata LLM senza config - Major
- **Ambito:** righe 424, 592-614, 653
- **Problema:** `--max-budget-usd`, `--max-turns` e timeout non hanno chiavi in `heideng.yaml` né default.
- **Raccomandazione:** aggiungerli sotto `nodes.*` o `limits` con default.

### C6-M8 - Semantica di `--until` - Major
- **Ambito:** riga 578
- **Problema:** esito, `run.end`, exit code e possibilità di resume mancano nella tabella Outcomes.
- **Raccomandazione:** aggiungere una riga.

### C6-M9 - La conversione dell'output di plan dipende da un formato non dichiarato - Major
- **Ambito:** righe 240-244
- **Problema:** non è detto quali file `to-tickets` scrive, dove, in che formato.
- **Raccomandazione:** dichiarare posizione e formato attesi o puntare al contratto della skill.

### C6-M10 - Gestione dei valori dei secret - Major
- **Ambito:** righe 410, 486, 575
- **Problema:** nessun comando o procedura dice come la persona inserisce i valori in `state/env`.

### C6-M11 - Posizione di `spec.lock` e significato di "valid" - Major
- **Ambito:** righe 168, 208, 210, 246
- **Problema:** il contenuto è dato, la posizione no. "Valid lock" non è definito, eppure `run` lo usa per rifiutare e manifest e plan ne usano l'hash.

### C6-M12 - Ordine dei passi di merge main - Major
- **Ambito:** righe 338-342
- **Problema:** il passo 2 ("first merges main into the branch") viene dopo il passo 1 che fa già il merge. Non è detto cosa succede se quel pre-merge confligge o se verify/regression ripetuti falliscono.
- **Raccomandazione:** riordinare i passi e dichiarare il routing dei fallimenti.

### C6-M13 - Regola di riuso del nodo host poco leggibile - Major
- **Ambito:** riga 297
- **Problema:** ricostruzione incerta: host saltato se HEAD coincide con l'ultimo head verde di host o con il commit di un ticket verde che includeva `test-host`. Non è detto chi imposta `checks` oltre ai ticket F di host (plan lo fa?).
- **Raccomandazione:** riscrivere come regola esplicita.

### Nit
- **C6-N1** - `base_round` (riga 320) non definito.
- **C6-N2** - `<slug>`, `<project>`, `<run>` e id escalation (`E-1`, riga 553) non definiti né derivati. Slug come `inputs`, `retired`, `runs`, `home`, `doctor` collidono con directory fisse sotto `holdout/` (righe 509-517).
- **C6-N3** - `codebase-design` e `domain-modeling` senza sorgente (righe 15-18 vs 681-682); "the working vocabulary" (riga 16) non spiegato.
- **C6-N4** - Non è detto esplicitamente che un solo `patience`/`max_passes` vale per tutti i loop (riga 602).
- **C6-N5** - Effetto di `n` al prompt di accept non dichiarato; effetto di `y` con altre escalation aperte ambiguo (righe 473 vs 479).
- **C6-N6** - Doctor gira anche ad accept, ma il diagramma lo mostra solo al run e la posizione (prima o dopo la sessione) manca (riga 400).
- **C6-N7** - Chi popola `holdout/inputs/<slug>/` e quando (righe 221, 231); cwd dell'oracle non dichiarata.
- **C6-N8** - Il nodo report legge solo `sandbox.read`: non è detto come riceve i dati del run.
- **C6-N9** - Riga 396 vs 664: il check `run` scrive con le regole del checkout, ma il template `srt` mette `<cwd>` in `allowWrite`. `<extra read-only paths>`, `<extra writable paths>` e lo "scripts directory" dello scope verify (riga 657) non definiti.
- **C6-N10** - "A branch checked out in the project repo" (riga 210) dovrebbe dire `heideng/<slug>`.
- **C6-N11** - Build rosso in verify (riga 305): "the round ends", ma il diagramma va comunque a review. Review gira?
- **C6-N12** - Nessuna regola se il config imposta per review lo stesso modello di implement (righe 28, 319).
- **C6-N13** - Punti minori non definiti: campo evento `v`; exit code degli altri comandi in errore (riga 451); scopo del `summary` in `supersedes`; `covers` dei ticket F di build/host; tracciamento dei commit del plan tra run (riga 246); `canary` descritto solo dall'uso.
