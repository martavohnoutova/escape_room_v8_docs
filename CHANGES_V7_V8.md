# CHANGES_V7_V8.md – rozdíl mezi v7.3-baseline a v8-team

**Baseline:** tag `v7.3-baseline` (`936d5f9`)
**Větev:** `v8-team` (`b0bb6df`)
**Datum:** 2026-11-22

> **POZOR:** tento dokument popisuje **stav kódu s criticem (v8.0)**. Návrh v8.3 critic odstraňuje
> (searcheři + deterministický coordinator); kód bude aktualizován po schválení designu.
> Aktuální design je v `DESIGN_TEAM_V8.md`.

## 1. Shrnutí

| | v7.3-baseline | v8-team (v8.3) |
|---|---|---|
rolí s LLM | 1 (`manager`) | 2–3 **searcheři** (round-robin) |
koordinace | – | **deterministický coordinator** každé K tahy |
uzlů na tah | 5 | 6 (`select → manager → act → observe → coordinator? → gate`) |
rozhoduje o akci | model | model; coordinator jen **kam se dívat** |
sdílená paměť | blackboard bez faktů | **fakta** `(subject, attribute, value)`, write-on-change |
veto / rady | – | **žádné** (critic odstraněn) |
seed do Ollamy | ne | **ano** (`ChatOllama(seed=…)`) |
filtr A+L | není | je, `filter_mode: on` ve všech ramenech |
notebooky / PNG | v repu | odstraněny z větve |

## 2. Změny po souborech (shrnutí)

**Nové:**
- `app/core/coordinator.py` – deterministická heuristika výběru pozornosti (bez modelu)
- `tests/test_team.py` – 6 testů: write-on-change, fact coverage, coordinator, round-robin

**Změněné:**
- `app/core/models.py` – `Fact`, `Directive`, `coordinate`, stavová pole
- `app/core/blackboard.py` – zápis faktů, `fact_coverage`, `record_facts_from_snapshot`
- `app/agents/nodes.py` – fakta v observe, coordinator node, round-robin výběr searchera
- `app/agents/prompts.py` – `[COORDINATOR FOCUS]` a `[SHARED FACTS]`, critic odstraněn
- `app/agents/graph.py` – `observe → coordinator? → gate`
- `app/api/main.py` – `searchers[]`, `coordinator_k`, manifest bez critic polí
- `configs/team_gemma_qwen.yaml` – dva searcheři, `coordinator_k: 3`, filtr `on`

**Odstraněné z větve:** `escape_room_v7-*.ipynb`, `escape_room_v7*.png`.

Přesná čísla: `git diff --stat v7.3-baseline..v8-team`.


## 3. Co zůstalo stejné

- `app/core/engine.py` a všechny scénáře v `app/scenarios/` – **beze změny**, takže srovnání
  s baseline je platné (`scenario_sha256` v manifestu to dokládá).
- `compiler` pravidla: location check, kontrola nástroje v inventáři, cap `COMMUNICATE`.
- stagnation guard (`terminate` / `log_only`), manifest, UUID `thread_id`, log i při výjimce.
- 7 univerzálních akcí. `COMMUNICATE` je stále v lexikonu, ale tým ho nepoužívá.

## 4. Komunikace – co je jinak

- **v7:** `COMMUNICATE` zapisoval do `messages`, ale s jedním agentem neměl příjemce;
  `messages` zůstávalo prázdné.
- **v8.3:** komunikace jde **výhradně přes blackboard**:
  - searcheři zapisují **fakta odvozená z enginu**
    (`subject, attribute, value, source_agent, turn, room`),
  - každý searcher vidí celý blackboard – sdílená paměť je pointa,
  - coordinator vydává jen direktivu `{focus_kind, focus_id}` – „kam se dívat“,
  - žádné rady, žádné veto, žádný volný text mezi agenty.
- Duplicity se nezakazují, ale **nezapisují** (write-on-change) a **měří** (`duplicate_fact_rate`).
- Důvod: volný text mezi agenty se hromadí v promptech a nedá se měřit. Strukturovaná fakta ano.

## 5. Deterministické a stochastické části

| část | deterministická? | poznámka |
|---|---|---|
engine, scénář, secrets | ano | `seed` řídí všechny náhody světa |
router (`select`, `after_observe`, `gate`) | ano | čistá funkce stavu |
**coordinator** | **ano** | deterministická heuristika, žádný model |
filtr A+L, guard | ano | pravidla, žádný model |
searcheři | ne | `temperature: 0.3`; `seed` dělá běh **reprodukovatelným**, ne deterministickým napříč vším |

Co reprodukovatelnost rozbije i se seedem:
- jiný digest modelu (Ollama `latest` se může změnit),
- jiná verze Ollamy / llama.cpp,
- jiný `num_ctx`, `num_predict`, prompt nebo commit (proto manifest),
- GPU offload a batching mohou změnit numeriku.

**Ověření (22. 11. 2026, akceptační gate):** dva běhy se stejným `seed 42` a stejným manifestem daly
**identickou sekvenci** (`PICK_UP flashlight`, `INSPECT dark_corner`), stejný počet faktů (8),
stejnou `fact_coverage` (0.2222) a stejné duplicity (7). **Gate prošel.**

## 6. Jak spustit

```bash
# v8 tým
ESCAPE_CONFIG=configs/team_gemma_qwen.yaml \
  uvicorn app.api.main:app --port 8011

curl -X POST localhost:8011/run -H 'Content-Type: application/json' -d '{
  "scenario": "medium_room", "seed": 42, "temperature": 0.3,
  "guard_mode": "log_only", "filter_mode": "on"
}'
```

Single-agent v8 zůstává funkční: config **bez** `critic_model` (např. `configs/single_gemma.yaml`)
znamená `mode: single` a uzel critic se chová jako průchod.

## 7. Metriky navíc

`manifest.mode` (`single`/`team`), `manifest.critic_model`, `manifest.critic_vetoes`,
`manifest.critic_errors`. V `decision_log` přibyl typ `critic_veto`.

## 8. Co tento dokument netvrdí

Netvrdí, že tým je lepší než single-agent. To je hypotéza, kterou teprve měříme proti
`v7.3-baseline`, a to se stejným scénářem, teplotou a seedy.
