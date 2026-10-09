# MatchWise: An Uncertainty-Aware Policy for Sequential Reciprocal Matching

**Research Proposal — Round 1**  
**Hackathon:** Romeo & Juliet × IIT Madras  
**Team project:** MatchWise

## 1. Abstract

Sequential reciprocal matching requires more than identifying individually promising pairs. A successful policy must account for both participants' constraints, incomplete information, limited clarification resources, changing availability, and feedback that becomes observable only after decisions have been made. An introduction that appears attractive from one person's perspective may violate the other person's preferences or practical constraints. Similarly, choosing one pair can eliminate opportunities for other participants, making allocation a pool-wide decision rather than an independent ranking exercise.

We propose **MatchWise**, an uncertainty-aware sequential matching policy designed to combine reciprocal feasibility checks, pool-wide allocation, selective clarification, and feedback-aware decision-making. The policy will treat hard constraints as non-negotiable, use observed soft preferences to estimate pair utility, and consider the value of additional information before spending the clarification budget. It will operate exclusively on observable simulator state and policy-owned memory.

We hypothesise that combining global allocation with selective clarification can improve Mutual Second-Meeting Intention (MSMI) per 100 arrived members relative to the supplied greedy, no-clarification, and random-feasible baselines. This hypothesis will be tested through controlled simulator experiments using identical seeds and all six public scenario families. Ablation studies will isolate the contributions of allocation and clarification. The proposed approach is a research direction; its performance advantage has not yet been established.

## 2. Problem Interpretation

The challenge is to make sequential introduction decisions in a changing population where preferences are reciprocal, observations may be incomplete, and outcomes are delayed.

At each decision step, a policy must determine:

1. Which available participants require clarification before their eligibility can be established.
2. Which pairs satisfy both participants' hard constraints and current operational requirements.
3. Which feasible pairs should be introduced together, ensuring that no participant appears more than once in the same batch.
4. When it is preferable to wait rather than make an introduction.
5. How newly observed answers and feedback should influence subsequent decisions.

The problem has three interconnected dimensions.

**Reciprocity:** Both participants must satisfy the other's explicit eligibility requirements. A high pair score cannot compensate for a hard-constraint violation.

**Allocation:** The best set of introductions may differ from the set obtained by repeatedly choosing the highest-scoring individual pair. Pair choices compete for the same participants and affect the remaining pool.

**Sequential uncertainty:** The policy must distinguish genuinely incompatible pairs from pairs whose eligibility remains unknown. Clarification consumes a limited budget, while waiting can preserve future opportunities but may leave participants unserved.

The principal evaluation outcome is MSMI per 100 arrived members. An introduction qualifies only when a date occurs within 30 simulation days of assignment and both participants record a second-meeting intention of Yes within three simulation days of the date. This is a deliberately downstream outcome: assignment, mutual acceptance, a date, and mutual second-meeting intention are distinct events.

The simulator uses synthetic participants and invented outcomes. Performance on these episodes will be interpreted only as evidence about policy behaviour within the specified simulation environment, not as evidence of real-world relationship compatibility.

## 3. Research Hypothesis and Objectives

### 3.1 Primary hypothesis

**H1:** A policy that combines reciprocal hard-constraint enforcement, global non-overlapping allocation, and selective clarification will achieve a higher mean MSMI per 100 arrived members than the supplied greedy, no-clarification, and random-feasible baselines when evaluated on identical seeds and scenario families.

This is a hypothesis to test, not an established result.

### 3.2 Secondary hypotheses

- **H2 — Global allocation:** Selecting a set of pairs collectively can improve total estimated pair utility compared with greedy pair selection, particularly when several promising pairs compete for the same participants.
- **H3 — Selective clarification:** Prioritising questions that can resolve important eligibility uncertainty or change an allocation decision can use the clarification budget more effectively than indiscriminately asking every eligible participant with missing information.
- **H4 — Robustness:** Explicit handling of missing observations and delayed feedback will reduce invalid decisions and help the policy remain operational across sparse, cold-start, delayed, and changing-condition scenarios.

### 3.3 Research objectives

1. Preserve all reciprocal hard constraints and simulator action requirements.
2. Develop an interpretable pair-utility function based on observable soft preferences and interaction history.
3. Investigate pool-wide allocation under the non-overlap constraint.
4. Design a budget-aware clarification strategy.
5. Evaluate the proposed components independently and jointly.
6. Document failure cases, uncertainty, reproducibility, and limitations.

## 4. Reciprocal Feasibility and Pair Utility

### 4.1 Hard-constraint enforcement

MatchWise will use the supplied eligibility semantics as the authoritative feasibility contract. Before considering a pair for allocation, it will check both directions of eligibility and the operational requirements of the simulator.

