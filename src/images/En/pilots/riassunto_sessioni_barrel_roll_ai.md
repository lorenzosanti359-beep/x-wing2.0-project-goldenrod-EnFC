# Riassunto sessioni — Barrel Roll AI per Fly Casual (Aggressor AI)

*Documento cumulativo: Sessione 1 (implementazione iniziale) + Sessione 2 (debug del controllo bullseye e fix della priorità; poi estesa a fallimento silenzioso del Barrel Roll per l'AI, calibrazione di Reinforce, e priorità/selezione azione libera per Coordinate + Lieutenant Sai).*

## Scopo

Contributo a **Fly Casual**, simulatore Unity/C# open-source di *X-Wing: The Miniatures Game* (2ª edizione). L'obiettivo iniziale (Sessione 1) era completare l'implementazione dell'azione **Barrel Roll** per l'AI **Aggressor**, includendo sia la logica di valutazione (l'euristica che decide *se* e *come* eseguirlo) sia quella di esecuzione fisica sul tavolo.

La Sessione 2 è nata da un dubbio concreto sull'euristica già scritta: l'utente sospettava che il controllo "ostacolo in bullseye a distanza di una basetta" non funzionasse, dato che l'AI non eseguiva mai il Barrel Roll in quello scenario nonostante fosse tatticamente la scelta migliore. Il lavoro si è esteso, passo dopo passo via log diagnostici, dalla verifica del rilevamento geometrico fino alla scoperta del vero collo di bottiglia: un gate di priorità nel sistema di scelta azioni dell'AI, a valle e indipendente dal rilevamento stesso.

## Architettura generale del gioco (per quanto emerso dai file esaminati)

- **Fasi e Subphase**: `Phases` (statica) gestisce `Phases.CurrentSubPhase`, un'istanza di `GenericSubPhase` o sue derivate. Le subphase si aprono/chiudono con `Phases.StartTemporarySubPhaseNew`/`StartTemporarySubPhaseOld` e si concludono con `Phases.FinishSubPhase`/`Phases.GoBack()`. Ogni subphase ha un campo `CallBack` (delegate `Action`, **non** un metodo virtuale) invocato al completamento — un errore concettuale emerso nella Sessione 1 è stato confondere "chiamare `CallBack()`" con "l'azione è riuscita", quando in realtà dipende da cosa quel delegate rappresenta nel contesto in cui è stato assegnato.

- **Sistema Triggers** (`Triggers.cs`): gestisce eventi di gioco con uno stack di `StackLevel`, ciascuno con una lista di `Trigger` e un callback di livello. `Triggers.RegisterTrigger` registra, `Triggers.ResolveTriggers` attiva, `Triggers.FinishTrigger` chiude il trigger corrente. **Scoperta centrale della Sessione 1**: `FinishTrigger()` invoca *sempre* il callback del livello (tramite `DoCallBack()`) quando non restano altri trigger sullo stack — non esiste un modo di "chiudere" un trigger senza eseguirne il continuo.

- **Azioni** (`ActionsList.GenericAction` e derivate: `BarrelRollAction`, `BoostAction`, `RotateArcAction`, ecc.): ogni nave ha una `ActionBar` con azioni disponibili, colorate Bianco/Rosso/Viola. `GetActionPriority()` è chiamato dall'AI per ogni azione disponibile di ogni nave; `ActionTake()` esegue l'azione scelta; `RevertActionOnFail()` gestisce il fallimento.

- **AI**: `AggressorAiPlayer` (deriva da `GenericAiPlayer`). Il metodo `PerformActionFromList` itera le azioni disponibili di una nave, chiama `GetActionPriority()` per ciascuna, scarta esplicitamente le azioni rosse (eccetto `RotateArcAction` e, dopo la Sessione 1, anche `BarrelRollAction`) forzandone la priorità a `int.MinValue`, ordina le azioni per priorità decrescente e **esegue la migliore solo se la sua priorità è > 0** (`if (prioritizedActions.Value > 0)`). **Scoperta centrale della Sessione 2**: questo gate `> 0` è indipendente da qualunque confronto tra azioni concorrenti — un'euristica può produrre correttamente "questa è la scelta migliore disponibile" e vedersela comunque scartare se il valore assoluto della priorità non supera zero. Qualunque euristica basata solo su penalità (valori ≤0) è strutturalmente incapace di superare questo gate senza un offset esplicito.

- **Comunicazione euristica AI ↔ subphase di esecuzione**: meccanismo `AiPlansStorage`/`AiSinglePlan`, esposto come campo `AiPlans` su `GenericShip`. `AddPlan`/`GetPlanByActionName(nome)`/`RemovePlan`. Canale "corretto", già usato da `BoostAction`/`BoostPlanningSubPhase` prima del lavoro sul Barrel Roll, e adottato anche lì (Sessione 1) dopo averlo scoperto. **Punto chiarito in Sessione 2**: `TryBarrelRollPossibilities` (in `NavigationSubSystem.cs`) *non esegue* il Barrel Roll — si limita a valutare le posizioni candidate e a registrare un piano in `AiPlans` se conviene. L'esecuzione reale dipende dal superare, a valle, sia il confronto con le altre azioni sia il gate `> 0` di `PerformActionFromList`.

- **Geometria nave** (`GenericShip`): `GetPosition()`, `GetAngles()`/`GetRotation()`, `TransformDirection/Point`, `GetLeft()/GetRight()` (basati su `ShipBase.HALF_OF_SHIPSTAND_SIZE`).

- **Ostacoli** (`Obstacles.GenericObstacle`, `ObstaclesManager`): ogni ostacolo ha un `MeshCollider`. `ObstaclesManager.SetObstaclesCollisionDetectionQuality` alterna tra qualità "Low" (`convex=true`, `isTrigger=true`, per query geometriche sincrone via `Physics.OverlapBox`) e "High" (`convex=false`, `isTrigger=false`, per la risoluzione fisica reale). **Verificato in Sessione 2** leggendo il codice sorgente: il toggle è sincrono (nessun problema di timing/bake), e itera `ChosenObstacles` anziché i soli ostacoli piazzati — imprecisione minore, non un bug per l'uso che ne fa `NavigationSubSystem`.

- **Template di manovra** (`BoardTools.ManeuverTemplate`): rappresenta fisicamente un template di movimento. Espone `ApplyTemplate`, `GetFinalPosition`, `DestroyTemplate` e, dopo la Sessione 1, anche `SetVisible`/`IsAlive`.

- **`CustomizedAi`** (namespace `Ship`, esposta come `GenericShip.Ai`): hook generico che permette a qualunque abilità/pilota di modificare priorità calcolate altrove, senza toccare la classe che le calcola. Tre coppie evento+metodo, stesso pattern per tutte: `OnGetWeaponPriority`/`CallGetWeaponPriority`, `OnGetRotateArcFacingPriority`/`CallGetRotateArcFacingPriority`, `OnGetActionPriority`/`CallGetActionPriority` — quest'ultima già invocata in `AggressorAiPlayer.PerformActionFromList` subito dopo `action.GetActionPriority()`, per ogni azione valutata su ogni nave. **Scoperta e usata per la prima volta nella seconda parte della Sessione 2** per il bonus di Lieutenant Sai su Coordinate, senza modificare `CoordinateAction.cs`.

- **`DecisionSubPhase`** (base di `ActionDecisonSubPhase`, `FreeActionDecisonSubPhase`, `ReinforceSideSubphase`, ecc.): il meccanismo generale per le decisioni AI "senza click" è `DoDefault()`, che esegue direttamente la decisione con nome `DefaultDecisionName` (via `GenerateDecisionCommand` + `GameMode.CurrentGameMode.ExecuteCommand`). Per l'azione principale della nave, `PerformActionFromList` lo scavalca calcolando la scelta migliore autonomamente e inviando il comando direttamente; per le altre decisioni (selezione lato di Reinforce, azione libera da Coordinate, ecc.) `DefaultDecisionName` resta l'unica fonte di verità per l'AI, e va quindi calcolato dinamicamente per essere tatticamente valido — se lasciato hardcoded (come era per `FreeActionDecisonSubPhase`, vedi sotto), l'AI esegue sempre la stessa decisione fissa a prescindere dal contesto. **Non confermato con certezza assoluta**: non è stato esaminato `GenericAiPlayer.TakeDecision()` per confermare esattamente come/quando `DoDefault()` viene invocato per un giocatore AI.

- **`GenericShip.AskPerformFreeAction`**: due overload, uno a lista di azioni e uno a singola azione — quest'ultimo è solo zucchero sintattico che incapsula l'azione in una lista di un elemento e richiama l'altro overload. Entrambi confluiscono sempre nella stessa `FreeActionDecisonSubPhase`, nessuna differenza di percorso tra i due né tra AI e umano a questo livello. Il quinto parametro dell'overload a singola azione (frainteso in un primo momento come possibile flag AI/umano) è semplicemente `imageHolder`, il ritratto della nave per la UI.

## Sessione 1 — Implementazione iniziale e correzione bug interconnessi

### File caricati (con funzione)

| File | Funzione / perché è stato rilevante |
|---|---|
| `BarrelRollSubPhases.cs` | Subphase di esecuzione Barrel Roll. File più modificato della sessione. |
| `NavigationSubSystem.cs` | Cervello AI: valuta quando/come eseguire Barrel Roll. File centrale. |
| `BarrelRollAction.cs` | Entry point azione Barrel Roll. Modificato. |
| `ObstaclesFiringLineDetector.cs`, `ObstaclesHitsDetector.cs`, `ObstaclesStayDetector.cs`, `ObstaclesStayDetectorForced.cs` | Rilevamento ostacoli "reale" (trigger Unity) — consultati per confronto con l'euristica AI. |
| `GenericObstacle.cs`, `ObstaclesManager.cs` | Classe base ostacoli e relativo manager, incluso il toggle qualità collisione. |
| `ColliderDistanceInfo.cs`, `DistanceInfo.cs`, `ShipObstacleDistance.cs`, `BoardState.cs`, `IBoardObject.cs` | Utility di distanza/range consultate marginalmente. |
| `GenericShip.cs`, `GenericShipTransformations.cs` | Classe nave (partial); consultate, mai modificate. |
| `Asteroid.cs`, `Debris.cs`, `GasCloud.cs` | Tipi concreti di ostacolo — consultati marginalmente. |
| `ActionSubphase.cs`, `GenericSubPhase.cs`, `Triggers.cs` | Chiave per capire il vero significato di `CallBack()` e `FinishTrigger()`. |
| `TriggeredAbility.cs` / `TriggerForAbility.cs` | Marginali. |
| `Phases.cs`, `Actions.cs`, `ActionsRule.cs`, `GenericAction.cs` | Gestione fasi/azioni generali. `ActionsRule.cs` modificato. |
| `ManeuverTemplate.cs` | Modificato (aggiunti `SetVisible`/`IsAlive`). |
| `BoostAction.cs` | Riferimento chiave: pattern `RevertActionOnFail` e `AiPlans`/`GetPlanByActionName`. |
| `AiSelectShipPlan.cs`, `SelectShipFilter.cs` | Non pertinenti (sistema diverso). |
| `AggressorAiPlayer.cs` | Consultato e modificato. |
| `ARC170.cs`, `ARC170Starfighter.cs` | Consultate per capire perché Barrel Roll (azione rossa su ARC-170 2.0) non veniva considerato dall'AI. |
| Screenshot stack trace + foto di gioco | Diagnosi di un `NullReferenceException` e di una sovrapposizione navi. |

### File modificati (con funzione delle modifiche)

1. **`BarrelRollAction.cs`** — `ActionTake()` semplificato a puro passthrough (mirror di `BoostAction`); `RevertActionOnFail()` con `SelectedTemplate = null`.
2. **`BarrelRollSubPhases.cs`** — File più modificato: `CancelBarrelRoll()` allineato al pattern Boost; guardiani aggiunti in `PerfromTemplatePlanning()`/`ConfirmBarrelRollPosition()`; percorso AI spostato prima della registrazione trigger, legge il piano da `AiPlans.GetPlanByActionName("Barrel Roll")`.
3. **`NavigationSubSystem.cs`** — `TryBarrelRollPossibilities` riscritta: penalità bullseye applicata anche ai candidati; `GetBullseyeObstaclePenalty` riscritta con `Physics.OverlapBox`; aggiunta `IsPositionOverlappingShip`; cache statica dei template Left/Right per prestazioni.
4. **`ManeuverTemplate.cs`** — Aggiunti `SetVisible(bool)` e `IsAlive`.
5. **`ActionsRule.cs`** — Rimossa l'esclusione AI dal trigger "Stress after red action".
6. **`AggressorAiPlayer.cs`** — Eccezione per `BarrelRollAction` nel filtro azioni rosse di `PerformActionFromList`.

## Sessione 2 — Debug del controllo Bullseye e fix della priorità

### Percorso di debug (cronologia, comprese le ipotesi corrette e quelle smentite)

1. **Punto di partenza**: l'utente applica due richieste preliminari su `NavigationSubSystem.cs` — fix del bug `bestBrScore` inizializzato a `0` invece che a `int.MinValue` (impediva di selezionare candidati con punteggio negativo ma comunque migliore della baseline), e un flag sperimentale `EXCLUDE_TACTICAL_SCORE_FROM_BR` per isolare `EdgePenalty` + `BullseyeObstaclePenalty` dal punteggio totale, escludendo `GetTacticalScoreFromCurrentPosition` (applicato in modo simmetrico a baseline e candidati, per non sbilanciare il confronto).
2. **Sospetto iniziale dell'utente**: il controllo bullseye non rileva un ostacolo posizionato a distanza di una basetta, a nessuna angolazione — quindi non è un problema di rotazione.
3. **Prima ipotesi (mia, poi falsificata)**: il corridoio bullseye, lungo esattamente una basetta a partire dal bordo frontale, potrebbe non raggiungere un ostacolo posizionato esattamente al limite del range — un problema di soglia geometrica.
4. **Instrumentazione**: aggiunto un flag `DEBUG_BULLSEYE` con log diagnostici — stato del toggle `CollisionDetectionQuality`, elenco ostacoli piazzati (tag/bounds), ed esito grezzo di `Physics.OverlapBox` (hit trovati e relativi tag), poi consolidati in una riga sola per chiamata per facilitare la lettura di log lunghi (fino a 7 hit per query).
5. **Falsificazione della prima ipotesi**: i log mostrano il tag `Obstacle` correttamente rilevato sia sulla baseline sia su un candidato specifico, e correttamente assente sui candidati dove l'ostacolo non è più nel corridoio — il rilevamento geometrico funziona come progettato. Confermato anche leggendo `ObstaclesManager.cs` (caricato per la prima volta in questa sessione): il toggle qualità è sincrono e corretto.
6. **Log della decisione finale**: aggiunta una riga di log che mostra `startingScore`, `bestBrScore`, `bestBrPlanName` e l'esito. Rivela tre casi distinti nei test dell'utente:
   - Pareggio esatto (`bestBrScore == startingScore`, es. `0 == 0`): correttamente nessuna azione — comportamento voluto, non un bug.
   - `bestBrScore = int.MinValue` con `bestBrPlanName = null`: nessun candidato ha superato i filtri precedenti (bounds/ostacolo/nave) — sentinella non utilizzata nella decisione finale (corretta), ma illeggibile nel log (fixato con testo esplicativo al posto del numero grezzo).
   - **Caso critico**: `startingScore=-60`, `bestBrScore=0`, log iniziale diceva "BARREL ROLL ESEGUITO" — ma l'utente conferma che in game non succede nulla.
7. **Auto-correzione sul caso critico**: la dicitura di log era fuorviante — `TryBarrelRollPossibilities` non esegue nulla, si limita a registrare un piano in `AiPlans`. L'ipotesi dell'utente (il flag sperimentale disattiva "il trigger che fa partire l'azione") è stata esclusa per via di codice: `EXCLUDE_TACTICAL_SCORE_FROM_BR` tocca solo due variabili locali, nessun collegamento ad `AiPlans`/`actionName`/trigger.
8. **Causa reale, confermata col codice di `BarrelRollAction.cs` e un estratto di `AggressorAiPlayer.PerformActionFromList`**: `PerformActionFromList` esegue l'azione vincente **solo se** la sua priorità è `> 0`. Con `EXCLUDE_TACTICAL_SCORE_FROM_BR` attivo, `totalScore` è per costruzione `≤ 0` (somma di sole penalità), quindi il massimo teorico è esattamente `0` — mai sufficiente a superare il gate, indipendentemente da quanto sia tatticamente ovvio il Barrel Roll.
9. **Ipotesi dell'utente sul fix**: invertire il segno di `EdgePenalty`/`BullseyeObstaclePenalty` (da negativi a positivi) per farli comportare come `TacticalScore`. Verificata: corretta nella sostanza, ma un'inversione letterale del segno premierebbe le posizioni peggiori (vicine al bordo/con ostacolo) invece di penalizzarle. La trasformazione corretta è uno **spostamento additivo** (stessa costante `K` aggiunta a baseline e candidati): non altera il confronto relativo `bestBrScore > startingScore` (la costante si semplifica), ma garantisce `bestBrScore ≥ 0` quando un piano viene registrato, quindi `> 0` a valle.

### File caricati (Sessione 2)

| File | Funzione / perché è stato rilevante |
|---|---|
| `ObstaclesManager.cs` | Caricato per la prima volta; verificato il toggle `SetObstaclesCollisionDetectionQuality` — corretto, nessuna modifica necessaria. |
| `BarrelRollAction.cs` | Ricaricato per vedere `GetActionPriority()` — chiama direttamente `TryBarrelRollPossibilities`, nessuna logica intermedia. Non modificato in questa sessione. |
| Estratto di `AggressorAiPlayer.cs` (`PerformActionFromList`) | Incollato dall'utente (non l'intero file) — ha rivelato il gate `> 0`, causa reale del comportamento osservato. |
| Log Unity Console (`[BR-Bullseye-DEBUG]`, molte iterazioni) | Output di runtime, non file di progetto; analizzati passo per passo per isolare baseline, candidati e decisione finale. |

