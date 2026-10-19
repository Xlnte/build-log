# Same mistakes on every model: the prompt was the problem

Our last post fixed blank answers from our reddit research pipeline. The answers were blank because the model spent its whole output budget on hidden reasoning. This post covers the next problem, which the fix exposed: the extraction prompt invented facts on every model we tried. Seven model and setting combinations did not fix it. A rewritten prompt plus a few deterministic checks did, though the change also cut the number of results.

## Where we left off

With reasoning turned off, every thread in our test returned an answer. A blind grader then read the extracted results against the source threads. It found the same extraction mistakes in every arm.

## What we tried

We ran seven arms on the same 30 threads. They varied the setting that controls randomness (temperature 0 or 1.0), the build of the model (one official release, two community builds, and our usual build), and the number of parallel channels (1, 2 or 4).

| Arm | Signals found | Mean score out of 1000 |
|---|---|---|
| Usual build, temperature 0, 1 channel | 25 | 628 |
| Usual build, temperature 0, 2 channels | 29 | 629 |
| Usual build, temperature 0, 4 channels | 24 | 640 |
| Usual build, temperature 1.0, 4 channels | 25 | 620 |
| Official release, temperature 1.0, 4 channels | 25 | 620 |
| Community build A, temperature 1.0, 4 channels | 12 | 625 |
| Community build B, temperature 1.0, 4 channels | 20 | 582 |

Quality per signal stayed between 582 and 640 across all seven arms. The official release was the fastest per thread but found fewer valid results. Turning temperature up gained nothing. Channel count changed speed, not quality: four channels took 315 seconds for the 30 threads, two took 300 and one took 377.

## What happened

The mistakes were the same in every arm. Across the 34 signals we graded, the model set funding to "bootstrapped" in 17 cases. The thread supported that in 3 of them. It filled a revenue field in 4 units, and none of those figures was current revenue. It marked 5 people as founders when they were former workers, advisers, or users of a product. It copied a username into the field for who has the problem. Across 52 units, it also named tools the thread never mentions in four units.

We changed the prompt and added checks that run in code, not in the model:

- Fields default to null. The model must have evidence to fill one.
- Funding is kept only when a funding keyword appears in the cited message.
- Revenue counts only when it is current income. A price, a cost, a plan, a goal, or a past figure is not revenue.
- A username cannot be the "who has the problem" value.
- A tool name is kept only if it appears in the thread.

On our production data, the funding check kept 9 of 79 labelled signals and nulled 70. We did not check how many genuine funding statements it removed.

The same 30 threads, rerun with the new prompt at two channels, all completed. The results changed as follows:

| Measure | Before | After |
|---|---|---|
| Threads completed | 30 of 30 | 30 of 30 |
| Signals found | 29 | 23 |
| Valid signals | 10 of 29 | 20 of 23 |
| Partly valid signals | 17 | 2 |
| Invalid signals | 2 | 1 |
| Fields fully correct | 10 of 29 | 18 of 23 |
| Mean score out of 1000 | 616 | 696 |
| Funding set to a value | 17 | 0 |

The number of signals fell from 29 to 23. The reviewer checked each of the six lost signals. None was removed by a code check. The model left them out under the new prompt, and the reviewer reads this as the longer list of exclusions making the model more cautious. Two of the lost signals were valid, with scores of 800 and 700. Counting valid and partly valid results together, the total fell from 27 to 22. We traded some recall for precision, and we have not fixed the recall loss yet.

A reviewer agent read the diff, the checks, and the stored rows before we trusted the numbers. It found five defects in our new checks:

- A phrase like "I worked with" was treated as employment.
- The negated phrase "no outside funding" counted as support for "funded."
- A total over a span ("$30k over six months") passed as a monthly figure.
- Two other keyword checks were too narrow or too broad.

We fixed all five, added a test for each, and the suite passed 171 of 171.

## What we learned

- **Measure validity, not counts.** The count fell from 29 to 23, which looks like lost data. Valid results rose from 10 to 20.
- **Changing the model did not fix a defect in the instructions.** Seven arms moved speed and cost, but the mistakes stayed.
- **Deterministic checks catch what a prompt does not.** A keyword check in code is easy to test and does not depend on the model following a rule. We did not isolate how much of the gain came from the prompt and how much from the checks.

## Caveats

- Thirty threads and one model family is a small sample.
- Each grade is a judgement by one grader agent, not by two.
- The grader did not know which prompt produced which signal; we matched grades to prompts only after grading finished.
- The before-and-after comparison uses one run per arm. We have not measured run-to-run variance.
- The reviewer's explanation for the lost signals is its reading, not a controlled test.
- The funding check may remove real funding statements. We have not measured that.
- The numbers come from our internal threads. We have not published the threads.

## Status

The corrected change has been reviewed and approved with conditions. It is not deployed.

## What's next

The next step is a prompt-only change that states that a co-founder, a part-time founder, or a solo founder counts as a signal. We will rerun the same 30 threads and grade both arms before deciding whether to deploy.
