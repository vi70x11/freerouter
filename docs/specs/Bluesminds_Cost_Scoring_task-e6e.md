# Bluesminds Provider Fix + Cost Scoring Dimension

## Context

Bluesminds is already registered as a custom provider (migration V27) with the correct endpoint `https://api.bluesminds.com/v1`. However, it has only 5 seeded models with incorrect IDs (e.g., `accounts/fireworks/models/deepseek-v4-pro` instead of actual Bluesminds model IDs). The routing scoring engine has 4 axes (reliability, speed, intelligence, latency) but no cost dimension. The spec requires adding cost-efficiency as a 5th scoring axis and updating the Bluesminds model catalog with ~35 models and their pricing.

---

## Task 1: Add pricing columns to the `models` table

**File:** `server/src/db/migrations.ts`

Add a new migration (V35 or next available) that:
- Adds `input_price_per_million` (REAL, nullable) to `models` table
- Adds `output_price_per_million` (REAL, nullable) to `models` table
- These are nullable so existing providers are unaffected (no pricing = cost prior)

---

## Task 2: Add cost scoring dimension to `scoring.ts`

**File:** `server/src/services/scoring.ts`

- Add `cost: number` to `RoutingWeights` interface
- Add `cost: number` to `ScoreInputs` interface
- Update `combineScore()` to include cost in the convex combination (weights renormalize automatically via `wSum`)
- Add `costScore(inputPrice, outputPrice)` function that normalizes cost to [0,1]:
  - Use log-scale normalization: cheaper models score higher
  - Reference range: $0.08/M (best) to $150/M (worst)
  - `score = 1 - (log(avgCost) - log(MIN)) / (log(MAX) - log(MIN))` clamped to [0,1]
  - Models with no pricing get a neutral prior (0.5) so they aren't penalized
- Add `COST_PRIOR = 0.5` constant

---

## Task 3: Update routing presets to include cost weight

**File:** `server/src/services/scoring.ts`

Redistribute existing preset weights to accommodate the cost dimension. Each preset's weights must still sum to 1.0:

```ts
balanced:  { reliability: 0.30, speed: 0.20, intelligence: 0.20, latency: 0.15, cost: 0.15 }
smartest:  { reliability: 0.25, speed: 0.10, intelligence: 0.40, latency: 0.15, cost: 0.10 }
fastest:   { reliability: 0.20, speed: 0.25, intelligence: 0.10, latency: 0.30, cost: 0.15 }
reliable:  { reliability: 0.50, speed: 0.10, intelligence: 0.15, latency: 0.10, cost: 0.15 }
```

---

## Task 4: Update router to compute cost scores

**File:** `server/src/services/router.ts`

- Import `costScore` from scoring module
- In `scoreChainEntry()`: compute cost score from `entry.input_price_per_million` and `entry.output_price_per_million`
- Pass cost into `combineScore()` inputs
- Add cost to `ChainRow` SQL query (select new columns)
- Update the `ScoreInputs` / `axes` return to include cost
- Update `getRoutingScores()` display to include cost axis

---

## Task 5: Update Bluesminds model inventory migration

**File:** `server/src/db/migrations.ts`

Add a new migration (V36 or next) that:
1. **Removes stale Bluesminds models** that have incorrect IDs (the 5 from V27):
   - `accounts/fireworks/models/deepseek-v4-pro`
   - `moonshotai/kimi-k2.6`
   - `qwen3.6-max-preview`
   - `z-ai/glm-5.1`
2. **Upserts the correct model catalog** from the spec with pricing data. Key models include:
   - OpenAI family: `gpt-4o`, `gpt-4o-mini`, `gpt-5-chat`, `gpt-5-mini`, `gpt-5-nano`, `gpt-5.5`, etc.
   - DeepSeek family: `deepseek-v3`, `DeepSeek-V4-Flash`, `deepseek/deepseek-reasoner`
   - Gemini family: `gemini-3-flash-preview`, `gemini-3.1-pro-preview`, etc.
   - GLM family: `glm-4.6`, `openai/zai-org/GLM-4.7`
   - Kimi family: `kimi-k2.5`, `Kimi-K2.6`, `kimi-thinking`
   - Llama family: `llama-4-maverick`, `llama-70b-fast`, `llama-8b-fast`
   - MiniMax family: `minimax-m2`, `MiniMax-M2.1`, `MiniMax-M2.1-lightning`, `MiniMax-M2.7`
   - Each with `input_price_per_million` and `output_price_per_million` from the spec
3. **Preserves case-sensitive model IDs** — do NOT normalize `DeepSeek-V4-Flash`, `Kimi-K2.6`, `MiniMax-M2.1`
4. Adds new models to the fallback chain at lowest priority

---

## Task 6: Update shared types

**File:** `shared/types.ts`

- Add `inputPricePerMillion?: number | null` and `outputPricePerMillion?: number | null` to the `Model` interface

---

## Task 7: Update settings/custom weights route

**File:** `server/src/routes/fallback.ts`

- Update the custom weights validation to accept a `cost` field
- Ensure GET /routing returns cost in the preset weights

---

## Task 8: Update client-side weight UI

**File:** `client/src/pages/SettingsPage.tsx` (or wherever routing weights are displayed)

- Add "Cost" slider/label to the custom weight controls so users can tune cost importance

---

## Task 9: Add tests

**File:** `server/src/__tests__/services/scoring.test.ts` (or new file)

- Test `costScore()` with various price points (cheap → high score, expensive → low score, null → prior)
- Test `combineScore()` with cost dimension
- Test that Bluesminds models have pricing data after migration
- Test that cost affects routing order (cheap model ranks higher when cost weight is high)

---

## Verification

1. `npm run test -w server` — all existing + new tests pass
2. Check migration runs without errors on a fresh DB
3. Verify Bluesminds `/models` endpoint returns matching IDs (manual check)
4. Verify cost score affects model ordering in the routing engine
5. Verify existing providers (no pricing data) get neutral cost prior and are unaffected
