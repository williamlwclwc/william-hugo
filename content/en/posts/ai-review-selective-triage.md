---
title: "My Thoughts on AI Code Review: Selective Review, with AI Doing the Triage"
date: 2026-11-08
draft: false
description: "Review intent and design early, ask AI for test evidence and risk assessments, then focus human review where it matters most."
tags: ["ai-code-review", "agentic-engineering", "software-engineering", "risk-based-review"]
categories: ["AI Engineering"]
---

Previous in this series: [What Code Review Looks Like After AI Starts Writing the Code: The Industry in 2026]({{< relref "ai-code-review-2026.md" >}})

*The previous post mapped the state of AI code review across the industry. This one describes my own approach and judgment today. It is a baseline, rather than a settled conclusion: I want to record what I think in 2026 and return to it a few years from now.*

## The central question: where to review

Let me start by ruling out the two extremes.

**We cannot keep trying to understand every block of code the way we used to.** There is simply too much of it. AI can generate in a day what would take a person several days to read carefully. The numbers no longer work.

**We cannot stop reviewing altogether either.** Things can get out of control. In the examples from the previous post, each PR looked reasonable on its own, but the combined result damaged the architecture. Nearly all the tests passed, yet the code contained more than ten thousand unsafe blocks.

That makes **selective review** the real problem: human attention is limited and needs to go where it matters most. The hard part is deciding which areas deserve that attention.

My answer has three steps: move review earlier, let AI do the initial triage, and let people choose what to examine based on the results.

## 1. Review intent and design before implementation

Do not wait until the code exists to begin reviewing.

Code is the final artifact, and it is also the largest and hardest layer to read. Many consequential mistakes, such as misunderstood requirements, a poor choice of approach, or incorrectly drawn boundaries, happen at the intent and design stages. By the time they reach the code, they are buried in thousands of lines of implementation and become harder to spot.

Reviewing intent and design means reviewing much less material, while changes are still cheap. Done well, this gives the later code review a reference point: is the implementation doing what we agreed it should do?

Moving review earlier is no cure-all. It can protect architecture to some extent, but I think architecture can gradually deteriorate even without AI as teams grow and iteration speeds up. At least at my company, when a system becomes too difficult to maintain, we usually undertake a major architectural migration or even build a replacement service. AI may accelerate that process, but it is not the only cause.

## 2. Let AI do the initial triage

Once the code is written, I would have AI make an initial assessment before asking a person to dive in. I want two kinds of output.

### Tests that can disprove correctness

Simple unit tests are not enough; they can be misleading. Tests written by an agent may merely repeat the implementation's logic, or may be incapable of failing. A green suite means the tests have been satisfied. It does not establish that the code is correct.

What I want is:

- **Mutation tests:** deliberately break the code and see whether the tests fail. If the tests cannot catch those mutations, they are going through the motions.
- **End-to-end tests with evidence:** exercise the workflow from the user's perspective and leave a record someone can inspect, beyond a statement that the tests passed.

AI should not be the sole judge of those results. For end-to-end tests in particular, a person needs to at least scan the evidence, such as a recording or screenshots.

### A risk assessment

Ask AI to describe the risk of the change:

- **Blast radius:** how widely a failure could spread, and which core paths, shared state, or external interfaces the change touches.
- **Ease of rollback:** whether a feature flag covers it and whether it can be switched off quickly if something goes wrong.
- **High-risk areas:** the specific places that need additional human review, and why.

## 3. Let people choose the depth of review

With test evidence and a risk assessment in hand, people can decide where to look and how deeply.

Core paths, changes that are hard to roll back, and anything flagged as high risk deserve real attention. Whether low-risk changes can receive a quick scan or go straight through depends on the business. In a fast-moving environment where incidents have limited consequences, there is room to be more permissive. In payments or other areas where a mistake can have a major impact, even an apparently low-risk change deserves more caution.

There is no precise formula. The goal is to **find a workable balance**: keep people from drowning in code while catching the changes that are truly dangerous.

This mechanism can fail too. AI triage can label a high-risk change as low risk. If AI writes the code, writes the tests, and performs the triage, those stages may share the same blind spots. I would respond in two ways:

- **Set the strictness of the rules according to the consequences.** Rules such as “automatic classification can only raise the risk level” and “sensitive paths require a human” should be as strict as the potential impact warrants. A business with limited downside and fast rollback can relax them somewhat; other situations call for tighter rules.
- **Keep improving the team's review skill.** There will always be blind spots. Each missed issue should feed back into the team's review rules so that the next round of triage is better.

## 4. The author must participate, but should not count as a reviewer

The previous post described a disagreement: should the person who prompts and directs the coding agent count as a reviewer? Anthropic's Claude Code team says yes, on the principle that whoever initiates a change is responsible for it. Gitar/Sonar's Korlam says no, because that person shares the agent's context.

My view is: **the author must participate in review, but should not count as a reviewer.**

**Why should the author not count?** I have two reasons.

First, **ultimate responsibility belongs to the author, rather than the reviewer.** Since the author already bears that responsibility, the required approval should come from someone else. Interestingly, Anthropic uses the same principle of responsibility to reach the opposite conclusion: because the author is responsible, they count toward the review requirement. I think the author's existing responsibility is precisely why their approval cannot serve as an additional check.

Second, **authors are biased when reviewing their own code and can overlook mistakes.** This is the same reason the review agent should be separate from the implementation agent. More teams are accepting that AI should not review its own work because its generation process anchors its judgment. People have the same problem.

**Why must the author still participate?** The author has much more context than the reviewer: why the design looks this way, which alternatives were tried, and which trade-offs were deliberate. The author can answer questions quickly and must respond to and implement review feedback. Their participation makes review efficient.

The author's role is therefore to read the evidence, answer questions, respond to feedback, and take responsibility for the result, without occupying an approval slot.

## 5. Simple to describe, harder to put into practice

“Let AI do the triage” is easy to say. Implementing it is harder. At a minimum, it requires:

- **Skills and instructions maintained by the team:** what counts as high risk in this repository, which paths are sensitive, and what the team's review standards are. A generic AI reviewer does not know these things.
- **Infrastructure AI can operate autonomously:** the ability to run CI/CD and even bring up an environment for end-to-end tests. Without that, AI can only read code and guess at the result; it cannot produce evidence.

That does not mean waiting until everything is ready. I would start with the lowest barriers and the most immediate returns:

1. **Skills and instructions:** write down the team's review experience so AI can use it.
2. **Risk assessments:** ask AI to provide a risk level and a rationale for each PR.
3. **Blast radius:** ask AI to identify the scope of the change and the critical paths it affects.

These steps do not require a large infrastructure overhaul, yet they can immediately help people decide where to focus. Mutation testing and autonomous end-to-end testing matter too: without them, AI triage lacks truly credible evidence. They are simply harder to implement all at once and need to be built progressively on top of those foundations.

## Closing thoughts

My current thinking is:

- The core question in code review is **where to review**.
- Review should **move earlier**, into intent and design.
- **AI should triage first:** provide test evidence capable of disproving correctness, assess blast radius and rollback difficulty, and identify high-risk areas.
- **People should review selectively based on that triage**, balancing efficiency and risk.
- **The author must participate, but should not count as a reviewer.**
- Implementation takes work, but teams can start with **skills, instructions, risk assessments, and blast radius**.

This is what I think in 2026. Tools and models change every few months. When I return to this post a few years from now, what I most want to know is which parts I got wrong.

## References

These are the practitioner accounts this post draws on or discusses. The position that authors must participate without filling an approval slot, and the proposed triage workflow, are my own judgments rather than validated industry conclusions.

- [John Kim, How I Review AI Code](https://www.youtube.com/watch?v=b2QkhmQ0sT0)
- [Matt Pocock, Fixing the PR Bottleneck](https://www.youtube.com/watch?v=LlgiOCmFG_w)
- [Addy Osmani, Morgan Westlee Lunt & Qing Wang, How the Claude Code Team Does Code Review](https://claude.dev/blog/solving-code-review-at-anthropic/)
- [Corey Gallon, I Believe Everything My Agents Tell Me: Three Levels of Review](https://www.youtube.com/watch?v=dYh7TQkuYFE)
- [Gautam Korlam, Rebuilding CI for AI-Scale Commit Volume](https://www.youtube.com/watch?v=7b9tov5Uc5s)
- [Dex Horthy × Patrick Akil, Why Coding Agents Keep Making Your Codebase Worse](https://www.youtube.com/watch?v=vKT59uoByXc)