The checks include:

- Each participant's acceptable age range and explicitly stated genders to meet.
- Compatible relationship structures under the development contract.
- Smoking preferences and children-related constraints.
- Reciprocal acceptable-zone requirements.
- At least one overlapping schedule slot.
- Current availability and pool membership.
- Absence of a previous introduction between the pair.
- Absence of outstanding introductions and duplicate participants within the proposed batch.

A known hard-constraint failure will exclude the pair even when other information is missing. If a decisive hard constraint is unknown, the pair will remain ineligible for introduction until sufficient information becomes observable. A soft-preference score will never override a hard constraint.

### 4.2 Missing information and eligibility states

The policy will distinguish among observed answers, fields that have not been asked, and answers that have been declined. A null value will not be interpreted as zero, rejection, or disinterest.

The supplied eligibility function distinguishes three outcomes:

- **Feasible:** all required eligibility checks pass.
- **Infeasible:** at least one known hard constraint fails.
- **Needs clarification:** eligibility cannot be established because required hard information is missing.

Declined answers will remain unavailable. MatchWise will not attempt to reconstruct them from identifiers, simulator generation patterns, future feedback, or other hidden state.

### 4.3 Soft-preference utility

Once reciprocal feasibility is established, MatchWise will use a transparent utility score to compare eligible pairs. Candidate signals may include agreement on relationship goals, relationship pace, lifestyle, conversation preferences, emotional availability, available space for a relationship, and relocation preferences where relevant.

The initial utility function will favour interpretable agreement and compatibility signals rather than an unnecessarily complex predictive model. Hard constraints and soft utility will remain separate.

Weights will be treated as design choices to evaluate, not as validated psychological coefficients. The utility score will rank feasible alternatives; it will not be presented as a probability that a relationship will succeed.

## 5. Proposed Allocation Method

### 5.1 General graph representation

At each matching phase, the available participants in a pool will be represented as vertices in an undirected graph. An edge will connect two participants only when their pair is currently feasible and has not already been introduced.

Each feasible edge will receive a utility weight derived from the observed soft-preference information and any permitted interaction-history features.

The allocation problem is to select a set of edges with no shared vertices. This is a general-graph matching problem, rather than necessarily a bipartite assignment problem, because the supplied environment does not require the participant population to be divided into two fixed sides.

### 5.2 Global allocation

The proposed approach will investigate maximum-weight matching over the currently feasible graph. The objective is to select a compatible set of non-overlapping pairs that maximises the total estimated utility for the current batch.

This addresses a limitation of greedy allocation: choosing the strongest individual edge first can prevent the selection of a better combination of other edges.

The global allocation objective will remain subordinate to feasibility. The policy may return an empty pair list when no eligible allocation is available or when waiting is preferable under the chosen decision rule.

### 5.3 Sequential decisions

Allocation will be recomputed from the currently observable state as the episode progresses. The policy will account for new arrivals, changes in availability, outstanding introductions, and feedback received since previous decisions.

Waiting will be treated as a valid action, not automatically as failure. However, excessive waiting may reduce coverage and leave participants unserved. The experiments will therefore examine the trade-off between pair utility, outcome rate, coverage, and waiting time.

## 6. Clarification Policy

### 6.1 Budget and action constraints

The simulator provides 12 clarification-budget units per day. Requesting the hard-constraint bundle for one participant costs three units, while requesting one named soft field costs one unit. Clarification is immediate in the supplied simulator, and declined answers remain unknown.

MatchWise will respect the budget and request information only for currently available participants. After clarification, decisions will use the refreshed observable state.

### 6.2 Proposed strategy

The clarification policy will prioritise unresolved hard constraints that prevent potentially useful candidates from becoming eligible. Among permissible questions, it will investigate whether selective soft-field clarification can improve the allocation decision enough to justify its cost.

The intended strategy is:

1. Identify missing hard information that blocks potentially useful candidate pairs.
2. Exclude questions about participants who are unavailable or whose known hard constraints already rule out relevant candidates.
3. Prioritise clarification requests according to their potential to change feasibility or improve the available allocation.
4. Consider soft-field questions only when the expected decision benefit justifies their budget cost.
5. Respect the daily budget and preserve declined answers as unknown.
6. Recompute feasibility and allocation after answers become observable.

A value-of-information-inspired heuristic is a candidate approach for ranking clarification requests. It will be evaluated empirically rather than assumed to be optimal.

### 6.3 Clarification ablation

To measure the contribution of clarification, the proposed policy will be compared with an otherwise equivalent variant that makes no clarification requests. Additional experiments may compare the selective strategy with the supplied baseline clarification procedure.

