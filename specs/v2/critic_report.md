# Stage 7 - Critic Review

**Target:** `v2/SPEC.md` (FeatureRange 1-698)
**Standard:** `feature-spec-standard.md` (design, structural); rubric della persona (style)
**Contesto Goldfish:** nessuno

## Stato dei revisori

| Revisore | Stato | Verdetto |
|---|---|---|
| `egm-design` | completed | non passa: 4 Blocker, 10 Major, 6 Nit |
| `egm-structural` | completed | non passa: 1 Blocker, 3 Major, 7 Nit |
| `egm-style` | completed | 0 Blocker, 11 Major, 14 Nit |

## Riepilogo (dopo deduplicazione)

| Severità | Conteggio |
|---|---|
| 🔴 Blocker | 4 |
| 🟡 Major | 22 |
| 🟢 Nit | 26 |

## 🔴 Blocker

### K7-B1 - Manca la sezione Alternatives Considered
- **Lenti:** `egm-design` B1, `egm-structural` 1
- **Ambito:** tutto il documento
- **Problema:** la Alternatives Rule dello standard fa fallire una spec che propone un solo percorso. Scelte presentate come fatti: ticket sequenziali su un clone (vs worktree paralleli); sandbox OS + `srt` (vs container/VM/utente separato); holdout in zona separata (vs verifier remoto); esecuzione sulla macchina della persona (vs CI); start/stop dei service per check (vs lifetime condiviso); un solo `contract.md` cumulativo; fast-forward di `main` locale (vs PR); binding `claude -p`; judge LLM solo per review e oracle.
- **Raccomandazione:** H2 "Alternatives considered" in Part I, tabella `Decision | Chosen | Rejected | Reason`.

### K7-B2 - Mancano Risks & Limits e i limiti di scala
- **Lenti:** `egm-design` B2, `egm-structural` 2
- **Ambito:** tutto il documento (materiale sparso in Isolation, Stops, "Isolation weakened")
- **Problema:** non sono dichiarati costo atteso per ticket/round, crescita della regression col numero di spec attive, tetto di throughput sequenziale, garanzie perse con ogni impostazione `unsandboxed`/`excluded`/`extra`/`$NAME`, modello di fiducia delle sessioni accept/resolve. Rischi senza mitigazione: check fuori `srt` che leggono la zona work; host directory condivisa tra build e verify (canale worker -> verifier); judge sullo stesso provider.
- **Raccomandazione:** H2 "Risks and Limits" con `Risk | Impact | Mitigation | Residual` e link alle sezioni che implementano le mitigazioni.

