---
title: "What Code Review Looks Like After AI Starts Writing the Code: The Industry in 2026"
date: 2026-11-08
draft: false
description: "A map of AI code review in 2026: risk-based review, independent review agents, verification evidence, and questions that remain open."
tags: ["ai-code-review", "agentic-engineering", "software-engineering", "verification"]
categories: ["AI Engineering"]
---

Next in this series: [My Thoughts on AI Code Review: Selective Review, with AI Doing the Triage]({{< relref "ai-review-selective-triage.md" >}})

*Once AI made writing code cheap, review became the slowest part of delivery. Over the past few months, I have read more than twenty talks, articles, and data reports on AI code review. This post organizes them into a map: where the problems lie, what people are doing, and what remains unanswered.*

One caveat up front: almost all of these materials are practitioner accounts and vendor data. None is a controlled experiment. The industry is still finding its way. This post records the state of practice in the second half of 2026; it does not offer settled conclusions.

## 1. The data: faster upstream, slower downstream

Several industry reports from 2025 and 2026 point to the same paradox: AI adoption is rising, defects and incidents are rising with it, but delivery is not getting faster.

- **DORA's generative AI report (2025 edition, based on the 2024 study):** a 25% increase in AI adoption is associated with an estimated 7.2% decline in delivery stability. This is a correlational estimate, not an incident-rate increase or proof that AI caused instability.
- **Faros AI** compared organizations before and after adoption across roughly 22,000 developers and 4,000 teams: incidents per PR rose 242.7%, median review time rose 441.5%, PR size rose 51.3%, and PRs merged without any human or agent review rose 31.3%. Upstream throughput did improve: task throughput increased 33.7%. Yet weekly deployments fell 11.7%, and time from commit to production rose 480%. Those last two metrics cover only the roughly 10% of teams in the dataset with deployment instrumentation. Faros also cautions that commit-to-production time varies substantially between teams.
- **Stack Overflow's 2026 survey:** among 14,304 respondents to the trust question, 48% trust AI when they can easily verify its output; only 6.6% trust it for many tasks including important work decisions. This measures the conditions under which people trust AI, not the accuracy of its output.
- **GitClear's 2026 report** analyzes roughly 623 million code changes from 2023–2026: the block-duplication metric, normalized per million changed lines, rose 81% against 2023. Moved code, a proxy for refactoring, fell from 21% in 2022 to 3.8% year-to-date in 2026. These are maintainability signals, rather than a direct measurement of AI's causal effect.

Laurie Voss cited an even more extreme figure in a talk: developers using autonomous agents produced 741% more code but delivered only 30% more software, with review identified as the bottleneck. The study was not named in the talk, and I have not checked the original, so I treat it only as a directional signal.

## 2. Why reading the diff breaks down

Traditional code review rests on a rarely stated assumption: reviewers have their own understanding of the system because they have written adjacent modules themselves. Review adds value through the gap between that mental model and the code in front of them.

Handing implementation to AI can undermine that assumption in two independent ways.

**First, the context disappears.** If a reviewer's entire understanding comes from reading the PR itself, there is no independent gap between their mental model and the code, and review adds no information. From this perspective, merging PRs without review is sometimes more than laziness: the review process itself has become an empty ritual.

**Second, there is too much to read.** Voss cited an older Cisco study covering roughly 2,500 reviews and 3.2 million lines of code: beyond about 400 lines in a sitting, reviewers stop detecting defects effectively. At that pace, a 10,000-line agent PR would take three to four working days of genuine review. The study predates AI and examined human-written code. Applying it to agent code is an assumption, though the direction seems plausible.

There is a more fundamental limitation too. Stellman, writing for O'Reilly, discusses **security defects**: the classic distinction he cites assigns roughly half to implementation bugs and half to design flaws, such as missing authorization constraints or incorrect trust boundaries. Reviewing code structure without knowing the intended security properties can miss the latter. That does not establish a universal rule that half of all defects are undetectable.

