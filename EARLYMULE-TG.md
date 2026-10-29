# EarlyMule-TG changes to AMLSim

This fork of [IBM/AMLSim](https://github.com/IBM/AMLSim) generates the synthetic dataset for the EarlyMule-TG thesis (early money-mule detection on temporal transaction graphs). The thesis needs three things from the simulator: a laundering flow that spans several days, mule accounts that behave normally before they launder, and money that moves through a cycle in time order. The original code broke all three, so it was changed as described below.

- **Base:** upstream commit `7338a4b` (2022-02-14, last upstream commit)
- **Branch:** `earlymule-fixes`
- **All changes at a glance:** [compare `7338a4b...earlymule-fixes`](https://github.com/Verzol/AMLSim/compare/7338a4b...earlymule-fixes)
- **License:** Apache License 2.0, unchanged. Modified source files carry a "Modified by EarlyMule-TG" header and an `EarlyMule-TG:` comment at each changed block.

## Summary

| # | Problem in the original output | Change | Before | After |
|---|---|---|---|---|
| 1 | Fan-in/fan-out alerts stretched over most of the simulation | Two lines removed in `FanInTypology.java`, `FanOutTypology.java` | median span 363 / 195 days | 9 / 8 days |
| 2 | Alert members often had no normal transactions at all | Member sampling in `scripts/transaction_graph_generator.py` | 41.2% of mule-like accounts had normal activity before the alert | 98.1% |
| 3 | Cycle relays sent money before receiving it | Parameter only: `schedule_id` 2 → 1 for cycle rows | 112 of 249 relays out of order | 0 of 266 |

Numbers come from the 10K configuration (12,043 accounts, 100 alerts), original vs. patched, same random seed.

## 1. Fan-in / fan-out start step

**Problem.** The generator gives every alert a window `[startStep, endStep]` of 5–20 days placed at a random point in the simulation. `FanInTypology` and `FanOutTypology` then overwrote `startStep` with a random step close to 0 but kept `endStep`. A fan-in alert planned for days 400–410 therefore ran from about day 0 to day 410. The median fan-in alert started on day 0 and lasted 363 days, so there was no early phase to detect.

**Before** (`FanInTypology.java`; `FanOutTypology.java` is the same with `numBenes`)
```java
int totalStep = (int)(endStep - startStep + 1);
int defaultInterval = Math.max(totalStep / numOrigs, 1);
this.startStep = generateStartStep(defaultInterval);  //  decentralize the first transaction step
```

**After**
```java
// EarlyMule-TG: removed `this.startStep = generateStartStep(defaultInterval)`.
// ...
```

**Why this is a bug and not a design choice.** `CycleTypology`, which uses the same "decentralize" idea, moves the whole window (`startStep` and `endStep` together). Only the fan classes move the start alone.

## 2. Alert members drawn from active accounts

**Problem.** `add_aml_typology()` picked alert members with `random.sample()` from all accounts in a bank. In the 10K run only 4,672 of 12,043 accounts make any transaction (many have a normal model that never fires), so 53.4% of alert accounts had no normal activity at all. That has two effects. A mule has no "normal → mule" transition to learn. And "has no normal transactions" alone almost reveals the label.

**Change.** Two helper methods were added, and all 17 `random.sample(self.bank_to_accts[...], k)` calls in `add_aml_typology()` now call `_sample_active_members(...)` instead:

```python
def _sample_active_members(self, candidates, num):
    pool = sorted(candidates)
    active = [a for a in pool if self._has_reliable_normal_activity(a)]
    if len(active) >= num:
        return random.sample(active, num)
    # fallback: take all active accounts, fill up with the rest
    ...

def _has_reliable_normal_activity(self, acct):
    for nm in self.g.node[acct].get('normal_models', []):
        if nm.is_main(acct) or nm.type not in ("fan_in", "fan_out"):
            return True
    return False
```

**Why this rule.** Measured on the original 10K run, an account transacts in about 99% of cases if it is the main account of any normal model or a member of a single, forward, mutual or periodical model. It transacts in only about 39% of cases if it is only a non-main member of a fan-in or fan-out normal model, because the Java fan-out model sends to the account's beneficiary list, not to the model members.

**Side effect to keep in mind.** After the change, alert accounts have *more* normal transactions than the average active account (median 123 vs. 4). The thesis therefore evaluates only on accounts with regular normal activity, where the gap disappears (median 115–120 vs. 107).

## 3. Cycle transactions in time order

**Problem.** Cycle alerts in the sample parameter files use `schedule_id = 2` (`UNORDERED`): each hop gets a random day, so a relay can send money before it has received it.

**Change.** No code change. In `paramFiles/*-earlymule*/alertPatterns.csv` the cycle rows use `schedule_id = 1` (`RANDOM_INTERVAL`). `CycleTypology.setParameters` uses the same code path for both values but sorts the steps only for `1`.

## Configurations added

| File | Accounts | Alerts | Notes |
|---|---:|---:|---|
| `conf_10K.json` | 12,043 | 100 | Original `paramFiles/10K`; the copy inside that folder lacks `normal_models`, `sar_interval`, `sar_balance_ratio` |
| `conf_10K_earlymule.json` | 12,043 | 100 | `paramFiles/10K-earlymule`, cycle `schedule_id = 1` |
| `conf_80K-earlymule-a100.json` | 96,344 | 100 | 10K-earlymule accounts and normal models ×8 |
| `conf_80K-earlymule-a200.json` | 96,344 | 200 | Same, 200 alerts; the configuration used in the thesis |

Alert mix in all EarlyMule-TG sets: 30% fan_in, 30% fan_out, 40% cycle.

## Run

Environment and build steps are in the EarlyMule-TG repository (`docs/setup.md`). With the environment ready:

```bash
python scripts/transaction_graph_generator.py conf_80K-earlymule-a200.json
bash scripts/run_AMLSim.sh conf_80K-earlymule-a200.json
python scripts/convert_logs.py conf_80K-earlymule-a200.json
```
