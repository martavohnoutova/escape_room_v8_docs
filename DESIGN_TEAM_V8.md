# DESIGN_TEAM_V8.md – hledači a koordinátor pozornosti (v8.2, návrh k review)

**Status:** draft, neimplementováno
**Autor:** Opičák (Lead Coder, MGD)
**Určeno k review:** Marta (Arbiter), Gemini, Kimi, Claude
**Historie:** v8.0 (Proposer + Critic s vetem) zamítnuto – veto bere iniciativu a kazí uvažování.
Nahrazeno blackboard architekturou. Baseline zůstává `v7.3-baseline`.

---

## 1. Cíl a hypotéza

Ověřit, zda **skupina hledačů se sdíleným blackboardem a koordinátorem pozornosti** dosáhne
vyššího postupu než single-agent se stejným modelem a než stejný model s více vzorky (sampling control).

- **Primární metrika:** postup `milestones_reached / total` (u `medium_room` 0–3) + `clean_escape`.
- **Sekundární:** tahy, kyslík, wall-clock, `new_facts_per_turn`, `duplicate_fact_rate`.
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
`searcher` | 2–3 | plný slovník (LOOK, INSPECT, GO, PICK_UP, USE_ITEM, MOVE) | herní akce + fakta, která z ní vyplynula |
`coordinator` | 1 | **žádné** | direktiva „kam se dívat“ (místnost / objekt), nebo „bez direktivy“ |

Searchers jsou **různé modely** (jiná rodina = jiné chyby): návrh `gemma3:12b`, `qwen2.5-coder:7b`,
případně `mistral:7b`. Coordinator dostane vlastní instanci (návrh `gemma3:12b`, `temperature 0.0`),
aby četl fakta nezaujatě – viz otevřená otázka č. 1.

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
- Fakt je **immutable**; nová hodnota vytvoří nový fakt s novým `turn` a `source_agent`.
- Agent smí zapsat jen fakta o tom, co skutečně viděl (byl v místnosti nebo měl objekt v inventáři).
- Coordinator fakta neupravuje, jen čte.

## 6. Fáze a účtování

1. `search` – searcher provede 1 akci; z odpovědi enginu se zapíšou fakta.
2. `coordinate` – po každých K tazích (návrh K=3) coordinator dostane fakta a vydá direktivu.
3. zpět na `search`.

- **Sdílený kyslík a sdílený `turn_budget`** pro celý tým (fér vůči baseline).
- Každý tah = právě jedna herní akce jednoho agenta; `turn` unikátní a rostoucí.
- Direktiva je metadata téhož tahu, **ne** nový tah a **ne** spotřeba kyslíku.
- Střídání searcherů: round-robin (deterministické). Jiný výběr je otevřená otázka č. 4.


## 7. Coordinator – co smí a co ne

**Smí:**
- číst fakta a historii akcí,
- vybrat místnost nebo objekt, který je málo prozkoumaný (fakta chybí nebo jsou stará),
- říct „bez direktivy“ (nechat searchery jít podle sebe),
- označit objekt jako neznámý (ještě nebyl `INSPECT`ován).

**Nesmí:**
- vybírat verb ani hodnotu (`USE_ITEM … value=…`),
- doporučovat postup ani pojmenovávat řešení,
- používat znalost milníků nebo scénáře.

Výstup (structured output):
```json
{"focus_kind": "room" | "object" | "none", "focus_id": "chamber", "reason": "no facts yet"}
```
`reason` jde do logu, ne do promptu searchera.

**Filtr (deterministický) zůstává** jako součást harness ve **všech** ramenech: formální platnost,
dosažitelnost cíle, nástroj v inventáři, no-op, stabilní opakování. Není to rada ani veto –
je to součást pravidel světa. Zapnutý i v baseline, jinak srovnání neplatí.

## 8. Metriky a mapování na MAST

| metrika | co měří | MAST mód |
|---|---|---|
`milestones_reached/total` | postup | – |
`clean_escape` | escape ∧ emitter inspectnutý | FM-3.1 |
`new_facts_per_turn` | užitečnost hledání | – |
`duplicate_fact_rate` | plýtvání hledáním | FM-1.3 step repetition |
`facts_per_agent` | kdo přispívá | FM-2.4 information withholding |
`directive_followed` | poslušnost direktivy | FM-2.5 ignored input |
`repeat_after_fact` | agent opakuje, i když fakt existuje | FM-1.3 |
`turns`, `wall_time_s`, `oxygen_left` | náklady | FM-1.5 |

Manifest zůstává: `git_commit`, `scenario_sha256`, `config_sha256`, `model_digest`,
`guard_mode`, `filter_mode`, `recursion_limit`, `log_status`.
`guard_mode: log_only` pro všechna experimentální ramena; `would_terminate` se reportuje zvlášť.


## 9. Implementační stopa

| soubor | změna | odhad |
|---|---|---|
`app/core/models.py` | `Fact`, `Directive`, stav `facts`/`directive`, `Literal["coordinate"]` | ~40 řádků |
`app/core/blackboard.py` | zápis a čtení faktů, deduplikace, počítadla | ~60 řádků |
`app/agents/nodes.py` | `searcher_node` (dnes manager), `coordinator_node`, router | ~100 řádků |
`app/agents/prompts.py` | searcher prompt bez rad; coordinator prompt s direktivou | ~40 řádků |
`app/agents/graph.py` | `select → searcher → act → observe → (coordinator?) → gate` | ~30 řádků |
`app/api/main.py` | `team_mode`, `coordinator_model`, manifest | ~30 řádků |
`configs/team_*.yaml` | searchers + coordinator | ~20 řádků |

**Odstranit:** `critic_node`, `after_critic`, `CriticVerdict`, `critic_veto`, `critic_*` metriky.
**Přepsat:** `tests/test_team.py` na fakta + direktivu.
Odhad ~320 řádků, z toho ~150 nových.

## 10. Rizika

1. **Coordinator jako skrytý řešitel** – hlídáno schématem (`focus_kind`/`focus_id` bez verba)
   a testem, že direktiva neobsahuje akci.
2. **Duplikace hledání** – `duplicate_fact_rate`; pokud tým nepřinese zlepšení, je to negativní výsledek.
3. **Sampling confound** – tým musí porazit stejný model s N vzorky a majoritou (More Agents).
4. **Sykofancie** – jakákoli rada v promptu mění rozhodování; proto jen fakta a direktiva „kam“.
5. **Ztráta kontextu** – fakta persistentní s `turn`; jinak FM-1.4 / FM-2.1.
6. **Koordinační režie** – jeden LLM call navíc každé K tahů; měřit wall-clock.
7. **Účtování** – direktiva nesmí spotřebovat tah; test invariantu „jeden záznam na tah“.

## 11. Otevřené otázky k review

1. Má mít coordinator **vlastní instanci** modelu, nebo má být jeden ze searcherů povýšen?
2. **K = 3** tahy mezi direktivami, nebo direktiva vždy, když se objeví nový fakt?
3. Mají searcheři vidět **cizí fakta** (celý blackboard), nebo jen svá? (Riziko: opisování cizích chyb.)
4. **Round-robin** střídání, nebo výběr podle „kdo má nejvíc neznámých objektů“?
5. Jak řešit duplicitu: jen metrikou, nebo i direktivou „tuhle místnost už máme“?

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


