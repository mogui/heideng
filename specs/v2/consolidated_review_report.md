# Report consolidato - Review di `v2/SPEC.md`

Status: resolved. Applicate tutte le azioni A1-A33. `decisions.md` resta con nome e ruolo attuali, con una sola copia sul branch (A4). La riga Outcomes per `--until` (A25) decade, perché A31 rimuove `--until`.

**Target:** `v2/SPEC.md` (698 righe)
**Modalità:** report consolidato con tutte le rubriche, senza checkpoint
**Split:** FeatureRange 1-698, DetailedRange N/A (nessuna sezione di piano implementativo)
**Contesto Goldfish:** nessuno

## Stage eseguiti

| Stage | Revisori | Stato | Report |
|---|---|---|---|
| 5 - Zoom Out iniziale | `egm-zoom-out` | completed | `zoom_out_report.md` |
| 6 - Comprehension Test | `egm-comprehension` | completed | `comprehension_report.md` |
| 7 - Critic Review | `egm-design`, `egm-structural`, `egm-style` | completed | `critic_report.md` |
| 8 - Zoom Out intermedio | `egm-zoom-out` | completed | `intermediate_zoom_out_report.md` |
| 9 - Implementation Readiness | - | saltato: DetailedRange N/A | - |

Il documento non è stato modificato tra uno stage e l'altro.

## Esito

La spec non passa. Il core (pipeline deterministica, verdetti da exit code, ticket sequenziali, holdout separato) regge. Cedono tre punti:
- **Isolamento:** la garanzia centrale (il worker non vede il proprio verificatore) si rompe in due modi concreti: le vie di fuga dalla sandbox e `.git/` scrivibile.
- **Lifecycle:** diversi percorsi non arrivano a una fine definita: dipendenti bloccati, cambio di spec a metà run, merge, segnali, sostituzione di AC tra spec.
- **Complessità:** molti meccanismi secondari. I revisori zoom-out mostrano che la maggior parte delle lacune di lifecycle si chiude togliendo meccanismi, non aggiungendone.

| Priorità | Azioni |
|---|---|
| 🔴 Blocker | 6 |
| 🟡 Major | 21 |
| 🟢 Nit | 6 gruppi |

## Disaccordi tra revisori

| Tema | Posizione A | Posizione B |
|---|---|---|
| Alternatives Considered / Problem Statement | `egm-design` B1, `egm-structural` 1: Blocker, lo standard feature-spec fa fallire una spec senza alternative | `egm-zoom-out` Z8-N1: cerimonia, riga 3 e Principles 3/6/9 contengono già problema e motivazioni |
| Contenuto holdout nell'output | `egm-design` B3: un filtro di contenuto unico su tutto il testo diretto al worker | `egm-zoom-out` Z8-M3: niente filtri nuovi; riformulare il Principle 3 e dichiarare il limite |
| Scala di verify/regression | `egm-design` M6: service condivisi, regression selettiva | `egm-zoom-out` Z8-M7: nessun meccanismo ora, dichiararlo in Limits; Z8-M6: baseline lazy |

Nelle azioni qui sotto, per i disaccordi si sceglie la correzione più piccola che chiude la lacuna, e la posizione dell'altro revisore è riportata.

## 🔴 Blocker

### A1 - Le vie di fuga dalla sandbox annullano l'invisibilità dell'holdout
- **Ambito:** righe 27-28, 141, 163-164, 183-189, 394-396, 597-599
- **Problema:** `test-host`, `services`, `checks` in `sandbox.unsandboxed` e `sandbox.excluded` eseguono codice scritto dal worker sull'host. Quel codice legge `~/.heideng/holdout/`, `state/env` e la home reale, e può riportare i check nel work clone o nell'output del ticket F. Riguarda 4 profili su 8 di default. Anche dentro `srt`, il codice prodotto caricato da un check `run` "can read that check" (riga 396). L'audit (riga 390) vede solo le tool call LLM.
- **Azione:** un unico profilo `srt` rilassato (rete, scritture, device aperti; negazioni di zona attive) che sostituisce `unsandboxed` ed `excluded`; directory del check non leggibile dal codice prodotto; dove il profilo rilassato non basta, report ed esito dicono "verdicts are not holdout-grade". Si semplifica anche la config `sandbox` (A13).