### K7-B3 - L'invisibilità dell'holdout (Principle 3) si rompe tramite output e script fuori sandbox
- **Lente:** `egm-design` B3
- **Ambito:** righe 27, 141, 163, 184-189, 297-299, 305-310, 394-396
- **Problema:**
  - (a) il codice prodotto caricato da un check `run` (test runner, overlay compilato, config scritta dall'agente) "can read that check" (riga 396). Il suo stdout finisce nel ticket F (riga 310); vengono oscurati solo i path, non il contenuto. Un hook del runner può stampare il sorgente del check.
  - (b) anche senza malizia, Playwright stampa frame di codice dello spec fallito, XCTest le espressioni delle assert. Il filtro sul contenuto overlay esiste solo per il build rosso (riga 305).
  - (c) `test-host` e `checks` in `sandbox.unsandboxed` (default in 4 profili su 8) eseguono codice dell'agente senza `srt`: leggono `~/.heideng/holdout/**`, `state/env`, il canary e la home reale. L'output torna al worker col ticket F di host. L'audit (riga 390) guarda solo le tool call LLM.
- **Raccomandazione:** un unico filtro di contenuto (lo stesso line-match di riga 471) su tutto il testo diretto al worker; check directory non leggibile dal codice prodotto; per le voci unsandboxed, utente OS separato o container, oppure esito marcato "holdout not independent"; rischio residuo in Risks.

### K7-B4 - Git hook e config nel work clone scrivibile eseguono codice sull'host
- **Lente:** `egm-design` B4
- **Ambito:** righe 269-271, 319, 338-341, 640
- **Problema:** il cwd del worker è il work clone, `.git/` incluso. Agente o script sotto `srt` possono scrivere `.git/hooks/*`, `core.fsmonitor`, `core.hooksPath`, `core.sshCommand`, filter/diff driver o `.gitattributes`. L'engine poi esegue `git commit`/`push`/`status`/`clean` sull'host senza sandbox, con la `HOME` della persona. Cade l'intero modello di isolamento (Principles 3 e 8).
- **Raccomandazione:** negare scrittura su `<cwd>/.git` nei settings worker e `srt`, o git dir fuori dal cwd (`--separate-git-dir` in `state`); git dell'engine con `-c core.hooksPath=/dev/null -c core.fsmonitor=false`, allowlist dei protocolli, senza config locale; probe doctor che tenta di scrivere `.git/hooks`.

## 🟡 Major

### K7-M1 - Route crea ticket F che aggirano le escalation
- **Lente:** `egm-design` M1 - **Ambito:** righe 122, 284, 292, 325-327
- **Problema:** con un ticket T escalated i check dei suoi AC falliscono, nascono ticket F, parte un nuovo round: il worker implementa proprio ciò che era stato dichiarato `blocked`, scegliendo "the most conservative choice". Si brucia budget fino alla patience dei round.
- **Raccomandazione:** sospendere i ticket F per check i cui AC sono coperti da ticket `escalated`/`failed`/`blocked`; riportarli come "pending escalation".

### K7-M2 - Nessun percorso per un holdout check difettoso dopo la generazione
- **Lente:** `egm-design` M2 - **Ambito:** righe 234, 307-313, 429-437
- **Problema:** assert sbagliata, check flaky per natura o in contrasto con una decisione di accept producono ticket F irrisolvibili. Il run si ferma solo per round o budget, con escalation etichettata `rounds`.
- **Raccomandazione:** implement può restituire `dispute` (escalation `holdout`), oppure escalation `holdout` dopo N round con catena `supersedes` sullo stesso check.

### K7-M3 - Regression può riportare indietro il prodotto
- **Lente:** `egm-design` M3 - **Ambito:** righe 200, 232, 333-334
- **Problema:** se accept non elenca un AC modificato (es. route Drive rinominata), i vecchi check vanno rossi e i fix ticket spingono il codice verso il comportamento vecchio.
- **Raccomandazione:** i rossi su check che toccano sezioni del contract cambiate diventano escalation `holdout`; oppure l'accept check confronta il delta del contract con i riferimenti Drive/Observe dei check attivi.

### K7-M4 - Effetti esterni non idempotenti su crash e resume
- **Lente:** `egm-design` M4 - **Ambito:** righe 31, 270, 338-343, 351, 523
- **Problema:** push, fast-forward di `main`, commit e fast-forward docs avvengono prima del salvataggio di stato. Un crash dopo `update-ref` fa ripartire merge main ("main moved"), con possibile doppio commit di `decisions.md` e doppio commit docs.
- **Raccomandazione:** intent record prima di ogni effetto git esterno; postcondizioni verificate al resume.

### K7-M5 - Merge main passo 2 sottospecificato, loop illimitati
- **Lenti:** `egm-design` M5 (vedi anche `comprehension_report.md` C6-M12) - **Ambito:** righe 333, 339, 342
- **Problema:** non è detto cosa produce un rosso al passo 2, né il conflitto nel merge di `main` nel branch; baseline calcolata sul vecchio `main`; "back to step 1" senza limite.
- **Raccomandazione:** rosso = rientro nei round con baseline ricalcolata; conflitto = escalation `merge`; limite ai retry.

### K7-M6 - Verify e regression crescono in modo super-lineare
- **Lente:** `egm-design` M6 - **Ambito:** righe 159, 306-307, 333-334
- **Problema:** start/stop dei service per check, attesa fino a `ready_timeout`, doppia esecuzione dei rossi, regression su tutte le spec attive a ogni run e di nuovo al merge. Mai stimato contro il budget di 6h.
- **Raccomandazione:** lifetime condiviso dei service per step con reset dati dichiarato nel contract; regression selettiva con run completo periodico; tempi attesi in Risks.

### K7-M7 - I secret arrivano a codice dell'agente con accesso di rete
- **Lente:** `egm-design` M7 - **Ambito:** righe 410, 594, 642, 667
- **Problema:** valori iniettati in `start`, `services`, `ready`, check `run`, scritti dal worker, con `sandbox.allow` (es. `registry.npmjs.org`, che accetta publish). La redazione copre solo i file dell'engine.
- **Raccomandazione:** dichiarare che in `state/env` vanno solo secret di test; egress limitato al build; voce in Risks.

### K7-M8 - Accept e resolve: superficie di prompt injection fuori sandbox
- **Lente:** `egm-design` M8 - **Ambito:** righe 195, 371, 457, 698
- **Problema:** resolve gira fuori sandbox, con `HOME` e config della persona, in una cartella marcata trusted, e legge evidenze prodotte da codice dell'agente. L'auto-trust elimina l'unico dialog.
- **Raccomandazione:** permission mode di default o sandbox con scrittura limitata alla staging; documentare il confine di fiducia.

### K7-M9 - Un SIGTERM qualsiasi abortisce il run in modo definitivo
- **Lente:** `egm-design` M9 - **Ambito:** righe 447, 522, 581
- **Problema:** logout, shutdown, chiusura del multiplexer producono `aborted`, non riprendibile. Su run di 6h in laptop è probabile.
- **Raccomandazione:** `heideng abort` scrive un marker o usa un segnale dedicato; SIGTERM e SIGHUP nudi diventano `interrupted`.

### K7-M10 - Mancano Problem Statement e Goals/Non-Goals
- **Lenti:** `egm-design` M10, `egm-structural` 3 - **Ambito:** righe 1-10
- **Problema:** l'intro dice cosa è Heideng, non quale problema risolve né per chi. Non-goal impliciti: niente run concorrenti, niente esecuzione remota/CI, solo Claude Code, nessuna persona durante il run, AC `manual` non bloccanti, OS supportati, singolo utente.
- **Raccomandazione:** H2 "Problem Statement" e "Goals and Non-Goals" prima di References.

### K7-M11 - Part I viola la propria regola "changes with the design"
- **Lente:** `egm-structural` 4 - **Ambito:** riga 10 vs righe 195, 202, 371, 390, 403, 498, 605-613, 618, 622
- **Problema:** Part I nomina `claude`, `/exit`, `stream.jsonl`, Claude Code config, `opus`/`sonnet`, `CLAUDE.md`, `HEIDENG_CLAUDE_BIN`, `srt`, plugin.
- **Raccomandazione:** spostare in Part II con termini neutri in Part I, oppure ammorbidire la riga 10.

### K7-M12 - "check" ha troppi significati
- **Lente:** `egm-style` M1 - **Ambito:** righe 45, 70, 106, 141, 156, 203, 233, 258, 297, 467
- **Problema:** unità holdout, nodo "holdout check", accept check, resolve check, graph-check, campo ticket `checks`, voce `checks` in `sandbox.unsandboxed`. "Holdout check" indica sia il validatore sia gli artefatti.
- **Raccomandazione:** rinominare il campo ticket (es. `extra_scripts`) e il nodo (es. "holdout lint"); usare sempre "check `run`" per l'eseguibile.

### K7-M13 - "blocked" ha due significati opposti
- **Lente:** `egm-style` M2 - **Ambito:** righe 260, 271, 283-284
- **Problema:** output `blocked` dell'agente -> stato `escalated`; stato `blocked` = dipendente in attesa. Riga 271 usa il primo senso sotto la tabella del secondo.
- **Raccomandazione:** rinominare uno dei due (es. `waiting` o `cannot_proceed`) e correggere riga 271.

### K7-M14 - Il glossario "Smoke" contraddice Scripts by step
- **Lente:** `egm-style` M3 - **Ambito:** riga 47 vs 156-161
- **Problema:** glossario: `services` dopo `test`; riga 161: `services` parte prima di `test`.
- **Raccomandazione:** il glossario rimanda alla tabella o dichiara l'ordine reale: build, services, services-ready, test, test-host, start, ready.

### K7-M15 - `exec` usato molto prima di essere definito
- **Lenti:** `egm-style` M4, `egm-structural` 10 - **Ambito:** righe 297, 333-334, 363, 499, 508, 515-517; definito a riga 520
- **Raccomandazione:** voce di glossario con i formati della chiave.

### K7-M16 - Termini usati e mai definiti
- **Lenti:** `egm-style` M5, `egm-structural` 10, `egm-design` N2 - **Ambito:** righe 64, 234, 271, 278, 280, 307, 320, 366, 553
- **Problema:** `base_round`, "seams", "ladder", "cache dirs", `fixed`, red/green rispetto agli exit code, "Requires", "flaky", "host directory", id `E-<n>`.
- **Raccomandazione:** glossario o definizione inline; per seams e ladder una frase o il puntatore al file della skill.

### K7-M17 - "most conservative choice" non verificabile
- **Lente:** `egm-style` M6 - **Ambito:** riga 283
- **Raccomandazione:** criteri concreti (nessuna nuova dipendenza, nessun cambio di API pubblica, diff minimo, comportamento esistente preservato) o puntatore al prompt.

### K7-M18 - Regola di riuso del nodo host illeggibile
- **Lente:** `egm-style` M7 (stesso punto di C6-M13) - **Ambito:** riga 297
- **Raccomandazione:** tre regole: l'engine registra lo sha di ogni host verde; host saltato se HEAD coincide; un test step verde con `test-host` registra il proprio commit.

### K7-M19 - Riga 366 contiene quattro regole in una frase
- **Lente:** `egm-style` M8 - **Ambito:** riga 366
- **Raccomandazione:** un bullet per regola; "host directory" nel glossario.

### K7-M20 - Heading "Product sandbox" duplicato
- **Lenti:** `egm-style` M9, `egm-structural` 6 - **Ambito:** righe 141, 392, 655
- **Problema:** `#product-sandbox` porta alla prosa di Part I; i settings `srt` di Part II diventano `#product-sandbox-1`. "Worker sandbox" esiste solo in Part II.
- **Raccomandazione:** "Product sandbox model" (Part I) e "Product sandbox settings" (Part II); simmetria tra le parti.

### K7-M21 - Il diagramma non mostra i confini dei comandi
- **Lenti:** `egm-style` M10, `egm-structural` 7, `egm-design` N1 - **Ambito:** righe 62-120
- **Problema:** accept, lock, doctor, holdout-gen sembrano un unico flusso; doctor gira anche a `init` e `accept`. Flowchart e tabella dei nodi non corrispondono uno a uno. Il flowchart manda "open escalations" direttamente al report, mentre il testo (riga 122) apre un nuovo round se ci sono ticket eseguibili.
- **Raccomandazione:** marcare dove inizia `heideng run`; allineare nodi del flowchart e righe della tabella; etichette di route secondo la precedenza del testo.

### K7-M22 - Nomi dei loop non coerenti
- **Lente:** `egm-style` M11 - **Ambito:** righe 352, 416-419, 424, 432-433
- **Problema:** tabella Stops: `rounds`, `visits`, `holdout-oracle`, `holdout-check`, `graph-check`. Riga 424: "new pass for holdout-gen and plan". Escalation: "holdout loops exhausted". Riga 352: "exhausted docs node" senza loop definito.
- **Raccomandazione:** un set fisso di nomi; dichiarare quale budget esaurisce docs.

## 🟢 Nit

| Id | Lente | Riga | Problema | Raccomandazione |
|---|---|---|---|---|
| K7-N1 | design N3 | 604-613 | Principle 4 non imposto: config può dare a review lo stesso modello di implement | validazione del config |
| K7-N2 | design N4 | 210, 270, 341 | checkout di `heideng/<slug>` a metà run fa fallire il push; `update-ref` su `main` in un altro worktree | `git worktree list` prima di push e fast-forward; fallimenti -> escalation `merge` |
| K7-N3 | design N5 | 210, 236 | `accept` su run aperto vs check immutabili | accept richiede run fermo; holdout rigenerato al resume |
| K7-N4 | design N6 | 390, 663-664 | audit best-effort non dichiarato; `<extra ... paths>` non definiti | dichiarare sandbox come controllo, audit come tripwire; definire i path |
| K7-N5 | structural 5 | 191, 453 | Accept è H2, Resolve è H3 sotto Stops | promuovere Resolve a H2 |
| K7-N6 | structural 7 | 333, 398 | Baseline descritta sotto Regression, doctor sotto Isolation | Baseline dopo Holdout; puntatore a doctor in ordine di pipeline |
| K7-N7 | structural 8 | 1, 21, 624 | tre H1 (MD025) | un solo H1, Parts a H2 |
| K7-N8 | structural 9 | 214-223, 483-518 | fence senza linguaggio; due notazioni diverse per la zona holdout | tag `text`; notazione zone-relative; layout holdout in un solo posto |
| K7-N9 | structural 11 | 557-566 | tabelle del template report senza riga separatrice | aggiungere `\|---\|` |
| K7-N10 | style N1 | 28 | "LLM" vs "llm" | uniformare |
| K7-N11 | style N2 | 28 | "assess only what code cannot" senza verbo | "what a script cannot assess" |
| K7-N12 | style N3 | 27 vs 362 | build scope più ampio nella tabella Scopes | allineare |
| K7-N13 | style N4 | 30 | "one candidate cause" assoluto | "the only code change since the last green commit" |
| K7-N14 | style N5 | 50 | "approved by graph-check" | "validated by" |
| K7-N15 | style N6 | 134 | colonna "Required" mista | "Required when" |
| K7-N16 | style N7 | 163-164 | "add it" ambiguo; frase finale confusa | "When the sandbox blocks a test, the worker moves it to `test-host` instead of returning `blocked`." |
| K7-N17 | style N8 | 178 | "changes neither" senza referente | esplicitare |
| K7-N18 | style N9 | 180-189 | colonna "Outside `srt`" contiene allargamenti di sandbox | "Sandbox exceptions" |
| K7-N19 | style N10 | 3 vs 193 | "two steps where a person takes part" vs report e AC manual | "two interactive sessions" |
| K7-N20 | style N11 | 210 | "a branch" non specificato; "open run" vs "active run" | `heideng/<slug>`; un solo termine |
| K7-N21 | style N12 | 253 | "a red host step or regression" ambiguo | "a red host step, or a red regression" |
| K7-N22 | style N13 | 285 | soggetto mancante per "mandatory note" | "The engine adds a mandatory review note when..." |
| K7-N23 | style N14 | 390 | "voids the visit" vale solo per implement | definire per gli altri nodi |
| K7-N24 | style N14 | 16 | "the working vocabulary" vago | nominare file o termini |
| K7-N25 | style N14 | 427, 618 | "set a precise one"; "heideng's prompt names the phase" | "explicit allowance"; dire dove |
| K7-N26 | style N14 | 373 vs 377 | zona `state` "Writable by the engine" sotto un'intro sugli scope in sandbox | riformulare |
