# DESIGN_TEAM_V8.md – hledači a koordinátor pozornosti (v8.3, revize po review)

**Status:** draft, neimplementováno
**Autor:** Opičák (Lead Coder, MGD)
**Určeno k review:** Marta (Arbiter), Gemini, Kimi, Claude
**Historie:** v8.0 (Proposer + Critic s vetem) zamítnuto – veto bere iniciativu a kazí uvažování.
v8.2 nahradilo critic blackboardem. **v8.3 zapracovává review** (Kimi, Gemini, Opičák) –
viz sekce 13. Baseline zůstává `v7.3-baseline`.

---

## 1. Cíl a hypotéza

Ověřit, zda **skupina hledačů se sdíleným blackboardem a koordinátorem pozornosti** dosáhne
vyššího postupu než single-agent se stejným modelem a než stejný model s více vzorky (sampling control).

- **Metriky se reportují odděleně, nikdy jako součet:**
  (a) `clean_escape` a `escape`, (b) `milestones_reached / total`, (c) `guess_escape_rate`
  (escape ∧ ¬clean), (d) náklady: `turns`, `wall_time_s`, `llm_calls_total`, `coordinator_wall_s`.
- **Rozsah:** jedna postava, o kterou se dělí více agentů. Kooperace více postav (`coop_min`)
  je samostatná větev a tento dokument ji neřeší.

## 2. Anti-cíle (co záměrně neděláme)

1. **Žádné veto.** Veto bere iniciativu a mění rozhodování (viz literatura o sykofancii).
2. **Žádné rady ani nápovědy v promptech.** Agent dostane pozorování, stav a případně direktivu
   „kam se dívat“ – nikdy „co udělat“.
3. **Koordinátor nevydává akce**, jen alokuje pozornost.
4. **Žádné volné texty na blackboardu** – jen strukturovaná fakta odvozená z enginu.
5. **Žádná paralelizace v první verzi** (VRAM a plánování Ollamy = confound).
6. Neměníme scénář ani engine, jinak nelze srovnávat s baseline.
7. Žádná vlastní paměť agentů; jediný zdroj pravdy je blackboard.

## 3. Architektura – klasický blackboard pattern

Mapování na Hearsay-II (viz literatura, sekce 12):

| komponenta | u nás |
|---|---|
blackboard (strukturovaná globální paměť) | `facts` + `tried_actions` + `decision_log` |
knowledge sources (specializované moduly) | 2–3 **searchers** (LLM agenti) |
control component (scheduler relevancí) | **coordinator** – výběr „kam se dívat“, ne „co udělat“ |

Kontrolní komponenta v klasickém pojetí **hodnotí relevanci**, nediktuje akce. To je hranice, kterou držíme.


## 4. Role

| role | počet | akce ve světě | výstup |
|---|---|---|---|
`searcher` | 2–3 | **7 univerzálních akcí jako baseline**: `LOOK, INSPECT, GO, MOVE, PICK_UP, USE_ITEM, COMMUNICATE` | herní akce + fakta, která z ní vyplynula |
`coordinator` | 1 | **žádné** | direktiva „kam se dívat“ (`focus_kind`, `focus_id`), nebo „bez direktivy“ |

- **Akční prostor searcherů je shodný s baseline.** `COMMUNICATE` zůstává povolený, ale tým ho
  nepoužívá jako kanál (fakta jdou na blackboard). `ENTER_PIN`/`ENTER_CODE` v enginu neexistují –
  jsou to `USE_ITEM` s `requires_value`.
- **Coordinator v1 je deterministický** (žádný model): z faktů vybere místnost/objekt s nejmenším
  pokrytím. LLM coordinator je **samostatné rameno v8.3**, aby šlo změřit jeho přínos nad heuristikou.
- Searchers jsou různé modely (jiná rodina = jiné chyby): `gemma3:12b`, `qwen2.5-coder:7b`,
  případně `mistral:7b`. Kontrolní rameno **C0** = 3× tentýž model (odděluje strukturu od diverzity).


## 5. Fakta na blackboardu