### A2 - `.git/` scrivibile nel work clone esegue codice dell'agente sull'host
- **Ambito:** righe 269-271, 319, 338-341, 640, 664
- **Problema:** hook, `core.hooksPath`, `core.fsmonitor`, filter driver, `.gitattributes` scritti dall'agente o da uno script sotto `srt` vengono eseguiti dai comandi git dell'engine, fuori sandbox, con la `HOME` della persona. Il worker può anche committare da sé.
- **Azione:** git dir nella zona `state` (`--git-dir`/`--work-tree`), già negata a ogni sandbox, oppure `denyWrite: ["<cwd>/.git"]` nei due template. Contratto in Part I: "git metadata is never writable by a scope". Si applica anche ai clone di plan, review, merge. Probe doctor che tenta di scrivere `.git/hooks`.

### A3 - Sostituzione di AC tra spec non definita
- **Ambito:** righe 200, 204, 208, 226, 231-232, 333-334, 386
- **Problema:** gli id AC sono per spec; holdout-gen non può riscrivere né ritirare check di un'altra spec; non è definito quando il ritiro ha effetto; baseline e regression eseguono ancora i check sostituiti. Un AC sostituito ma non elencato fa sì che regression riporti il codice al vecchio comportamento.
- **Azione:** riferimenti qualificati `<slug>/AC-n` in `spec.lock`; i check della nuova spec coprono il nuovo comportamento; baseline e regression saltano i check sostituiti; al merge l'engine (tool) li sposta in `retired/`. I rossi di regression su check che toccano sezioni del contract cambiate dal delta diventano escalation `holdout`, non fix ticket.

### A4 - `decisions.md` va in conflitto a ogni merge con decisioni
- **Ambito:** righe 208, 338, 521
- **Problema:** il branch aggiunge `.heideng/specs/<slug>/decisions.md` (versione accettata). Merge main committa su un clone di `main` la versione del run allo stesso path, poi merge il branch: add/add con contenuti diversi, quindi escalation `merge` ogni volta che un agente ha registrato una decisione.
- **Azione:** una sola copia, sul branch: l'engine committa le decisioni appese sul branch (già committa per ticket); il merge diventa un merge semplice. Si elimina la copia in `state/runs/<run>/decisions.md`.

