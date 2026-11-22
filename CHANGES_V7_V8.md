# CHANGES_V7_V8.md – rozdíl mezi v7.3-baseline a v8-team

**Baseline:** tag `v7.3-baseline` (`936d5f9`)
**Větev:** `v8-team` (`b0bb6df`)
**Datum:** 2026-11-22

> **POZOR:** tento dokument popisuje **stav kódu s criticem (v8.0)**. Návrh v8.3 critic odstraňuje
> (searcheři + deterministický coordinator); kód bude aktualizován po schválení designu.
> Aktuální design je v `DESIGN_TEAM_V8.md`.

## 1. Shrnutí

| | v7.3-baseline | v8-team |
|---|---|---|
rolí s LLM | 1 (`manager`) | 2 (`proposer`, `critic`) |
uzlů na tah | 5 | 6 (`select → manager → critic → act → observe → gate`) |
rozhoduje o akci | model | model, ale **critic může vetovat** |
veto | – | 1 tah, 0 kyslíku, `critic_veto` |
chyba criticа | – | fail-open (`critic_error_count`) |
seed do Ollamy | ne | **ano** (`ChatOllama(seed=…)`) |
filtr A+L | není | je, ale v team configu `filter_mode: off` |
notebooky / PNG | v repu | odstraněny z větve |

## 2. Změny po souborech

```
DESIGN_TEAM_V8.md            | 130 ++   nový návrh v8 (2 role, veto účtování)
app/agents/graph.py          |  11 +-   +critic node, manager→critic, conditional critic→{act,observe}
app/agents/nodes.py          | 155 ++   critic_node, _build_critic_human, after_critic
app/agents/prompts.py        |  39 +    CRITIC_SYSTEM, CRITIC_HUMAN_TEMPLATE
app/api/main.py              |  35 +-   critic LLM z configu, manifest mode/critic_*
app/core/compiler.py         |  34 +    neutrální filtr A+L (v týmu off)
app/core/models.py           |  15 +-   CriticVerdict, critic_veto, stavová pole
configs/team_gemma_qwen.yaml |  17 +    proposer gemma3:12b + critic qwen2.5-coder:7b
tests/test_filter.py         |  65 +    testy filtru (4)
tests/test_team.py           | 124 +    testy týmu: veto, fail-open, pass (3)
escape_room_v7-*.ipynb       | -6635   odstraněno z v8
escape_room_v7*.png          |  -2     odstraněno z v8
```

## 3. Co zůstalo stejné

- `app/core/engine.py` a všechny scénáře v `app/scenarios/` – **beze změny**, takže srovnání
  s baseline je platné (`scenario_sha256` v manifestu to dokládá).
- `compiler` pravidla: location check, kontrola nástroje v inventáři, cap `COMMUNICATE`.
- stagnation guard (`terminate` / `log_only`), manifest, UUID `thread_id`, log i při výjimce.
- 7 univerzálních akcí. `COMMUNICATE` je stále v lexikonu, ale tým ho nepoužívá.

## 4. Komunikace – co je jinak

- **v7:** `COMMUNICATE` zapisoval do `messages`, ale s jedním agentem neměl příjemce;
  `messages` zůstávalo prázdné.
- **v8:** komunikace jde **výhradně přes blackboard**:
  - `critic` dostane jen jeden návrh + `current_room`, `inventory`, viditelné objekty a selhané akce;
  - `proposer` dostane `tried_actions` a stav světa jako dosud;
  - `critic` nesmí navrhnout akci – jeho schéma má jen `{veto, reason}`;
  - sdílený kanál nejsou volné texty, ale `tried_actions` a `decision_log` (faktické záznamy).
- Důvod: volný text mezi agenty se hromadí v promptech a nedá se měřit. Blackboard ano.
- **Plánovaný další krok** (není v této větvi): `facts` jako strukturovaný kanál
  `{subject, attribute, value, source_role, turn}`.

## 5. Deterministické a stochastické části

| část | deterministická? | poznámka |
|---|---|---|
engine, scénář, secrets | ano | `seed` řídí všechny náhody světa |
router (`select`, `after_critic`, `gate`) | ano | čistá funkce stavu |
filtr A+L, guard | ano | pravidla, žádný model |
`critic` | při `temperature: 0` prakticky ano | + `seed` do Ollamy; ale viz níže |
`proposer` | ne | `temperature: 0.3`; `seed` dělá běh **reprodukovatelným**, ne deterministickým napříč vším |

Co reprodukovatelnost rozbije i se seedem:
- jiný digest modelu (Ollama `latest` se může změnit),
- jiná verze Ollamy / llama.cpp,
- jiný `num_ctx`, `num_predict`, prompt nebo commit (proto manifest),
- GPU offload a batching mohou změnit numeriku.

Ověření: dva běhy se stejným `seed` + stejným manifestem musí dát stejnou sekvenci akcí.
To je akceptační test, který ještě není změřený (čeká na doběhnutí baseline).

## 6. Jak spustit

```bash
# v8 tým
ESCAPE_CONFIG=configs/team_gemma_qwen.yaml \
  uvicorn app.api.main:app --port 8011

curl -X POST localhost:8011/run -H 'Content-Type: application/json' -d '{
  "scenario": "medium_room", "seed": 42, "temperature": 0.3,
  "guard_mode": "log_only", "filter_mode": "off"
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