```python
Fact = {
  "subject": str,      # object_id
  "attribute": str,    # location | state | takeable | in_inventory
  "value": str,
  "source_agent": str,
  "turn": int,
  "room": str,
}
```

- Fakt vzniká **odvozením z enginu** po akci (`revealed_objects`, `object_states`, `inventory`),
  nikdy z LLM shrnutí. Tím mizí halucinace i „kecání“.
- **Write-on-change:** fakt se zapisuje jen při změně trojice `(subject, attribute, value)`.
  Opakovaný zápis stejné trojice je `duplicate_fact` a jde do metriky, ne do blackboardu.
- Fakt je **immutable**; nová hodnota vytvoří nový fakt s novým `turn` a `source_agent`.
- Agent smí zapsat jen fakta o tom, co skutečně viděl (byl v místnosti nebo měl objekt v inventáři).
- Coordinator fakta neupravuje, jen čte.
- `fact_coverage = unikátní (subject, attribute) / možná` – jmenovatel ze scénáře
  (`medium_room` ~30 slotů). Délkově imunní náhrada za `new_facts_per_turn`.


## 6. Fáze a účtování

1. `search` – searcher provede 1 akci; z odpovědi enginu se zapíšou fakta.
2. `coordinate` – po každých K tazích (návrh K=3) coordinator dostane fakta a vydá direktivu.
3. zpět na `search`.

- **Sdílený kyslík a sdílený `turn_budget`** pro celý tým (fér vůči baseline).
- Každý tah = právě jedna herní akce jednoho agenta; `turn` unikátní a rostoucí.
- Direktiva je metadata téhož tahu, **ne** nový tah a **ne** spotřeba kyslíku.
- Střídání searcherů: round-robin (deterministické). Jiný výběr je otevřená otázka č. 4.


## 7. Coordinator – deterministický scheduler (v1)

**Vstupy (explicitně, nic jiného):**
- `facts` (`subject, attribute, value, source_agent, turn, room`),
- historie akcí jako `(turn, agent, verb, target, result)`,
- **ne** `rationale` ze `SemanticAction`, **ne** surové observace – to jsou zadní vrátka pro rady.

**Algoritmus v1 (deterministický, bez modelu):**
1. Objekt v aktuální místnosti bez `INSPECT` faktu → `focus_kind=object`.
2. Místnost, kterou ještě nikdo nenavštívil → `focus_kind=room`.
3. `(subject, attribute)` slot s nejstarším faktem → `focus_kind=object`.
4. Jinak `focus_kind=none`.

**Výstup:**
```json
{"focus_kind": "room" | "object" | "none", "focus_id": "chamber"}
```
Do logu jde jen `reason_code` (enum), ne volný text. `focus_id` se do promptu searchera
dostane jako „kam se dívat“, nikdy jako verb ani hodnota.

**LLM coordinator** je samostatné rameno v8.3 – měří se, zda přinese něco nad heuristikou.

**Filtr (deterministický) zůstává** jako součást harness ve **všech** ramenech: formální platnost,
dosažitelnost cíle, nástroj v inventáři, no-op, stabilní opakování. Není to rada ani veto –
je to součást pravidel světa. Zapnutý i v baseline, jinak srovnání neplatí.


## 8. Metriky a mapování na MAST

| metrika | definice | MAST mód |
|---|---|---|
`clean_escape` | `escaped` ∧ `INSPECT laser_emitter` proběhl před `color_pad` | FM-3.1 |
`escape` | jen flag `escaped` (může být uhodnutím) | – |
`guess_escape_rate` | `escape` ∧ ¬`clean_escape` | – |
`milestones_reached/total` | postup, u `medium_room` 0–3 | – |
`fact_coverage` | unikátní `(subject, attribute)` / možné sloty scénáře | – |
`duplicate_fact_rate` | duplicitní pokusy o zápis / všechny pokusy o zápis | FM-1.3 |
`facts_per_agent` | fakta / vlastní tahy agenta (normalizováno na příležitost) | FM-2.4 |
`directive_followed` | první akce do 2 tahů po direktivě má `target == focus_id`, nebo `GO` do místnosti s `focus_id` | FM-2.5 |
`repeat_after_fact` | akce zopakovaná, i když o cíli existuje fakt | FM-1.3 |
`llm_calls_total` | počet volání LLM za běh (searcheři + coordinator) | – |
`coordinator_wall_s` | čas spotřebovaný koordinací | – |
`turns`, `wall_time_s`, `oxygen_left` | náklady | FM-1.5 |