## 3. Counterexamples at both extremes

The two immediate responses are to stop reading entirely or to keep reviewing every line as before. Both have counterexamples.

**Stop reading entirely.** Dex Horthy publicly advised people to stop reading code, then withdrew that advice six months later after a codebase he had stopped reading for three months developed problems no prompt could solve. DHH described designers at 37signals writing code directly with AI: each PR looked defensible in isolation, but together they damaged the system's architecture and required manual cleanup. Bun's million-line Zig-to-Rust port took six days and passed 99.8% of existing tests, yet contained 13,044 unsafe blocks, compared with roughly 74 in a comparable human-written Rust codebase.

**Read every line.** The data above already shows that human attention does not scale with code volume.

Look closely at the successful “we don't read code” stories, and people have not actually let go. What they gave up was line-by-line reading. They still control the verification systems: tests, harnesses, and review agents.

## 4. Practices taking shape

### Allocate review depth by risk

This is where convergence is clearest. However, that convergence is mainly among practitioners who share their work publicly; actual adoption remains very low. At least four independent sources describe the same idea in different terms: review depth should depend on **blast radius**, or how widely a failure could spread, and **reversibility**, or how easily the change can be rolled back.

- Meta's John Kim treats a codebase as a tree: read trunk code, such as entry points, shared state, and infrastructure, line by line; scan leaf code that is isolated or fully behind a feature gate against the accompanying evidence.
- Matt Pocock borrows AWS's distinction between one-way and two-way doors and prints a “merge danger” line at the bottom of each PR. Most PRs are two-way doors. An expensive migration, data loss, or an apparently small change that could email 60,000 people is a one-way door and demands intense scrutiny.
- Kun Chen's personal workflow ends with a risk assessment and skips diff review for low-risk changes.
- In Voss's survey of practice, human checkpoints remain where correctness is hard to verify automatically, blast radius is large, or someone must explicitly sign off and take responsibility.

Anthropic's Claude Code team offers a concrete policy. Its risk tiers specify how many people must approve: documentation, copy, internal features, and incremental changes behind feature flags require none; ordinary product changes and sensitive areas such as caching, auto-updates, and system prompts require one; the highest-risk code requires two. Several supporting rules are notable:

- A classifier assigns the tier based on the change, rather than the author's choice.
- A second classifier can only **add reviewers**, never remove them.
- Paths designated sensitive by their owners, and all CI configuration files, always require a human.
- Agent approval can never replace a required human approval.
- Approvals bind to a particular commit and become invalid when a new commit arrives.
- The rollout advice is to start strict and relax only when evidence supports it.

There is an asymmetry here that I think matters: underestimating risk costs far more than overestimating it. Automatic classification can raise the tier, but lowering it should require a person or an explicit rule with a human owner.

For comparison, WorkOS's Nick Nisi proposed a three-layer cascade that has not yet been validated in deployment: a team review skill, a lightweight classifier, and a commercial review tool. A failure at any layer escalates to a person; passing all three allows automatic merging. The difference is that this has no deterministic floor requiring human review for certain changes regardless of what the layers conclude.

### Design the review agent as an independent component

Asking the coding agent to check its own work is insufficient. Several sources describe design rules for the reviewer itself:

