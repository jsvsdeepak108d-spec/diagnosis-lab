# Diagnosis Lab

**Human, AI or both: when is an expert's time worth adding to an AI's diagnosis?**

100xEngineers Cohort 7 capstone, Collaborative Intelligence track. Built by Deepak, with Claude (Anthropic) as build collaborator.

Diagnosis Lab tests one repeated task: explaining *why* a real business event happened (a profit warning, a layoff, a share crash). It runs three setups on the same cases (me alone, AI alone, and me plus AI as a pair), plus a router that decides which setup a case should get.

## Result in one table

20 real events from July to October 2026, all after the AI's training data ends. Score = coverage: the share of causes the company or reputable press stated that appear in the answer's top 3.

| Setup | Cases | Stated causes found | Same cases, AI alone | My minutes |
| --- | --- | --- | --- | --- |
| AI alone | 20 | 69% | — | 0 |
| Me alone | 10 | 35% | 58% | 33.6 |
| Pair (me + AI) | 10 | 70% | 80% | 35.8 |
| Routed system (estimate) | 19 | 68% | — | 13.5 |

**Kill condition, fixed before scoring:** the pair must beat AI alone by 15 points on the same cases. It came in 10 points *below*. The pair tied AI alone on 8 cases, lost 2 and won none; both losses came from my "kill" decisions removing correct causes. The router caught 1 of the 3 cases where the AI missed most causes.

![Coverage on the same cases](docs/images/coverage-chart.png)

## How it works

![Architecture](docs/images/architecture.png)

1. **Case card:** facts known before the event only (size, industry, geography, ownership, finances, technology context, recent context, one shared macro paragraph).
2. **Three AI attempts:** the same diagnostician prompt, run independently.
3. **Router (plain code, not AI):** agreement between the attempts plus case flags (private, regulated, unusual finances, new leader) pick a lane: AI only, pair, or human only.
4. **Pair lane:** the AI lists 10 unranked candidate causes; I keep or kill each and may add my own (3 minutes plus 1 minute overtime); the AI ranks the survivors. My additions are guaranteed a top-3 place (rules v1.1).
5. **Logging and blind scoring:** every answer, decision and second is stored; an AI scorer matches answers to the stated causes without knowing which setup wrote them. I re-checked 5 of its decisions and agreed with all 5.

![Pair screen, Volkswagen case](docs/images/pair-screen.jpg)

### Where the agent is

| Part | Kind |
| --- | --- |
| Diagnostician: reads the card, returns top 3 causes, decides whether to call the analyst | Agent |
| Context analyst: answers one question in its own context, using a calculator | Sub-agent (called 0 of 69 times) |
| Calculator | Tool (plain code) |
| Candidate generator, ranker, scorer | Single AI calls |
| Router, timer, random assignment, scoring maths | Plain code |

## Repository contents

| Path | What it is |
| --- | --- |
| `app/index.html` | The whole application: UI, all prompts, router, guardrails, scoring and results, in one file |
| `PROMPTS.md` | Every prompt and rule, verbatim, in readable form |
| `data/cases.json` | The 26 case cards (3 practice, 23 test), pre-event facts only |
| `data/answer_keys.json` | The stated causes and source links for each case |
| `data/results_by_case.csv` | One row per test case: setup, coverage, AI-alone coverage, minutes, rules version |
| `data/run_log.json` | Full export: every run, every AI attempt, every score, scorer checks and the random schedule |
| `docs/images/` | Architecture diagram, results chart and screenshots |

## How to run it

The app is a claude.ai Artifact. It uses two platform capabilities: calls to Claude on the viewer's own plan, and a small database for cases, runs and scores. Opened outside claude.ai it shows a message that it needs Claude.

To run your own copy: in a claude.ai chat, attach `app/index.html` and ask Claude to publish it as an Artifact with the `db` and `sample` capabilities, then load `data/cases.json` and `data/answer_keys.json` into the `cases` and `zkeys` collections.

## Guardrails

- Case cards contain only facts known before the event.
- Every AI reply uses a fixed JSON format; one retry, then the failure is logged.
- The AI may answer "not enough information" instead of guessing.
- Time saved is never reported without the router's catch rate (Goodhart's law).
- The scorer never knows which setup wrote an answer; answer keys are read only at scoring time.

## Honest limits

- I was the builder and the only human tested, which breaks the brief's "not you" rule. Hidden keys, random assignment, frozen rules and blind scoring limit the bias; the result went against my own hypothesis.
- 20 test runs rather than 30: read the direction, not the decimals.
- Answer keys are *stated* causes; a sharper read the company never said out loud scores nothing.
- One case (JLR) was hand-scored after the automatic scorer failed twice; one scorer inconsistency (Fluence) is left as scored.
- Claude wrote the case cards after seeing the stated causes, using a fixed template.

## What I'd build next

- Let the expert add causes, never delete the AI's.
- Route on novelty (new technology, new business models, post-training-cut-off shifts), not on agreement between attempts.
- Test a second practitioner who isn't the builder.
- Package it as a Claude skill: give it a company and an event, and it researches, builds the card, routes, and asks for review only when needed.