Manifest zůstává: `git_commit`, `scenario_sha256`, `config_sha256`, `model_digest`,
`guard_mode`, `filter_mode`, `recursion_limit`, `log_status`.
`guard_mode: log_only` pro všechna experimentální ramena; `would_terminate` se reportuje zvlášť.



## 9. Implementační stopa

| soubor | změna | odhad |
|---|---|---|
`app/core/models.py` | `Fact`, `Directive`, stav `facts`/`directive`, `Literal["coordinate"]` | ~40 řádků |
`app/core/blackboard.py` | zápis faktů (write-on-change), dedup, `fact_coverage` | ~60 řádků |
`app/core/coordinator.py` | **deterministická heuristika** (bez modelu) | ~50 řádků |
`app/agents/nodes.py` | `searcher_node` (dnes manager), `coordinator_node`, router | ~80 řádků |
`app/agents/prompts.py` | searcher prompt bez rad; direktiva jako „kam se dívat“ | ~30 řádků |
`app/agents/graph.py` | `select → searcher → act → observe → (coordinator každé K) → gate` | ~30 řádků |
`app/api/main.py` | `team_mode`, manifest (`coordinator: deterministic`), nákladové metriky | ~30 řádků |
`configs/team_*.yaml` | searchers, `K=3`, `coordinator: deterministic` | ~20 řádků |

**Odstranit:** `critic_node`, `after_critic`, `CriticVerdict`, `critic_veto`, `critic_*` metriky.
**Přepsat:** `tests/test_team.py` na fakta + direktivu.
**Gate před během:** akceptační test reprodukovatelnosti (2 běhy, stejný seed + manifest ⇒ stejná sekvence akcí).
Odhad ~340 řádků, z toho ~170 nových.

## 10. Rizika

1. **Coordinator jako skrytý řešitel** – v1 deterministický, výstup bez verba a bez volného textu;
   `focus_id` se do promptu dostane jen jako „kam se dívat“.
2. **`rationale` jako zadní vrátka** – coordinator nesmí číst `rationale` ani surové observace.
3. **Duplikace hledání** – `duplicate_fact_rate`, `fact_coverage`.
4. **Sampling confound** – tým musí porazit (a) single-agent a (b) sampling control; navíc rameno
   **C0** = 3× tentýž model, aby se oddělila struktura od diverzity modelů.
5. **Sykofancie** – jakákoli rada v promptu mění rozhodování; proto jen fakta a direktiva „kam“.
6. **Ztráta kontextu** – fakta persistentní s `turn`; jinak FM-1.4 / FM-2.1.
7. **Koordinační režie** – LLM coordinator by přidal 20 callů na běh; v1 je zdarma, LLM jen jako rameno.
8. **Účtování** – direktiva nesmí spotřebovat tah; test invariantu „jeden záznam na tah“.
9. **Timeout na rameno** – v předchozím experimentu zabil seed 123; timeout musí být na běh.
10. **Reprodukovatelnost** – bez prošlého akceptačního testu se experiment nespouští.


## 11. Rozhodnuto po review a co zůstává otevřené

**Rozhodnuto (v8.3):**
1. **Coordinator v1 deterministický**; LLM coordinator = samostatné rameno v8.3.
2. **K = 3 fixní** – proměnný počet callů je nákladový confound.
3. **Searcheři vidí celý blackboard.** Duplicita se měří (`duplicate_fact_rate`), nezakazuje.
4. **Duplicita se řeší write-on-change**, ne direktivou „tuhle místnost už máme“.
5. **Round-robin** střídání; výběr podle neznámých objektů je až varianta.
6. **`laser_color` zůstává 3 možnosti** (srovnatelnost s baseline); `guess_escape_rate` to zviditelní.
7. **Akční prostor shodný s baseline** – 7 akcí včetně `COMMUNICATE`.