- **Independent, fresh context.** An agent carrying the generation history is anchored by the earlier conversation. Corey Gallon described a drafting agent that checked every date in a contract and passed its own work. Two reviewers with fresh contexts independently found a three-year automatic renewal clause: the contract had renewed three weeks earlier, and the drafting agent had deleted the sentence that correctly pointed this out. Its self-check endorsed its own mistake.
- **Separate context budgets.** Pocock observes that the implementer's window is already overloaded with exploration, edits, and debugging. Adding coding standards makes it worse. The reviewer does not need the implementation and debugging history and has room for extensive standards. In this approach, team review standards belong in the review agent's context.
- **Default skepticism and multiple passes.** Cursor's early reviewer ran eight passes over each diff, shuffling the order because order affected the results. After a redesign, it needed explicit instructions to distrust the code by default; otherwise it would simply approve it.
- **Submit fixes directly.** A reviewer that only comments creates more reading for people. Fixing clear issues and commenting only on genuine questions gives the human a better version to review.
- **Pair agents with deterministic program analysis.** Sonar's approach gives agents logic and intent, taint and data-flow analysis the parts that can be calculated precisely, and supply-chain scanners the dependencies.
- **Make the loop converge.** Anthropic found that author and review agents could generate endless work for each other. One PR went through more than fifty rounds of review and repair overnight. A later git blame analysis showed that roughly 59% of the repeatedly reported problems were in old code already present at the first round. The reviewer was finding a few more old issues each time. Their changes included tracking rounds-to-pass alongside recall, requiring the author to reconsider the approach after three rounds, reviewing only new diffs after the early rounds, reporting by bug category so the author could fix related issues together, and avoiding repeated findings.

### Layer checks with different blind spots

Layered review resembles the Swiss cheese model: every slice has holes, but stacked slices are unlikely to have holes in exactly the same places. That depends on the holes being different.

Layers organized by defect category naturally cover different gaps: deterministic analysis handles bugs and security, agent review handles logic and intent, and supply-chain scanning handles dependencies. Several reviewers built on similar models may have correlated blind spots instead. Voss cited a result in which vulnerable code accompanied by an innocent-looking commit message fooled automated reviewers 88% of the time, compared with 35% for human reviewers. Confidently packaged bad code is exactly what agents are good at producing. Many apparent layers may provide little actual independence.

### Change what people inspect

As reading diffs yields less information, human judgment needs a different object. Two directions stand out:

- **Move earlier and inspect boundaries.** Vercel's Guillermo Rauch calls the remaining human work “verification engineering”: agree on correctness, performance budgets, and speed-versus-security trade-offs before the agent starts. Spend attention on system boundaries rather than implementation details. One practical rule is to notify him in Slack whenever the shape of an external API changes.
- **Review behavioral diffs.** Meticulous replays recorded user flows for every PR and presents visual differences for a reviewer to judge. Reviewing those differences does not require having written adjacent modules.

Both approaches come from vendors whose products fit their arguments. Still, the underlying idea is general: separate the inspection of what a change touches, which need not require deep context, from an approve button that tries to cover everything.

### Require evidence of completion

The weakest check is the agent's own claim that the task is done. Another overlooked problem is that passing tests establish only what those tests were designed to check. Even a well-intentioned agent can write tautological tests or tests that cannot fail. Several sources independently argue that completion should be demonstrated by a record someone other than the author can verify, such as the exact commit tested and a record of the verification run, rather than the agent's account.

## 5. Questions without answers

- **Does the person issuing the prompt count as a reviewer?** This is the clearest disagreement I found. Gautam Korlam of Gitar, now part of Sonar, says no: the prompter and agent share context, and separation of author and approver should apply to the whole creation process. Anthropic takes the opposite approach: the person directing the agent counts as one required reviewer because whoever initiates a change is responsible for it, with risk tiers and fast rollback as compensating measures. Neither side has published the effect of this rule on missed defects.
- **How do we verify maintainability?** Coding agents keep improving at making tests pass, with little progress in keeping codebases maintainable. One explanation is that training rewards passing tests, while bad architecture takes months or years to impose its costs. Two independent teams offer the same remedy: move design earlier. For now, people still carry responsibility for maintainability.
- **Who reviews the checks and review agents?** Voss's point is that every layer of a verification system remains unreviewed until someone takes responsibility for it.
- **When is it safe to lower a risk tier?**
- **How will junior engineers develop review judgment?** If engineers stop writing adjacent modules from the beginning of their careers, will they be able to review trunk code?
- **Where does the cost of skipped review go?** One view is that it survives as downstream incident-response work.

## Closing thoughts

