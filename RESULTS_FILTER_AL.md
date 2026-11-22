# RESULTS_FILTER_AL.md – uzavření ramene A vs A+L

**Datum:** 2026-11-22
**Kód:** `main` (filtr `7190fbe`), baseline na tagu `v7.3-baseline` (`936d5f9`)
**Scénář:** `medium_room`, `turn_budget 60`, `temperature 0.3`, `guard_mode log_only`
**Model:** `gemma3:12b`

## Účel

Změřit, zda neutrální filtr (L1 stabilní opakování, L2 no-op pick-up) zlepší výsledek
single-agenta, nebo jen překategorizuje neplatné akce. Šlo o kontrolní rameno před v8:
pokud by filtr sám zlepšoval úspěšnost, nešlo by zlepšení týmu připsat spolupráci.

## Výsledky

| rameno | seed | tahů | O₂ | úspěšné | pass | compile | filter_block | coverage | wall |
|---|---|---|---|---|---|---|---|---|---|
A (off) | 42 | 60 | 26 | 12 | 2 | 46 | 0 | 0.67 | 1175 s |
A (off) | 99 | 60 | 33 | 6 | 1 | 53 | 0 | 0.33 | 1297 s |
A+L (on) | 42 | 60 | 26 | 14 | 1 | 8 | 37 | 0.33 | 1426 s |
A+L (on) | 99 | 60 | 32 | 7 | 1 | 0 | 52 | 0.33 | 1387 s |

Baseline na tagu (kontext, stejný model a scénář):

| běh | tahů | O₂ | úspěšné | pass | compile | coverage | wall |
|---|---|---|---|---|---|---|---|
gemma seed 42 | 60 | 3 | 32 | 7 | 21 | 0.67 | 1283 s |
thinkingcap seed 42 | 15 | 25 | 14 | 1 | 0 | 1.00 | 1024 s |

## Co z toho plyne

1. **Filtr nepřinesl postup.** Úspěšných akcí bylo 12 → 14 (seed 42) a 6 → 7 (seed 99),
   coverage stejná nebo nižší. Rozdíl je v šumu.
2. **Filtr překategorizoval selhání.** 46–53 compile errorů se změnilo na 37–52 `filter_block`.
   Běh skončil stejně – na `turn_budget`, ne na kyslíku.
3. **`waste` nelze mezi rameny srovnávat.** A+L účtuje blokované akce zvlášť, takže „waste“
   vypadá dramaticky lépe (0.79 → 0.15), ale počet úspěšných akcí se nezměnil. Kdo srovnává
   `waste` napříč rameny, měří kategorizaci, ne výkon.
4. **Kyslík není v tomto scénáři limitující.** Filtr ušetří O₂ (26/32 vs. 26/33), ale běh ukončuje
   počet tahů.

**Závěr:** filtr je uzavřený jako **negativní výsledek**. Zůstává v kódu jako volitelný
(`filter_mode off|on`), protože je neutrální a brání mrtvým smyčkám, ale **do v8 se nezapočítává
jako přínos** a pro srovnání s baseline se používá `filter_mode: off`.

## Omezení měření

1. **n = 2 na rameno.** `timeout 3000` byl nastaven na celé rameno, ne na běh, a třetí seed
   (123) byl zabit v 50. minutě. Pro uzavření filtru to stačí, pro tvrzení o velikosti efektu ne.
2. **Běhy nejsou reprodukovatelné.** `seed` se v době měření nepředával Ollamě, takže při
   `temperature 0.3` řídil jen generování secrets, ne sampling modelu. Doklad: baseline gemma
   seed 42 (tag) má 32 úspěšných akcí, A na `main` se stejným seedem 12.
   Opraveno commitem `88814b3` (`ChatOllama(seed=...)`); měření provedená před ním nejsou
   replikovatelná.
3. **Baseline běžela na tagu bez model seedu**, takže její čísla jsou populační odhad, ne
   replikovatelný běh. Pro v8 se seed předává a replikace je možná.