**Otevřené (k review):**
1. Mají searcheři vidět i **cizí `tried_actions`** (kdo co zkoušel), nebo jen `facts`?
2. Má být **K závislé na scénáři** (2 pro malé, 4 pro velké), nebo vždy 3?


## 12. Literatura

- Blackboard pattern: [Wikipedia](https://en.wikipedia.org/wiki/Blackboard_(design_pattern)) –
  tři komponenty, control = scheduler relevancí.
- [CrewAI – Processes](https://docs.crewai.com/en/concepts/processes) – hierarchical manager
  deleguje úkoly, ne konkrétní akce.
- [Why Do Multi-Agent LLM Systems Fail? (MAST)](https://arxiv.org/abs/2503.13657) –
  14 failure modes ve 3 kategoriích, zisky MAS jsou často minimální.
- [More Agents Is All You Need](https://arxiv.org/abs/2402.05120) – zisky z počtu agentů
  jsou hlavně sampling-and-voting.
- [Towards Understanding Sycophancy](https://arxiv.org/abs/2310.13548) – model mění odpovědi
  podle naznačeného přesvědčení, i když je špatné.

## 13. Zapracované review (v8.3)

| zdroj | podnět | rozhodnutí |
|---|---|---|
Kimi | akční prostor neodpovídá baseline (chybí `COMMUNICATE`, `ENTER_PIN` neexistuje) | **přijato** – §4 opraveno |
Kimi | write-on-change + `fact_coverage` místo `new_facts_per_turn` | **přijato** – §5, §8 |
Kimi | coordinator v1 deterministický, LLM jen jako rameno | **přijato** – §4, §7, §11 |
Kimi | `rationale` je zadní vrátka pro rady | **přijato** – §7 (explicitní vstupy) |
Kimi | chybí rameno C0 (3× stejný model) | **přijato** – §4, §10 |
Kimi | akceptační test reprodukovatelnosti jako gate; timeout na běh | **přijato** – §9, §10 |
Kimi | definice `directive_followed`, `guess_escape_rate`, nákladové sloupce | **přijato** – §8 |
Kimi | diagram ukazuje starou v8.0 s criticem | **přijato** – puml přepsán |
Gemini | nemíchat milníky a `clean_escape` do jedné metriky | **přijato** – §1, §8 |
Gemini | rozhodovací pravidlo předem; malý vzorek = pilot | **přijato** – §14 |
Gemini | omezit `reason` coordinatora na neutrální | **přijato** – `reason_code` enum |
Opičák | coordinator je pořád řízení | **částečně** – v1 deterministický, LLM jen rameno |
Opičák | searcheři mají vidět jen svá fakta | **odmítnuto** – sdílený blackboard je pointa; měříme duplicitu |
Opičák | `laser_color` 6–8 možností | **odmítnuto** – rozbilo by srovnatelnost; `guess_escape_rate` |
Claude | poradní kritik / veto-streak guard | **neaktuální** – critic z designu odstraněn |

## 14. Předregistrované vyhodnocení

Zapsáno **před** prvními běhy, aby nešlo pravidla doladit podle výsledku.

- **Rameno:** A (single agent), B (sampling control: N vzorků + majorita), C1 (tým různých modelů),
  C0 (3× tentýž model), D (single thinkingcap jako strop).
- **Seedy:** 10 společných seedů napříč rameny (párový design), `temperature 0.3`, `turn_budget 60`.
- **Primární metriky:** `clean_escape` (McNemar) a `milestones_reached/total` (Wilcoxon signed-rank).
- **Sekundární:** `fact_coverage`, `duplicate_fact_rate`, `turns`, `llm_calls_total`, `wall_time_s`.
- **Rozhodovací pravidlo:** „tým vyhrál“ ⟺ párový medián Δ`milestones` > 0 **a** `clean_escape`
  není horší (margin 0) **a** C1 > B. Jinak **neprůkazné**.
- **Zákaz průběžného nahlížení** a předčasného ukončení; smoke testy se do vyhodnocení nepočítají.