### File modificati (Sessione 2)

**`NavigationSubSystem.cs`** — unico file toccato in questa sessione:

- Fix: `bestBrScore` inizializzato a `int.MinValue` invece di `0`.
- Aggiunto `EXCLUDE_TACTICAL_SCORE_FROM_BR` (const bool, attualmente `true`): esclude `GetTacticalScoreFromCurrentPosition` dal calcolo di baseline e candidati, applicato in modo simmetrico per non sbilanciare il confronto; nessun costo prestazionale aggiuntivo quando attivo (il metodo non viene nemmeno chiamato).
- Aggiunto `DEBUG_BULLSEYE` (const bool, attualmente `true`): logga in una riga per chiamata l'elenco ostacoli piazzati, ogni query `Physics.OverlapBox` (baseline e ogni candidato, con scomposizione tactical/edge/bullseye), e la decisione finale — con dicitura corretta ("piano aggiunto ad AiPlans", non "eseguito") e valore leggibile al posto del sentinel `int.MinValue` quando non esistono candidati validi.
- Aggiunto `BR_ZERO_TACTICAL_OFFSET = 160` (const int): applicato a baseline e candidati solo quando `EXCLUDE_TACTICAL_SCORE_FROM_BR` è attivo, garantisce matematicamente che un piano registrato abbia sempre priorità `> 0`. Valore = somma delle magnitudini massime di `EdgePenalty` (100) e `BullseyeObstaclePenalty` (60) — il minimo che garantisce correttezza; non calibrato per coerenza di scala con le altre azioni (vedi punti aperti).