Taken together, these materials make “code review is dead” both understandable and misleading. What is disappearing is the activity of reading every line of a diff. The function of review remains. It is splitting apart: some moves earlier into design and agreements, some moves later into behavioral verification and runtime, and much routine checking goes to layered automation. Human attention concentrates on a smaller number of important changes according to risk.

Uncertainty runs through this map. Most evidence consists of practitioner accounts and vendor data, while tools and models change every few months. In the next post, I describe my own current thinking, including how I view the question of whether the author should count as a reviewer.

## References

These links cover the data, examples, and practitioner accounts discussed above. The 741% / 30% figures, Cisco review thresholds, Bun unsafe counts, and 88% / 35% figures are relayed by Laurie Voss; I have not independently checked the underlying studies or code counts. The DHH example comes from a secondary account saved in my research notes; the original interview is linked for traceability. WorkOS's cascade is a proposal, not a validated deployment. The observation that actual adoption is very low is my own, not an industry adoption rate measured by these sources.

- [DORA, Impact of Generative AI in Software Development (2025 guide, based on the 2024 study)](https://dora.dev/ai/gen-ai-report/dora-impact-of-generative-ai-in-software-development.pdf)
- [Faros Research, The Acceleration Whiplash (2026)](https://www.faros.ai/research/ai-acceleration-whiplash)
- [Faros Research, Ten Takeaways from the AI Engineering Report 2026](https://www.faros.ai/blog/ai-acceleration-whiplash-takeaways)
- [Stack Overflow Developer Survey 2026 — Trust in AI Tools](https://survey.stackoverflow.co/2026/ai/data/ai-trust)
- [GitClear, The Maintainability Gap: AI Code Quality in 2026](https://www.gitclear.com/the_ai_code_quality_maintainability_gap)
- [Andrew Stellman, AI Code Review Only Catches Half of Your Bugs](https://www.oreilly.com/radar/ai-code-review-only-catches-half-of-your-bugs/)
- [Laurie Voss, The Death of the Code Review: What the Data Actually Says](https://www.youtube.com/watch?v=_mi3alkqy4s)
- [John Kim, How I Review AI Code](https://www.youtube.com/watch?v=b2QkhmQ0sT0)
- [Kun Chen, L8 Principal's Agentic Engineering Workflow](https://www.youtube.com/watch?v=JEQh3W0gfOY)
- [Matt Pocock, Fixing the PR Bottleneck](https://www.youtube.com/watch?v=LlgiOCmFG_w)
- [Addy Osmani, Morgan Westlee Lunt & Qing Wang, How the Claude Code Team Does Code Review](https://claude.dev/blog/solving-code-review-at-anthropic/)
- [Nick Nisi, I Made a Fruit Fly Review My Code](https://www.youtube.com/watch?v=geMAe-PLJF4)
- [Corey Gallon, I Believe Everything My Agents Tell Me: Three Levels of Review](https://www.youtube.com/watch?v=dYh7TQkuYFE)
- [Manish Kapur & Addy Osmani, How Anthropic Merges 30,000 Code Changes a Day](https://www.youtube.com/watch?v=AnsXWH2_EBU)
- [Guillermo Rauch × David Ondrej — interview on agentic engineering](https://www.youtube.com/watch?v=WeiB_gLOdQE)
- [Gabriel Spencer-Harper, Why AI Didn't Actually Make You Ship Faster](https://www.youtube.com/watch?v=HLTa7Vcs4X0)
- [Gautam Korlam, Rebuilding CI for AI-Scale Commit Volume](https://www.youtube.com/watch?v=7b9tov5Uc5s)
- [Dex Horthy, Harness Engineering Is Not Enough: Why Software Factories Fail](https://www.youtube.com/watch?v=Ib5GBkD555M)
- [Dex Horthy × Patrick Akil, Why Coding Agents Keep Making Your Codebase Worse](https://www.youtube.com/watch?v=vKT59uoByXc)
- [DHH × Lex Fridman — interview (original episode linked in the Wiki's secondary account)](https://www.youtube.com/watch?v=NYFGCESmikA)
