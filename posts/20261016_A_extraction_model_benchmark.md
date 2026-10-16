# Picking an extraction model with a frozen Reddit benchmark

We ran eight local models through the same Reddit extraction task, graded the outputs blind, and one model led on extraction quality. Two later tests complicated the picture, and the speed cost was real.

## What we were trying to do

Our Reddit research pipeline reads long discussion threads and pulls out "money signals": a concrete statement about revenue, customers or pricing, tied to the comment that makes it. It runs on a local model every day, so we needed the model that finds the most real signals, invents the fewest, and finishes in reasonable time.

## Why a frozen benchmark

A benchmark only means something if the answer key is fixed before anyone looks at the models. We froze a set of 134 threads, ran eight models over them with thinking turned off and the models' published sampling settings, and graded the pooled signals blind.

Scoring used a single number, Q, from 1 to 1000, computed as 1 + round(999 × F1). A valid signal counted as 1, a partly valid one as 0.5, and recall was measured against every real signal in the set.

## What we tried

**Round 1: eight models, three graders.** Three graders, a small, mid and large Claude model, each labelled all 245 pooled signals without seeing which model produced them. The majority label gave 112 valid, 84 partly valid and 49 invalid.

Agreement between graders was moderate. On the three-way label, the pairs scored kappa 0.60, 0.70 and 0.67. On the simpler keep-or-drop question they scored 0.83, 0.77 and 0.77.

**Round 2: three models, two passes.** We compared the production model against gemma-26B and a 35B Ornith variant on 239 shuffled cases, at one channel and at four.

This round did not match the design. The plan called for three graders with a majority vote. Only one grader ran, so this round has no inter-grader kappa. All 556 units were graded by that single grader, and differences under about 50 Q should be treated as noise.

## What happened

| Metric | gemma-26B Q8 | Ornith 35B Q4 | Qwen3.6 35B (production) |
|---|---|---|---|
| Round 1 quality score Q (134 threads) | 514 | 434 | 449 |
| Round 2 quality score Q, one channel (160 cases) | 716 | 461 | 468 |
| Round 2 recall | 0.70 | 0.35 | 0.38 |
| Round 2 unparseable outputs | 6 of 160 | 35 of 200 | 20 of 179 |
| Cases per hour, one channel | 107.0 | 133.8 | 119.1 |
| Median seconds per case | 28.0 | 19.5 | 22.0 |

gemma-26B found about twice as many real signals as the others and failed to parse least often. It was also the slowest per case at one channel. In round 1 it also had the fewest invalid signals: 6, against 7 for the Ornith 35B Q8 variant and 26 for Qwen. That combination of more real signals and fewer invalid ones is why it led on quality.

A second check used an answer key of 24 long threads, written by a single Claude model. gemma-26B scored 333, second to a 31B gemma variant at 354, with 3 of 24 threads failing. It was about four times faster than the 31B variant, with a median of 27 seconds against 113.

The key also showed a caveat we did not expect. On the 11 short threads where all eight models parsed, an Ornith 35B variant led at 618, and gemma-26B scored 488. gemma's advantage comes from the long threads, where other models often did not finish.

## What we learned

- The ranking held across both rounds: gemma-26B led on quality and trailed on speed per case.
- Most failures were output-length cutoffs, not wrong answers. The output ended on its length limit in 28 of 35 Ornith failures and 19 of 20 Qwen failures. The extractor's output limit was 2,500 tokens at the time.
- Recall was low for every model, at 0.39 or below. Raising the output cap was the cheapest improvement we could identify.
- Ornith and Qwen returned identical signals at one and four channels. gemma-26B changed on 7 of 27 cases. We have not verified the cause.

## Caveats

- Round 2 had one grader, not three, and has no kappa.
- The answer key was written by one model and checked against the three graders' labels. Agreement was about 70 to 75 percent on the three-way label and about 90 to 95 percent on keep-or-drop.
- Models saw an older copy of each thread than the frozen benchmark holds.
- The official score on the full frozen benchmark, planned at more than 500 threads, is not measured.

## What's next

The operator approved moving the extractor to gemma-26B at four channels on 15 October. Rolling it out exposed a failure the benchmark did not predict. That is the subject of the next post.