### Barrel Roll — fallimento silenzioso per l'AI (continuazione Sessione 2)

**Segnalazione dell'utente**: nonostante l'AI pre-validi le posizioni prima di tentare il Barrel Roll, in game a volte fallisce comunque — compare il messaggio d'errore a schermo e la nave esegue un Focus al posto del Barrel Roll pianificato.

**Decisione di design esplicita dell'utente, non un bug**: è voluto che l'AI possa tentare un'altra azione dopo un fallimento — un vantaggio deliberato rispetto al giocatore umano (per cui sarebbe illegale). Non toccato, resta così.

**Causa del messaggio visibile, confermata col codice**: in `BarrelRollSubPhases.CancelBarrelRoll()`, `ShowInformationAboutProblems()` veniva chiamato incondizionatamente, anche nel ramo AI — un residuo del percorso umano, non escluso quando fu aggiunto il bypass `RevertActionOnFail` per l'AI in Sessione 1.

**Fix applicato**: `ShowInformationAboutProblems()` spostato nel solo ramo umano (`else`). Aggiunto un log da sviluppatore nel ramo AI (flag `BR_LOG_AI_SILENT_FAILURES`, const bool, stesso pattern reversibile di `NavigationSubSystem.cs`), per non perdere visibilità sul problema di fondo — il pre-check euristico AABB (`IsPositionOverlappingObstacle`/`IsPositionOverlappingShip`) non sempre coincide col controllo reale a collider (`ObstaclesStayDetectorForced`) usato in esecuzione — pur non mostrando più nulla al giocatore.

