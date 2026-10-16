# A good extractor is not a good coder

The model that won our Reddit extraction benchmark failed our coding and judgement tests. It is now restricted to extraction. Whether it can earn more work depends on a fix we have not made yet.

## What we were trying to do

Our model router has several tiers: a heavy general tier, coding tiers and a small tier. Once gemma-26B won extraction, the obvious question was whether it could take on more work, such as the heavy tier's judgement tasks or a coding tier. We tested it against the current occupants of those roles.

## What we tested

- **Three VP-style judgement prompts**, for technical review, operations and finance, each scored out of 10 against a frozen rubric.
- **A single-shot coding task**, scored out of 100 across six subscores: constraints, code quality, tests, operations, production readiness and stability.
- **A software-engineering benchmark**, 30 items, Python slice, scored on hidden tests.
- **A hallucination check**, 10 cases.

The comparison set was the heavy Qwen3.6 35B tier, the qwen3-coder-next coder model and a small huihui e2b model. Several comparison scores come from earlier runs, in April and June, and are not directly comparable to the October run.

## What happened

| Test | gemma-26B | Heavy (Qwen3.6 35B) | Coder-next | Small e2b |
|---|---|---|---|---|
| VP mean, out of 10 | 3.0 | 8.3 | not graded | 8.0 |
| VP mean, final answers only | 7.8 | not applicable | not applicable | not applicable |
| Coding, out of 100 | 46 | 76 | 65 | 68 |
| SWE benchmark full passes, of 30 | 1 | 5 | 10 | 0 |
| SWE mean pass fraction | 0.336 | 0.347 | 0.582 | 0.117 |
| Decode speed, tokens per second | 46.8 | 55.8 | 44.0 | 98.9 |

The judgement scores are capped by a defect, not by content. All three VP answers and the coding answer included the model's raw thinking text before the real answer. The rubric caps any answer like that at 3 of 10. Scoring only the final answers gives 7.8, still below the heavy tier's 8.3, and the model was slower.

The coding answer had concrete faults. It used wall-clock time for a circuit-breaker timer. It ended its tests with a placeholder, "I'll assume the logic is verified", which the prompt bans. It also omitted a requirements file that its own Dockerfile needed. The coding grade was produced by one model, not a human, so it carries a margin of about plus or minus 5.

On the software-engineering benchmark, gemma-26B passed one of 30 items in full. The coder-next model passed ten.

## The thinking defect is specific to our import

An earlier Ollama build of the same model did not leak its thinking text on the same prompt. The leak comes from the way our copy of the model was imported, probably its chat template and thinking handling. Google's model card says the 26B variant still writes an empty thinking block when thinking is off. We have not verified whether that explains our leak.

The hallucination check does not prove much. All 10 cases were positive extractions, which cannot show fabrication. Forty cases never ran.

## Decision

The model is routed to extraction only. We will not send it to the heavy, VP or coder tiers. The current data does not show that it beats the heavy tier on any of those jobs.

## What we learned

- A winning model on one job says nothing about another. Score every candidate on the job it will actually do.
- A leaked reasoning block can cap a score even when the answer is good. Check the raw output before reading the score.
- Speed matters for tier choice. The 26B variant decodes about 47 tokens per second against 56 for the heavy tier.

## What's next

The fix is to correct the thinking handling on this import, then re-run the three judgement prompts. We estimate that run at about five minutes of GPU time. That is an estimate, not a measurement. Until it runs and passes, the model stays in extraction only. Nothing in this post shows the fixed model would beat the heavy tier. That is not measured.
