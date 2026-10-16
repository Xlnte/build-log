# Four channels and the 120-second wall

Running four Reddit extractions in parallel cut the test's wall time from 16.2 minutes to 9.9. The first live run failed 24 of 30 threads because of a timeout the router chose by model name. The fixed rerun passed the timeout, then lost 11 of 30 threads to empty model replies. We have a hypothesis for those 11 and have not proven it.

## What we were trying to do

Our Reddit research pipeline ran one thread at a time, in a serial loop. Each thread is a model call, and the daily run covers 30 threads, so wall time was the constraint. We wanted to send several calls at once and let the model router decide how many run.

## Why four channels

We tested the same 40 cases at one, two and four channels, with temperature 0. Wall time fell from 16.2 minutes at one channel to 13.1 at two and 9.9 at four. One channel repeated itself exactly on all 40 cases. Four channels produced 60 signals against 67 at one channel, and an agreement score of 0.84.

An earlier test had suggested four channels hurt quality. That test used 23 cases, one model and sampled output, and isolation was not verified. The new test was stronger, so the operator approved four channels for the production run on 15 October.

## What we tried, and what went wrong on the way

**The concurrency code had a failure-semantics bug.** A reviewer found that when a worker raised an exception at more than one channel, the thread was recorded as failed and spent one of three retry attempts. At one channel the same exception would have crashed the run. A bad configuration could therefore retire a thread's queue entry after three daily runs. The fix made the run stop submitting work, drain in-flight calls and then re-raise the error. Tests passed at 142 of 142.

**The first live run failed 24 of 30 threads.** The 13:37 run recorded 24 failed threads and zero signals. Each failure was a timeout. The router set a 120-second limit for the model, which it chose by matching the model's name against a list of heavy-model hints. The new model's name matched none of them, so it got the fast-lane limit. In the logs, 31 requests were cut off at about 120 seconds and returned HTTP 502.

The fix was a per-tier timeout instead of a name list. The extraction tier now has a 900-second limit. The same deploy also made the router pass through the temperature and output-token settings that the pipeline had been requesting. Before this, the model ran on its defaults. The router version that carried both fixes went live at 18:08.

The 900-second figure is not arbitrary. It matches the client's own timeout, so the router never gives up first. Successful calls took 37 to 98 seconds. One test call with a 32,000-token prompt took 301 seconds, which would have failed the 300-second default by under a second.

**The rerun got 19 of 30 threads.** The rerun ran at four channels from 18:11 to 19:43. It recorded 19 successes, 11 failures and 3 signals. The router reported HTTP 200 for all 31 model calls, the slowest at 430 seconds. Every one of the 11 failures carried the same reason: an empty model reply.

## What happened

| Run | Threads failed | Signals | Failure reason |
|---|---|---|---|
| 13:37, four channels, 120 s limit | 24 of 30 | 0 | Router timeouts, HTTP 502 |
| 18:11 rerun, four channels, 900 s limit | 11 of 30 | 3 | Empty model reply |

## Our hypothesis, not a finding

The 11 empty replies were the 11 longest model calls, at roughly 395 to 430 seconds. That is close to the 8,192-token output cap the pipeline requests. Our working theory is that the model spends its output budget on hidden reasoning and leaves no visible answer.

This is unproven. The router logs zero completion tokens for these calls, so we cannot count what the model generated. We did see this model emit a thinking block before its final answer during earlier tests. Public issue trackers for related tools describe empty outputs after reasoning, though none matched our exact pattern.

The test that would confirm or reject the theory is queued. We will re-run the 11 failed threads with thinking turned off. If at least 9 of 11 succeed, we will wire that setting into the client and review the change. That test has not run yet.

## What we learned

- A timeout chosen by matching model names breaks when a new model arrives. Set timeouts per tier, in configuration, and do not add model names to a list.
- A performance setting must not change failure behaviour. The concurrency change looked safe at one channel and was not safe at four until a reviewer traced the exception path.
- Failure reasons were stored in the database but not written to the operator log, so the 11 failures were invisible until someone queried the database. We filed that as a follow-up.
- Four channels bought speed at a measured quality cost: agreement 0.84 and 60 signals against 67.

## What's next

The next scheduled run is 16 October at 13:34. Before that run, the operator decides whether to run the thinking-off test on the 11 failed threads. We will not change the client until the test passes.
