# The empty replies were thinking

Our last post ended with 11 empty model replies and an unproven theory. This one has the proof, the fix and its test results. While tracing the cause we also found a second, unrelated bug in our router's logging. The fix is now live.

## What we saw

On the rerun at four channels, 19 of 30 threads succeeded and 11 failed. Every failure had the same reason: an empty model reply. The router reported HTTP 200 for all 31 model calls, and the slowest took 430 seconds. The calls worked. The answers were blank.

## How we found the cause

The evidence chain had five links.

1. **The failure reason was stored.** The row says "empty model reply".
2. **The router passed the reply through unchanged.** It returned the model server's response as-is, with no check on why generation stopped.
3. **Our client read one field.** It took only the answer field. The model's reasoning arrived in a separate field, which the client never read.
4. **The model server's log showed the cap.** Exactly 11 generations stopped at the 8,192-token cap, all inside the thinking block. The other 19 generated between 541 and 4,572 tokens.
5. **The timing matched.** The server decodes at about 20 tokens per second. 8,192 tokens at 20 per second is about 410 seconds, inside the 395 to 430 second range of the failed calls.

Our reviewer read the 19 successes the same way. Their JSON answers are a few hundred tokens long, so most of their output budget went to thinking, and they finished under the cap.

## Why the model was thinking

The model's chat template turns thinking on by default. Our server launch set no reasoning flags, so llama.cpp used its default, which is thinking on. A research agent read the model card: with thinking off, the model still emits an empty thought block, so the answer comes out clean.

## Why not raise the cap

Raising the cap to 16,384 was the obvious stopgap. We rejected it because it would break our time budget per call and would run a setup our benchmark never scored:

- At 20 tokens per second, 16,384 tokens is about 820 seconds of decoding. With prompt processing and queueing, that passes the 900-second limit on the tier.
- Thinking length is set by the model, not the input. Two threads with no comments hit the 8,192 cap, and no measurement says 16,384 would be enough.
- It doubles GPU time for the failing threads and keeps production on a setting nobody benchmarked: thinking on, temperature 0.

## The fix

The fix sends `think=false` with each request. It is one optional argument in the pipeline's client. The default is off, and setting one environment variable back to 1 removes it, so a rollback is one variable. The failure reason now carries the stop reason, the token count and the thinking length, so this failure will show on the row next time. The 8,192 cap stays as a safety net.

## How we knew it would work

An earlier bake-off sent `think=false` to the same model tag, on the same server binary, for 134 threads. Thinking characters were zero on 134 of 134. Median output was 331 tokens, and 131 calls stopped normally. The three that hit the cap were long JSON, not thinking.

That proves the setting works on the model server. It does not prove it works through our router. The bake-off went straight to the server, so the router path is untested until the run finishes.

## A second bug

The trace also showed that the router logged zero output tokens on every local call. All 346 model route events from the day show zero. A token-counting library was missing from the Python the router runs under, so its counter returned 0. The server reported the real counts, and the router never read them. The same dead counter disabled a routing branch that upgrades context size.

So the one number that would have exposed the problem was blank. The fix is on the task list.

## What we learned

- **A reply can succeed and still be empty.** The status code said 200 and the content said nothing. Read the stop reason, not only the status.
- **Log the stop reason and token counts on every call.** Without them, this sat invisible until someone traced the logs.
- **If a model can put its output in another field, the client has to read that field.**

## Test results

To test the fix (sending thinking off), we ran the same 30 threads through the model with thinking on and with thinking off, at four parallel channels, each run isolated on the box.

| Setting | Threads OK | Ran out of tokens | Median seconds per thread | Signals found |
|---|---|---|---|---|
| Thinking on (the old setting) | 18 of 30 | 12 | 168 | 2 |
| Thinking off | 30 of 30 | 0 | 20 | 24 |

Turning thinking off fixed every failure and made the median thread about eight times faster. With thinking off, the model wrote no hidden reasoning at all on any of the 30 threads. We also tried two channels instead of four: the whole run took 300 seconds against 315, and it found 29 signals against 24. A blind grader scored the quality of each signal about the same either way.

The fix is now live. The next scheduled run will be the first real proof that the setting reaches the model through our router.

## What's next

The fix is live; the next scheduled run is its first real test through our router. Separately, the blind grading showed the extraction prompt itself needs work: it invents some fields and misses some revenue facts on every model we tried. That is the next post.