This design will help determine whether improvements arise from the allocation method, the clarification strategy, or their interaction.

## 7. Handling Missing and Delayed Data

### 7.1 Missing and declined answers

Unknown values will remain explicitly unknown. Declined fields will not be repeatedly queried as though they were unanswered, and a missing field will not be treated as a negative preference.

A pair blocked by missing hard information will not be introduced merely because its observed soft-preference score is high.

### 7.2 Delayed feedback and censoring

Feedback will be used only after it becomes observable. The policy will not access future events, hidden participant attributes, future arrivals, latent simulator outcomes, or simulator internals.

The following distinctions will be preserved:

- No recorded introduction response by its deadline is not proof of personal dislike.
- A pair that has not had a date has no observed second-meeting preference.
- A date without a recorded second-meeting response is not automatically a negative preference label.
- An outcome whose observation window has not matured will be treated as unresolved rather than automatically unsuccessful.

Evaluation will follow the supplied outcome definition and follow-up period so that recently assigned introductions are not unfairly labelled as failures.

### 7.3 Selective feedback

The policy observes outcomes only for pairs it chooses to introduce. Consequently, the available feedback is selective rather than a complete record of every possible pair.

MatchWise will acknowledge this selection effect when interpreting results. It will not claim unbiased offline evaluation using inverse-propensity methods because historical logging propensities are not supplied. Controlled simulator rollouts will be the primary evaluation method.

## 8. Optional Probability Analysis

Probability estimation is optional under the challenge specification and is not required for the executable policy or primary ranking.

The initial MatchWise proposal will therefore prioritise feasibility, allocation, and clarification decisions rather than make unvalidated probability predictions.

If a later experiment introduces probability estimates, it will distinguish three targets:

1. The probability that participant A accepts participant B.
2. The probability that participant B accepts participant A.
3. The probability of mutual acceptance.

These are distinct outcomes. Any model will define the target, observation window, training data, and evaluation procedure. The joint probability will not be obtained by multiplying directional probabilities without explicitly justifying the required independence assumption. If probabilities are reported, their calibration and uncertainty will be evaluated on held-out observations.

## 9. Experimental Design

### 9.1 Baselines

MatchWise will be compared against all three supplied baselines:

- **Greedy:** the reference clarification procedure followed by greedy allocation based on observed soft-preference agreement.
- **No-clarification:** a baseline that makes no clarification requests and uses the corresponding matching procedure.
- **Random-feasible:** a baseline that randomises feasible candidate edges and selects a non-overlapping set.

All methods will be evaluated with identical seeds and scenario variants. Baseline comparisons will use the supplied evaluator and will preserve the simulator's feasibility and outcome definitions.

### 9.2 Public scenario families

Experiments will cover all six public scenario families:

- **Development:** the standard public development setting.
- **Sparse:** reduced candidate availability due to sparse geography.
- **Cold start:** limited information and interaction history.
- **Delayed:** delayed dates and downstream feedback.
- **Shift:** changed outcome-weight conditions.
- **Drift:** changing response conditions.

The experiments will investigate whether each design choice behaves consistently across these settings rather than relying on a single favourable episode.

### 9.3 Seeds and reproducibility

The initial public comparison will use seeds 101, 102, and 103 across all six variants. Each policy will be evaluated using the same settings, and the full machine-readable episode results will be retained.

The initial runs are development experiments, not the private competition evaluation. Additional seeds may be used where feasible to assess variation and reduce dependence on a small sample of synthetic episodes.

Policy memory will start from its declared initial state for every independent episode. Any randomisation used by MatchWise will be controlled and documented.

### 9.4 Evaluation metrics

The primary metric will be the equally weighted mean of scenario-level MSMI per 100 arrived members, as defined by the supplied evaluator.

Secondary metrics will include:

- Distinct-member introduction coverage.
- Mutual acceptances per 100 arrived members.
- Number of assignments, mutual acceptances, dates, and qualifying MSMI outcomes.
- First-introduction waiting times and the number of unserved members.
- Clarification cost and the amount of missing feedback.
- Runtime and valid-episode rate.

Results will be reported at scenario level and across seeds. Outcome counts and uncertainty will be discussed explicitly because qualifying events may be rare in small synthetic episodes. The primary score will not be replaced with a more favourable metric.

## 10. Planned Ablations

At least one hypothesis-driven ablation will test a causal design claim within the simulator. The planned studies are:

**Ablation A — Global versus greedy allocation:** Replace global maximum-weight matching with greedy pair selection while keeping the remaining policy components unchanged. This will test whether pool-wide allocation improves the primary outcome and how it affects coverage and waiting time.

**Ablation B — Selective clarification versus no clarification:** Disable clarification while retaining the same allocation procedure. This will measure the contribution of acquiring missing information under the simulator's budget and disclosure rules.