**Non ancora indagato**: la causa profonda del mismatch euristica/controllo reale (perché una posizione pre-validata risulta comunque illegale) resta un'indagine aperta, solo resa silenziosa e loggata, non risolta alla radice.

### Reinforce — calibrazione priorità e conteggio per lato (continuazione Sessione 2)

**Richiesta iniziale**: aumentare la priorità di Reinforce (l'AI la esegue troppo poco) e cambiare il conteggio da "chi mi sta puntando" (`ActionsHolder.CountEnemiesTargeting`) a "quante navi nemiche ho geometricamente in quell'arco".

**Primo tentativo**: implementato un conteggio geometrico puro (`CountEnemyShipsByArc`, basato su `GetFrontFacing()` + `Vector3.SignedAngle`, stesso calcolo già verificato in `NavigationSubSystem.ProcessHeavyGeometryCalculations`), con riserva esplicitata prima di procedere: contare "chi è nell'arco" ignora range/linea di vista/arco di tiro, quindi è una misura di minaccia meno precisa di "chi mi sta puntando davvero".

**Ripensamento dell'utente** dopo aver letto la riserva: ripristinato `CountEnemiesTargeting` come originale, mantenendo però l'aumento di scala delle costanti (che riguardava un problema distinto e non era stato messo in discussione).

**Stato finale**: `REINFORCE_BASE_PRIORITY = 35` (era 25), `REINFORCE_PER_ENEMY_BONUS = 40` (era 30), logica di conteggio identica all'originale. Calibrazione di primo passaggio basata sul confronto con la scala massima di Boost (~70) osservata in questo codebase — non validata empiricamente in game come gli altri parametri di questa sessione.

### Coordinate + Lieutenant Sai — priorità e scelta dell'azione libera per l'AI (continuazione Sessione 2)

**Richiesta**: quando la nave pilota Lieutenant Sai, la priorità di Coordinate deve aumentare; verificare se/implementare che l'AI non debba passare per un menù per scegliere l'azione libera risultante dal Coordinate.

**Verifica preliminare**: `CoordinateAction.GetActionPriority()` e `CoordinateTargetSubPhase.GetAiCoordinatePriority` erano già implementati (scelta di usare Coordinate, e scelta di quale nave coordinare) — mancava solo il legame con Sai e la scelta dell'azione libera finale.

**Primo tentativo di implementazione (poi annullato su richiesta esplicita)**: bonus scritto dentro `CoordinateAction.GetActionPriority()`, controllando `HostShip.ShipAbilities` per `LieutenantSaiAbility`.

**Richiesta di redesign dell'utente**: il peso deve venire dal file di Lieutenant Sai, non da `CoordinateAction` — `CoordinateAction` deve restare generica, ignara di quale pilota la usa.

**Soluzione**: hook `CustomizedAi.OnGetActionPriority` (vedi Architettura) — già invocato da `AggressorAiPlayer.PerformActionFromList` per ogni azione/nave valutata, quindi zero modifiche a `CoordinateAction.cs`. `LieutenantSaiAbility` si iscrive/disiscrive nello stesso punto in cui già gestisce `OnCoordinateTargetIsSelected` (`ActivateAbility`/`DeactivateAbility`), aggiungendo `SAI_COORDINATE_BONUS = 40` quando l'azione valutata è un `CoordinateAction`. `CoordinateAction.cs` ripristinato esattamente all'originale.

**Indagine collaterale sul "menù per l'AI"**: ipotesi iniziale sbagliata (due overload di `AskPerformFreeAction` con comportamento diverso, uno "senza menù") smentita leggendo il codice reale di `GenericShip.cs` — sono equivalenti, l'overload a singola azione è solo zucchero sintattico (vedi Architettura). La causa reale è stata trovata in `FreeActionDecisonSubPhase.PrepareDecision()`: `DefaultDecisionName` era hardcoded a `"Focus"`, indipendentemente dalle azioni libere realmente disponibili o dalla loro priorità.

**Fix applicato**: `DefaultDecisionName` ora calcolato dinamicamente (`GetBestFreeActionName`), con la stessa logica di priorità (`GetActionPriority()` + `Ai.CallGetActionPriority`) già usata per l'azione principale della nave — stesso principio di `ReinforceSideSubphase` (che calcola il proprio default dinamicamente fin dalla Sessione 1), qui applicato a un caso dove mancava. Estratta anche `GetFullDecisionName` per eliminare la duplicazione tra il calcolo del default e la generazione dei pulsanti reali, evitando che potessero disallinearsi.

**Grado di certezza**: alto ma non assoluto — non è stato esaminato `GenericAiPlayer.TakeDecision()` per la conferma finale al 100% di come/quando `DoDefault()` viene invocato per un giocatore AI su questa specifica subphase.

## Punti aperti / da tenere presente in una nuova chat

- **`EXCLUDE_TACTICAL_SCORE_FROM_BR` è ancora `true`**: la modalità sperimentale (solo edge+bullseye) è tuttora attiva. Da decidere: lasciarla per continuare a validare il comportamento "difensivo puro", oppure disattivarla (`false`) ora che il rilevamento bullseye è confermato funzionante, per tornare al comportamento con tactical score incluso.
- **`DEBUG_BULLSEYE` è ancora `true`**: genera log verbosi ad ogni valutazione. Da disattivare/rimuovere a debug concluso.
- **`BR_ZERO_TACTICAL_OFFSET = 160` non ancora testato in game**: è il minimo che garantisce correttezza matematica, ma introduce una scala di priorità (fino a 160) sensibilmente più alta di quella tipica delle altre azioni (`CalculateBoostPositionPriority`, di norma sotto ~70-100). Possibile conseguenza da osservare: il Barrel Roll potrebbe vincere il confronto con le altre azioni quasi sempre quando scatta, anche per miglioramenti marginali. Se osservato, ricalibrare riducendo il valore (sapendo che sotto 160 si riapre, in misura ridotta, lo stesso problema per i casi di doppia penalità massima).
- **Caso `bestBrPlanName = null` / nessun candidato valido**: osservato nei log, non toccato dal fix dell'offset (è un problema distinto, a monte — probabilmente `IsPositionInsideBoardBounds`/`IsPositionOverlappingObstacle`/`IsPositionOverlappingShip` scartano tutti i candidati in certe configurazioni). Proposto ma non implementato: un log per il motivo esatto di scarto di ogni candidato.
- **Conferma in game ancora mancante**: i log confermano che, con l'offset, un piano registrato supera ora il gate `> 0` di `PerformActionFromList`, ma non è stata ancora osservata una nave eseguire fisicamente il Barrel Roll a schermo in questa sessione.
- **Opzione B** (calcolo analitico puro della posizione finale, senza istanziare `ManeuverTemplate`) resta non implementata (Sessione 1) — formula nota, non necessaria dopo l'ottimizzazione con cache.
- **Sovrapposizione tra navi** (`IsPositionOverlappingShip`, Sessione 1): stessa approssimazione AABB degli ostacoli, stesso limite teorico noto (falsi negativi rari su rotazioni ~45°), non ancora ritestata esplicitamente dall'utente.
- **Costo dello stress token**: l'AI non lo pesa ancora contro il beneficio tattico nelle azioni rosse (Rotate Arc, Barrel Roll) — obiettivo a lungo termine, non affrontato.
- **Fix del messaggio d'errore Barrel Roll per l'AI** (`BR_LOG_AI_SILENT_FAILURES`): implementato ma non ancora ritestato in game dall'utente. Se il log `[BarrelRoll-AI]` continua a comparire spesso, conferma che il mismatch euristica/controllo reale è un problema ricorrente da affrontare alla radice, non solo da silenziare.
- **Reinforce**: valori `REINFORCE_BASE_PRIORITY = 35` / `REINFORCE_PER_ENEMY_BONUS = 40` non ancora validati in game.
- **Lieutenant Sai**: bonus `SAI_COORDINATE_BONUS = 40` su Coordinate non ancora validato in game.
- **`FreeActionDecisonSubPhase`**: fix del `DefaultDecisionName` dinamico non ancora validato in game. Se l'AI continua a scegliere sempre "Focus" dopo un'azione libera, il prossimo file da esaminare è `GenericAiPlayer.cs` (in particolare `TakeDecision()`), mai visto per intero in questa sessione.
- **`ActionDecisonSubPhase`** ha ancora lo stesso `DefaultDecisionName = "Focus"` hardcoded di `FreeActionDecisonSubPhase` prima del fix, non toccato: per ragionamento (non conferma diretta) dovrebbe essere reso irrilevante da `PerformActionFromList`, che sceglie l'azione principale per un altro percorso — ma non è stato verificato con certezza.
- Percorsi log Unity per debug futuro: `Editor.log` (Console → "Open Editor Log", o `%LOCALAPPDATA%\Unity\Editor\Editor.log` su Windows / `~/Library/Logs/Unity/Editor.log` su macOS) o `Player.log` (build compilate, `%USERPROFILE%\AppData\LocalLow\<Azienda>\<Progetto>\Player.log` su Windows).