### A5 - Mancano Non-Goals e Limits
- **Ambito:** righe 1-10; materiale di rischio sparso in Isolation, Stops, Report
- **Problema:** non-goal impliciti (nessun run concorrente, niente CI/remoto, solo Claude Code, single user, OS supportati, AC `manual` non bloccanti) e limiti non dichiarati (valori attesi visibili nell'output dei check, crescita del tempo di regression, prodotti non isolabili, secret raggiungibili da codice con egress, fiducia nelle sessioni accept/resolve). Senza questa sezione ogni revisione legge un compromesso come una lacuna.
- **Azione:** H2 "Non-Goals and Limits" in Part I, 6-10 bullet più una tabella `Risk | Impact | Mitigation | Residual` con link alle sezioni che implementano le mitigazioni.

### A6 - Manca Alternatives Considered (contestato)
- **Ambito:** tutto il documento
- **Problema:** lo standard feature-spec fa fallire una spec con un solo percorso. Scelte senza alternativa documentata: ticket sequenziali su un clone, sandbox OS + `srt`, holdout in zona locale, esecuzione sulla macchina della persona, service per check, contract cumulativo, fast-forward locale di `main` invece di PR, `claude -p`.
- **Posizione contraria:** Z8-N1 lo ritiene cerimonia.
- **Azione:** tabella breve `Decision | Chosen | Rejected | Reason` con 5-8 righe, nella stessa sezione di A5 o subito dopo i Principles.

## 🟡 Major

### Lifecycle

| Id | Azione | Righe |
|---|---|---|
| A7 | **Dipendenti mai sbloccati:** eliminare lo stato `blocked`; un dipendente resta `queued` e non eseguibile finché il padre non è `done`. Definire cosa rimettono in coda resolve e resume | 260, 263, 284, 292, 427, 464, 477 |
| A8 | **Cambio spec a metà run:** un nuovo accept o un resolve che cambia la spec termina il run (`stopped`, reason `spec-changed`); il nuovo run riusa holdout e commit dei ticket. Rende vera l'immutabilità dei check | 210, 236, 246, 474 |
| A9 | **Merge main:** riordinare i passi; il merge di `main` nel branch avviene prima e un conflitto è escalation `merge`; un rosso dopo quel merge apre un nuovo round con baseline ricalcolata; limite ai retry di "back to step 1"; merge main è fatto se l'head del branch è antenato di `main` (idempotenza al resume) | 31, 333, 338-343, 523 |
| A10 | **Segnali:** ogni segnale = `interrupted`; `heideng abort` scrive `aborted` in `run.json` prima di segnalare; exit code distinti per interrupted e aborted; valutare di unire `abort` a `close` | 446-447, 522, 581-582 |
| A11 | **Ticket F che aggirano le escalation:** sospendere i ticket F per check i cui AC sono coperti da ticket `escalated`/`failed`; riportarli come "pending escalation" | 122, 284, 292, 325-327 |
| A12 | **Holdout check difettoso:** implement può restituire `dispute` su un ticket F (escalation `holdout`), oppure escalation `holdout` dopo N round con catena `supersedes` sullo stesso check | 234, 307-313, 429-437 |

### Semplificazione

| Id | Azione | Righe |
|---|---|---|
| A13 | **Config `sandbox`:** ridurre le quattro vie di indebolimento (`$NAME`, `excluded`, `extra`, `unsandboxed`) al profilo rilassato di A1 più `allow`, `read`, `write`; `extra` eliminato o ristretto al caso toolchain macOS | 366, 593-599, 616, 652 |
| A14 | **Nodo docs:** spostarlo prima del merge (ticket docs finale o nodo sul branch prima di regression). Elimina la seconda scrittura di `main`, il clone `docs/` e l'idempotenza docs | 95-97, 119, 344-352, 506, 601, 612 |
| A15 | **`test-host`:** un solo punto di esecuzione (nodo host a fine round); eliminare il campo ticket `checks` e la regola "counts as green". Se resta, riscrivere riga 297 come tre regole esplicite | 84, 154-164, 258, 294-299 |
| A16 | **Baseline lazy:** eliminare il nodo; un check rosso in regression viene rieseguito su `main` e saltato se rosso anche lì | 72, 108, 333-334 |
| A17 | **Doctor:** probe per nodo solo da `heideng doctor` e al cambio di versione heideng, versione Claude Code o manifest; a ogni run solo Requires e isolamento | 400-405, 677, 692-693 |
| A18 | **Part I vs Part II:** spostare in Part II precedenze ponytail/tdd, algoritmo di compressione, `git clean`, `tmp + fsync + rename`, polling, TUI, nomi di tool (`claude`, `/exit`, `stream.jsonl`, `opus`/`sonnet`, `CLAUDE.md`, `HEIDENG_CLAUDE_BIN`); in Part I restano i contratti. Altrimenti ammorbidire riga 10 | 10, 150, 195, 202, 271, 277-282, 291, 305, 390, 492, 586, 605-613, 618, 622 |

### Sicurezza

| Id | Azione | Righe |
|---|---|---|
| A19 | **Output dei check:** riformulare il Principle 3 (il worker non legge né modifica il codice dei check, riceve l'output del verdetto); a riga 305 allowlist sulle righe che citano file prodotto invece della denylist; esposizione dei valori attesi in Limits. Posizione alternativa (`egm-design` B3): filtro di contenuto unico su tutto il testo diretto al worker | 28, 305, 310, 471 |
| A20 | **Secret:** dichiarare che `state/env` contiene solo secret di test; egress di `sandbox.allow` limitato al build; voce in Limits | 410, 594, 642, 667 |
| A21 | **Accept/resolve:** documentare il confine di fiducia; permission mode di default o scrittura limitata alla staging invece dell'auto-trust | 195, 371, 457, 698 |
| A22 | **Escalation `leak`:** definire l'effetto per ogni tipo di nodo (non solo implement), lo stato del ticket, se il round continua, la closure attesa; dichiarare l'audit come tripwire | 390, 437, 463-466 |

### Completezza

| Id | Azione | Righe |
|---|---|---|
| A23 | **Run e lock:** definire "active run" (e unificarlo con "open run"), quali esiti rilasciano `run.lock`, dove vive `spec.lock`, cosa rende un lock "valid" | 6, 168, 208, 210, 246, 449, 485, 581-582 |
| A24 | **Output dei test:** definire come l'engine individua "failing tests" e "first error line" in output arbitrario, o una convenzione di output nel contract | 291, 312, 417 |
| A25 | **Config e comandi mancanti:** chiavi e default per i limiti per chiamata (`--max-budget-usd`, `--max-turns`, timeout); riga Outcomes per `--until`; posizione e formato dei file di `to-tickets`; procedura per inserire i valori in `state/env` | 240-244, 410, 424, 486, 575, 578, 592-614, 653 |
| A26 | **Terminologia:** rinominare il campo ticket `checks` e il nodo "holdout check"; separare i due sensi di `blocked` (si risolve con A7); glossario per `exec`, `base_round`, seams, ladder, cache dirs, `fixed`, red/green, Requires, flaky, host directory, `E-<n>`, `<slug>`/`<project>`/`<run>`; correggere l'ordine di Smoke nel glossario (`services` prima di `test`); criteri concreti per "most conservative choice"; nomi dei loop coerenti tra Stops, riga 424, escalation e docs | 45-47, 64, 141, 156-161, 234, 258, 271, 278-283, 297, 307, 320, 352, 366, 416-433, 520, 553 |
| A27 | **Diagramma e heading:** marcare dove inizia `heideng run` e dove gira doctor; nodi del flowchart uguali alle righe della tabella; etichette di route secondo la precedenza di riga 122; heading "Product sandbox" univoci con simmetria Worker/Product tra Part I e II; Resolve a H2 accanto ad Accept; Baseline dopo Holdout | 62-122, 141, 191, 333, 392, 398, 453, 655 |

## 🟢 Nit

| Id | Gruppo | Riferimenti |
|---|---|---|
| A28 | **Formattazione:** un solo H1; fence dei tree con tag `text` e notazione zone-relative; layout holdout in un solo posto; righe separatrici nelle tabelle del template report | K7-N7, K7-N8, K7-N9 |
| A29 | **Formulazione:** LLM/llm; verbo mancante a riga 28; build scope a riga 27 vs 362; "one candidate cause"; "validated by"; "Required when"; righe 163-164; "changes neither"; colonna "Sandbox exceptions"; "two interactive sessions"; righe 253, 285, 427, 618; zona `state` a riga 377 | K7-N10..N26 |
| A30 | **Config e runtime:** imporre il Principle 4 nella validazione del config; `git worktree list` prima di push e fast-forward; definire `<extra read-only paths>`/`<extra writable paths>` e lo scripts directory dello scope verify; scrivibilità del cwd dei check (riga 396 vs 664) | K7-N1, K7-N2, K7-N4, C6-N9 |
| A31 | **Rimozioni minori:** accepted copy; resolve multi-escalation; nodo LLM per il sommario del report; `heideng stats`, `--until`, riuso del plan tra run; `close --delete-branch` cancella `holdout/<slug>/` invece di ritirare | Z8-N2, Z5-N2, Z5-N3, Z5-N4, Z8-N3 |
| A32 | **Riferimenti:** sorgente di `codebase-design` e `domain-modeling`; spiegare "the working vocabulary"; Product sandbox da un solo set di regole | Z5-N6, C6-N3, K7-N24, Z5-N7 |
| A33 | **Punti aperti minori:** effetto di `n` all'accept e di `y` con altre escalation aperte; doctor ad accept; chi popola `holdout/inputs/<slug>/` e cwd dell'oracle; input del nodo report; build rosso in verify e review; campo evento `v`; exit code degli altri comandi in errore; `summary` in `supersedes`; `covers` dei ticket F di build/host; canary; collisioni di slug con directory fisse (`inputs`, `retired`, `runs`, `home`, `doctor`) | C6-N2, C6-N4..N8, C6-N10..N13 |

## Matrice di tracciabilità

| Azione | Stage 5 | Stage 6 | Stage 7 | Stage 8 |
|---|---|---|---|---|
| A1 | Z5-M3 | - | K7-B3 (c) | Z8-B1 |
| A2 | - | - | K7-B4 | Z8-M1 |
| A3 | - | C6-B1 | K7-M3 | Z8-M5 |
| A4 | - | C6-M2 | - | Z8-M4 |
| A5 | - | - | K7-B2, K7-M10 | Z8-M2, Z8-M7 |
| A6 | - | - | K7-B1 | Z8-N1 (contro) |
| A7 | - | C6-M3 | K7-M13 | Z8-M4 |
| A8 | - | C6-M1 | K7-N3 | Z8-M4 |
| A9 | - | C6-M12 | K7-M4, K7-M5 | Z8-M4 |
| A10 | - | - | K7-M9 | Z8-M4, Z8-N4 |
| A11 | - | - | K7-M1 | - |
| A12 | - | - | K7-M2 | - |
| A13 | Z5-M3 | - | - | Z8-B1 |
| A14 | Z5-M1 | - | - | Z8-M4 |
| A15 | Z5-M2 | C6-M13 | K7-M18 | - |
| A16 | - | - | K7-M6 | Z8-M6 |
| A17 | Z5-M5 | - | - | - |
| A18 | Z5-M4 | - | K7-M11 | - |
| A19 | Z5-N5 | - | K7-B3 (a, b) | Z8-M3 |
| A20 | - | - | K7-M7 | - |
| A21 | - | - | K7-M8 | - |
| A22 | Z5-N5 | C6-M4 | K7-N4, K7-N23 | - |
| A23 | - | C6-M5, C6-M11 | K7-N20 | - |
| A24 | - | C6-M6 | - | - |
| A25 | Z5-N4 | C6-M7, C6-M8, C6-M9, C6-M10 | - | - |
| A26 | - | C6-N1, C6-N2 | K7-M12..M17, K7-M19, K7-M22 | - |
| A27 | Z5-N1 | C6-N6 | K7-M20, K7-M21, K7-N5, K7-N6 | - |
| A28 | - | - | K7-N7, K7-N8, K7-N9 | - |
| A29 | - | - | K7-N10..N26 | - |
| A30 | - | C6-N9 | K7-N1, K7-N2, K7-N4 | - |
| A31 | Z5-N2, Z5-N3, Z5-N4 | - | - | Z8-N2, Z8-N3 |
| A32 | Z5-N6, Z5-N7 | C6-N3 | K7-N24 | - |
| A33 | - | C6-N2, C6-N4..N8, C6-N10..N13 | K7-N23 | - |
| (K7-M6 scala verify) | - | - | K7-M6 | Z8-M7: chiuso con A5 e A16 |