**Ablation C — Utility design:** Where time permits, compare the initial interpretable utility with a simpler agreement score. This will examine whether added utility complexity contributes measurable benefit.

The comparisons will use identical seeds and scenarios. Each ablation will change one main component at a time wherever possible. Results will be interpreted cautiously when the number of qualifying outcomes is small.

## 11. Expected Failure Cases and Risk Analysis

### Sparse candidate supply

Some participants may have no feasible partner because of reciprocal preferences, geography, schedule, or other hard constraints. No allocation algorithm can create a valid edge where none exists. MatchWise will permit empty batches and report unserved participants rather than relaxing hard constraints.

### Missing or declined hard information

A potentially useful pair may remain unresolved because a required answer is missing or declined. Clarification may help only when the information is disclosed and the resulting pair remains feasible. The policy must accept that some candidates cannot be safely introduced.

### Cold starts

New participants may have little observable history. The initial approach will rely on explicit constraints and interpretable observed preferences rather than treating absent feedback as negative evidence. Its utility estimates may be less informative when little information is available.

### Delayed and missing feedback

Downstream outcomes may not be observable by the end of an episode's decision period. This can make apparent performance noisy and can bias interpretations if unresolved cases are treated as failures. The prescribed follow-up and outcome windows will be respected.

### Distribution shift and drift

Utility weights that work under one synthetic scenario may not work under another. The policy may lose performance when response conditions or outcome-generating assumptions change. Scenario-level results and ablations will be used to identify such weaknesses.

### Clarification budget exhaustion

Questions may consume budget without changing eligibility or allocation. The selective clarification heuristic may prioritise the wrong question or overestimate the usefulness of information. Clarification cost will therefore be reported alongside primary outcomes.

### Utility misspecification

Observed agreement on soft fields may not translate into the simulator's downstream outcome. An interpretable score can still be poorly aligned with MSMI. The experiments will test this assumption rather than treating the score as a validated measure of compatibility.

### Over-allocation and excessive waiting

A policy focused on high utility may serve too few participants, while a policy focused on coverage may make lower-value introductions. The study will examine the trade-off between MSMI, coverage, and waiting times.

## 12. Preliminary Evidence and Interpretation

Initial experiments have been conducted using the supplied public evaluator with seeds 101, 102, and 103 across the six public scenario variants. The following primary scores were observed for the supplied baselines:

| Baseline | Preliminary primary score |
|---|---:|
| Greedy | 0.389 |
| Random-feasible | 0.278 |
| No-clarification | 0.139 |

These values are preliminary observations from public synthetic simulator runs. They are not results from an implemented MatchWise policy, do not establish statistical significance, and do not represent the private competition leaderboard.

The initial comparison motivates two questions for further research: whether clarification improves downstream outcomes under the tested conditions, and whether a more deliberate allocation method can improve the selection of non-overlapping pairs. The proposed experiments are intended to investigate these questions while accounting for coverage, clarification cost, scenario variation, and low outcome counts.

No performance advantage for the proposed MatchWise policy is claimed at this stage.

## 13. Reproducibility, Data Provenance, and Implementation Boundaries

The research uses the public participant kit and its synthetic dataset. No real personal records, production database credentials, or external relationship data are required.

The executable policy will follow the supplied versioned JSON interface. It will consume only observable state, permitted clarification results, observable feedback, and explicitly carried policy memory. It will return valid clarification requests or feasible non-overlapping pairs and may return an empty list when appropriate.

The evaluation environment is designed for offline CPU inference. Any additional dependency, model, training asset, or external resource introduced during development will be documented with its provenance and licence. Random seeds and evaluation commands will be recorded to support reproducibility.

The current repository contains the starter policy and preliminary baseline results. The global allocation and selective clarification components described in this note remain proposed work until implemented and evaluated. Subsequent results will be reported with their exact commands, seeds, scenarios, and limitations.

## 14. Limitations and Responsible Interpretation

The simulator's participants, answers, interactions, and outcomes are invented. Its fictional geography, immediate clarification mechanism, whole-day timing, and outcome-generation assumptions are simplifications of an introduction process.

A higher MSMI score in this environment would establish only that a policy performs better under the tested simulation conditions. It would not demonstrate that the policy can predict real relationships, infer private preferences, or improve outcomes for real people.

Any later authorised product integration would require an explicit adapter, validation of data semantics, privacy safeguards, human review, and separate permission for product use. The policy should never bypass a participant's stated constraints or infer undisclosed preferences.

The research will therefore prioritise constraint compliance, transparent decision logic, controlled evaluation, and explicit limitations over unsupported claims about human compatibility.

## 15. References
