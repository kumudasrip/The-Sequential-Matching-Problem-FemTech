# MatchWise: Decision-Aware Clarification for Sequential Reciprocal Matching

**Round 1 Research Proposal**  
**Hackathon:** Romeo & Juliet × IIT Madras  
**Team project:** MatchWise

## Abstract

The One Introduction Problem is a sequential decision problem in which a policy must decide which information to request, which reciprocal pairs to introduce, and when to wait. Decisions must respect strict hard constraints, a limited daily clarification budget, non-overlapping introductions, changing availability, and feedback that may be missing or delayed. Selecting individually attractive pairs is insufficient because one introduction can prevent other introductions in the same batch.

We propose **MatchWise**, a policy centred on two connected ideas: (1) decision-aware clarification, which values a question by its expected effect on the best feasible global matching, and (2) global maximum-weight matching, which selects a non-overlapping batch from the currently feasible candidate graph. Reciprocal hard-constraint checks are an invariant, not a score that can be traded away. Pair weights will initially use a transparent soft-preference utility; outcome-aware estimates of Mutual Second-Meeting Intention (MSMI) may be added only if they can be learned and evaluated reliably. Feedback adaptation and time-aware waiting will be treated as optional extensions.

Our primary hypothesis is that decision-aware clarification combined with global allocation can improve the benchmark's primary MSMI score over the supplied greedy, no-clarification, and random-feasible baselines. We will test this through paired simulator experiments using common seeds across all six public scenario families. Ablations will isolate the effects of allocation and clarification. No performance improvement is claimed before these experiments are completed.

## 1. Problem Interpretation

The policy operates over a changing pool of synthetic adults. At each decision step, it must choose clarifications within a limited budget, identify pairs that satisfy both participants' hard constraints, propose a valid non-overlapping batch, and decide whether to act or wait. The consequences of a decision can remain unknown for several days.

The problem combines four challenges:

1. **Reciprocal feasibility:** both participants must satisfy the other's stated hard constraints.
2. **Global allocation:** pairs compete for the same people, so selecting the highest-scoring edge first can prevent a better total batch.
3. **Information acquisition:** missing hard information can block otherwise promising pairs, but clarification has a cost and a fixed daily budget.
4. **Sequential uncertainty:** feedback is observed only for selected pairs and may not have matured by the current decision time.

The benchmark uses a 60-day decision horizon followed by 40 days of outcome follow-up. The daily clarification budget is 12 units; the `constraints` bundle costs 3 units and a named soft-field question costs 1 unit. Clarification is immediate in the supplied specification. Each person may have at most one outstanding introduction, and a pair may not be introduced more than once in an episode. The official participant specification and interface are authoritative for all action and scoring details.

### 1.1 Primary objective

The primary metric is **MSMI per 100 arrived members**. An introduction qualifies only when:

1. both participants return a positive introduction response;
2. a date occurs within 30 simulation days of assignment; and
3. both participants report “yes” to a second meeting within three simulation days of the date.

Mutual acceptance, a date, and second-meeting intention are intermediate stages; none is a substitute for the full MSMI event. Unresolved or immature outcomes must not automatically be labelled failures.

The official primary score equally weights the mean MSMI score of each of the six public scenario families: development/standard, sparse, cold start, delayed, shift, and drift. The specified tie-breakers are distinct-member coverage, mutual acceptances per 100 arrived, lower clarification cost, and lower inference time, in that order. Invalid actions or runtime failures are unacceptable.

## 2. Research Question and Hypotheses

### Research question

**Does valuing clarification by its expected marginal improvement to global reciprocal matching, combined with maximum-weight batch allocation, improve the official MSMI score over the supplied greedy baseline?**

### Primary hypothesis

**H1 — Decision-aware global matching:** MatchWise will achieve a higher mean MSMI per 100 arrived members than the supplied greedy baseline when all methods are evaluated on identical seeds and scenario families.

### Component hypotheses

- **H2 — Global allocation:** Maximum-weight matching will improve the total utility of the selected batch relative to greedy selection. Whether this improves MSMI depends on how well the edge weights represent downstream outcomes.
- **H3 — Selective clarification:** Prioritising questions that can resolve hard-constraint uncertainty or materially change the best batch will use the daily clarification budget more effectively than asking without considering decision impact.
- **H4 — Outcome-aware utility (exploratory):** A reliable estimate of expected MSMI may rank feasible pairs more usefully than a raw soft-field agreement score. This component will be retained only if the available data supports meaningful estimates.
- **H5 — Feedback and waiting (exploratory):** Conservative updates from mature feedback or a time-aware wait rule may help in some scenarios, but may also add noise, cost, or delay. Neither is assumed to improve performance in advance.

Each hypothesis is falsifiable. If a component adds complexity without reliable gains, the simpler variant will be preferred.

## 3. Reciprocal Feasibility: Hard Constraints Before Scoring

MatchWise will use the supplied eligibility logic as the authoritative feasibility contract. No utility score, probability estimate, or coverage preference may override a hard-constraint violation.

For each candidate pair, the policy will respect the official rules for:

- reciprocal age-range and `who_to_meet` preferences;
- relationship structure;
- smoking and children-related constraints;
- reciprocal acceptable-zone requirements;
- overlapping schedules;
- current availability and pool membership;
- outstanding introductions; and
- previously introduced pairs.

Each pair will be classified as:

- **Feasible:** all required eligibility checks are known and pass.
- **Infeasible:** at least one known hard constraint fails.
- **Needs clarification:** no known hard constraint has failed, but required hard information is missing.

A known failure takes precedence over missing information. A pair with unresolved required hard information will not be introduced until eligibility can be established. Declined fields remain unknown; the policy will not infer their values or ask for them again.

### 3.1 Candidate graph

At time \(t\), available members form the vertices of a graph. An undirected edge is included only for a currently feasible pair that has not already been introduced and has no outstanding assignment. The graph is generally non-bipartite: the challenge does not require a fixed proposer/receiver partition.

## 4. Proposed Allocation Method: Global Maximum-Weight Matching

For each feasible edge \((i,j)\), MatchWise will assign a weight \(w_{ij}(t)\) based only on information observable at the current decision time. The initial implementation will use an interpretable soft-preference utility based on observed fields such as relationship goals, pace, lifestyle, conversation preferences, emotional availability, space for a relationship, and relocation preferences where relevant.

The policy will select a set of edges \(M_t\) that maximises total edge utility, subject to every participant appearing at most once in the batch:

\[
\max_{M_t \subseteq E_t} \sum_{(i,j)\in M_t} w_{ij}(t)
\quad \text{subject to no shared vertices.}
\]

An exact general-graph maximum-weight matching solver, such as NetworkX's `max_weight_matching`, is a candidate for implementation, subject to dependency and runtime testing. If an approximation is used, it will be labelled as an approximation and compared with the exact solver on sampled states where feasible.

Global matching is motivated by the fact that a locally strongest pair may consume two people and prevent a better combination of other pairs. Optimality applies only to the supplied weights: it does not guarantee that those weights predict MSMI accurately or solve the sequential decision problem by itself.

The policy may return an empty batch when no feasible allocation exists or when waiting is selected. Hard constraints will never be relaxed to increase coverage. Coverage-oriented utility adjustments, if explored, will be a separate ablation because coverage is a tie-breaker rather than the primary objective.

## 5. Clarification Policy: Decision-Aware Value of Information

### 5.1 Core principle

A question is useful when its answer is expected to improve the decision—not merely because it reduces uncertainty. For example, clarifying a hard field may unlock several candidate edges, while clarifying an irrelevant field may leave the best matching unchanged.

For a candidate query \(q\), define its estimated value as:

\[
\Delta(q) =
\mathbb{E}_{a}\left[V\left(M^*(S_t\mid a)\right)\right]
-
V\left(M^*(S_t)\right),
\]

where \(S_t\) is the current observable state, \(a\) is a possible answer, \(M^*(S_t)\) is the best matching under the current state, \(M^*(S_t\mid a)\) is the best matching after observing the answer, and \(V(M)\) is the total utility of the selected edges.

This estimates the expected marginal improvement to the global matching from obtaining the answer. It is a proposed decision rule, not a claim of optimality or a guarantee that the value function is submodular.

### 5.2 Practical query selection

The proposed clarification process is:

1. Exclude fields that are already observed or declined, and exclude unavailable members.
2. Identify candidate questions that could resolve `needs_clarification` edges or materially change weights on high-value candidate pairs.
3. Prioritise missing hard-constraint information that blocks potentially useful candidates.
4. Estimate the plausible decision impact of each candidate answer using only permitted training data and currently observable history.
5. Where feasible, recompute the best matching under candidate answers and rank queries by their expected marginal matching improvement.
6. Select questions within the exact daily budget, updating marginal values after selected queries when practical.
7. Refresh the state after clarification resolves and then allocate using the newly observable information.

As a simple comparison, the policy may use a **blocking-power heuristic**: prioritise a member whose unresolved hard information blocks many potentially feasible pairs. This heuristic is cheaper than full value-of-information calculation and will be tested as a practical approximation, not assumed to be equivalent to exact value of information.

The daily costs must be respected exactly: a hard-constraint bundle costs 3 units and a named soft-field question costs 1 unit, with a total budget of 12 units. Cost is a budget constraint and a ranking tie-breaker; the policy will not subtract arbitrary “cost points” from MSMI utility without a justified conversion.

### 5.3 Runtime-aware approximation

Evaluating every possible query by recomputing every possible matching may be expensive. The initial scope will restrict queries to those that can affect eligibility or high-value candidate edges, cache the current matching and reusable edge calculations, and benchmark the clarification calculation under the inference time limit. Exact query-value calculations may be compared with a cheaper heuristic on sampled states. Any approximation will be labelled and measured.

## 6. Optional Probability and Outcome-Aware Utility

Probability estimation is optional in the challenge. The primary proposal does not depend on a complex predictive model.

The initial edge weight will be a transparent soft-preference utility. If there is sufficient permitted data to estimate downstream outcomes reliably, an exploratory variant may estimate the probability that a feasible pair achieves MSMI, conditional on information observable at the decision time:

\[
U_{ij}(t) =
P(A_{ij}\mid S_t)
\times
P(D_{30,ij}\mid A_{ij},S_t)
\times
P(R_{2,ij}\mid D_{30,ij},A_{ij},S_t),
\]

where \(A_{ij}\) is mutual acceptance, \(D_{30,ij}\) is a date within 30 days of assignment, and \(R_{2,ij}\) is both second-meeting “yes” responses within three days of the date. This is a conditional-probability chain rule, not an independence assumption.

Directional acceptance and mutual acceptance are distinct targets. The policy will not multiply two directional acceptance probabilities unless their dependence is explicitly modelled or the independence assumption is justified. Hard constraints will remain separate binary gates, never features that a probability model can override.

Because qualifying MSMI outcomes may be sparse, any model will begin simply, use smoothing where appropriate, and track intermediate stages separately for diagnosis. Probability estimates, if reported, will be evaluated for calibration on held-out data. If estimates are unstable, data-starved, or not useful to decisions, the transparent rule-based score will be retained.

## 7. Missing, Declined, Delayed, and Censored Data

### 7.1 Missingness semantics

- `null` means unknown, not zero, “no,” or rejection.
- `not_asked` means the policy has not obtained the answer and may be able to clarify it.
- `declined` means the answer remains unavailable and must not be requested again.
- A missing response is not evidence of dislike.
- A future observation must never influence an earlier decision.

### 7.2 Delayed outcomes and right-censoring

Only feedback that has become observable may be used at a decision time. Recent introductions may still be awaiting responses, dates, or second-meeting outcomes. Such cases will be treated as pending or censored until the relevant observation window closes, not automatically labelled failures.

The evaluation will follow the prescribed 40-day follow-up period. Stage-level records will distinguish introduction assignment, directional responses, mutual acceptance, date, and second-meeting responses. Late responses may remain part of the history without satisfying the official outcome window.

### 7.3 Selective feedback and temporal leakage

Outcomes are observed only for pairs selected by a policy, so feedback is selective and can be biased by earlier decisions. Static historical files will not be treated as an unbiased sample of all possible pairs. Since historical logging propensities are not supplied, the proposal will not claim valid inverse-propensity weighting from those files. Comparisons will rely primarily on controlled simulator rollouts with common seeds.

Training or updating will use only observations available at the time of the corresponding decision. The policy will not use hidden simulator truth, future profile values, future feedback, private evaluation worlds, or inferred answers derived from identifiers or seeds. Adaptation will be conservative and optional because the number of positive outcomes may be small and selected.

## 8. Waiting and Sequential Decisions

Waiting is a legitimate action, but neither always waiting nor always matching immediately is assumed to be best. A delayed decision may preserve future options, but it also reduces the remaining time for a date and the second-meeting outcome and may leave members unserved.

The study will compare:

1. **Immediate matching:** allocate the best current feasible batch.
2. **Clarification followed by same-day matching:** obtain high-value information and then allocate using the refreshed state.
3. **Time-aware waiting (exploratory):** wait only when estimated future-option value exceeds the cost of delay and the reduced time remaining for the MSMI outcome window.

No fixed day-based rule will be introduced without experimental evidence. Waiting-time distributions and the number of members never served will be reported alongside MSMI and coverage.

## 9. Experimental Design

### 9.1 Required baselines

All three supplied baselines will be evaluated:

- **Greedy:** supplied clarification procedure and greedy pair allocation.
- **No-clarification:** no clarification requests, using the corresponding baseline matching procedure.
- **Random-feasible:** randomised selection among feasible candidate edges with non-overlapping introductions.

All comparisons will use identical seeds and scenario variants. Baseline results are reference points, not evidence that the proposed MatchWise policy improves performance.

### 9.2 Proposed policy variants

To isolate components, the planned comparisons are:

- **Global matching only:** global allocation with a simple utility and the baseline clarification strategy.
- **Blocking-power clarification + global matching:** test the lower-cost clarification heuristic.
- **Decision-aware clarification + global matching:** evaluate query selection by expected change in best matching utility.
- **Outcome-aware utility (optional):** replace the soft-agreement proxy only if a reliable estimator can be developed.
- **No-feedback variant (optional):** compare any conservative feedback update against a fixed utility.
- **Immediate vs time-aware variant (optional):** test waiting separately from allocation and clarification.

The full proposed policy is decision-aware clarification with global matching. Optional extensions will be added only if there is enough time to implement and evaluate them reliably.

### 9.3 Scenarios and seeds

Evaluate all six public scenario families: development/standard, sparse, cold start, delayed, shift, and drift. Use common seeds for every method so that comparisons are paired. The current development comparison uses seeds 101, 102, and 103; additional common seeds should be run if time and compute permit. The official private evaluation uses its own seeds and is not represented by the public development runs.

Training, validation, and development-test pools will remain separate. People or pairs must not be split across training and test. Every independent episode starts with fresh policy memory.

### 9.4 Metrics and aggregation

The primary metric is the official equally weighted mean of the six scenario-family means of MSMI per 100 arrived members. The following secondary metrics will be reported:

- distinct-member coverage;
- mutual acceptances per 100 arrived members;
- clarification cost, including cost by question type where available;
- inference time, including startup and worst-case invocation;
- invalid actions and invalid episodes;
- first-introduction waiting time and number never served;
- stage-level counts for responses, mutual acceptance, dates and second-meeting outcomes; and
- pending or missing feedback and censoring.

Means, per-seed results, scenario-level outcomes, and uncertainty will be reported. Denominators will not be pooled across scenario families, and the best seed will not be cherry-picked. If probabilities are reported, calibration will be assessed separately. All episodes must be valid.

## 10. Ablations and Failure Cases

### 10.1 Hypothesis-driven ablations

At least one ablation will be completed; the preferred initial set is:

**Ablation A — Greedy vs global allocation.** Replace global maximum-weight matching with greedy selection while keeping the utility and clarification strategy unchanged. This tests whether global allocation improves batch utility and the primary outcome.

**Ablation B — No clarification vs selective clarification.** Disable clarification while retaining the same allocation procedure. This measures the contribution of information acquisition.

**Ablation C — Blocking-power vs decision-aware clarification.** Compare the inexpensive heuristic with the expected-marginal-improvement approach under the same daily budget. This tests whether the additional computation is worthwhile.

Optional ablations will compare the soft-agreement proxy with outcome-aware utility, fixed utility with feedback adaptation, and immediate allocation with time-aware waiting. One main component should change at a time wherever possible.

### 10.2 Expected failure cases

- **Sparse candidate graph:** hard constraints, geography, or schedules may leave few or no feasible edges. No matching algorithm can create a valid partner where no edge exists.
- **Cold start:** little observed history can make utility weights weak and outcome estimates noisy.
- **Clarification budget exhaustion:** some hard constraints may remain unresolved, preventing otherwise possible introductions.
- **Declined hard fields:** a pair may remain unmatchable because eligibility cannot be established; the policy will not infer or override the missing answer.
- **Delayed feedback and censoring:** outcomes may not mature before a decision or episode ends. Pending outcomes must not be counted as failures.
- **Selection bias:** the policy observes outcomes only for its chosen pairs, so naïve learning may reinforce earlier selection decisions.
- **Utility misspecification:** soft-field agreement or estimated MSMI utility may not rank outcomes correctly.
- **Over-waiting:** waiting may reduce coverage or leave insufficient time for the full MSMI outcome window.
- **Runtime or invalid actions:** expensive query-value calculations or an invalid batch can invalidate an episode. Runtime and action validity are hard requirements, not optional quality metrics.
- **Small outcome counts:** a zero MSMI count in a small episode does not establish equivalence between methods.

The policy will be kept simple if more complex components do not produce reliable improvements.

## 11. Preliminary Baseline Observations

The current public development runs using seeds 101, 102, and 103 across the six scenario variants produced the following primary scores for the supplied baselines:

| Baseline | Preliminary primary score |
|---|---:|
| Greedy | 0.389 |
| Random-feasible | 0.278 |
| No-clarification | 0.139 |

These values are preliminary observations from the supplied synthetic simulator and evaluator. They are not results from the proposed MatchWise policy, do not establish statistical significance, and do not represent the private competition leaderboard. The final report should retain the corresponding machine-readable results and exact evaluation settings.

The baseline comparison motivates further investigation of clarification and allocation, but it does not prove that either proposed component will improve MSMI. All claims about MatchWise performance will be made only after the corresponding experiments have been run.

## 12. Limitations, Provenance, and Reproducibility

The benchmark uses synthetic participants, preferences, and outcomes. Results establish performance only within the tested simulation settings and cannot demonstrate real-world relationship compatibility or predict real people's decisions.

The implementation will follow the supplied policy interface and use only observable state, permitted clarification results, mature feedback, and explicitly carried policy memory. The public data contract and eligibility logic define the source of truth for feasibility and missingness. Random seeds, dependencies, evaluation commands, and all experiment outputs will be documented. Any additional model, tool, or external asset will have its provenance and licence recorded.

The repository currently contains the supplied starter policy and baseline experiment outputs. Global allocation, decision-aware clarification, outcome-aware utility, feedback adaptation, and time-aware waiting must be described as proposed components until they are implemented and evaluated. The final build report will include measured results, scenario-level outcomes, ablations, runtime, and limitations.

## References

1. Romeo & Juliet (2026). *The One Introduction Problem: Sequential Reciprocal Matching Under Uncertainty*. Participant specification, version 1.0.0, 5 October 2026. [https://github.com/RomeoJulietLove/The-Sequential-Matching-Problem/blob/main/PROBLEM_STATEMENT.md](https://github.com/RomeoJulietLove/The-Sequential-Matching-Problem/blob/main/PROBLEM_STATEMENT.md).

2. Romeo & Juliet (2026). *Participant Contract 1.0.0*. Supplied challenge documentation, [`docs/DATA_CONTRACT.md`](https://github.com/RomeoJulietLove/The-Sequential-Matching-Problem/blob/main/docs/DATA_CONTRACT.md).

3. Romeo & Juliet (2026). *Policy Interface 1.0.0*. Supplied challenge documentation, [`docs/POLICY_INTERFACE.md`](https://github.com/RomeoJulietLove/The-Sequential-Matching-Problem/blob/main/docs/POLICY_INTERFACE.md).

4. Karp, R. M., Vazirani, U. V., & Vazirani, V. V. (1990). An optimal algorithm for on-line bipartite matching. *Proceedings of the 22nd Annual ACM Symposium on Theory of Computing*, 352–358. [https://doi.org/10.1145/100216.100262](https://doi.org/10.1145/100216.100262).

5. Edmonds, J. (1965). Paths, trees, and flowers. *Canadian Journal of Mathematics, 17*, 449–467. [https://doi.org/10.4153/CJM-1965-045-4](https://doi.org/10.4153/CJM-1965-045-4).

6. Gale, D., & Shapley, L. S. (1962). College admissions and the stability of marriage. *The American Mathematical Monthly, 69*(1), 9–15. [https://doi.org/10.2307/2312726](https://doi.org/10.2307/2312726).

7. Pizzato, L., Rej, T., Chung, T., Koprinska, I., & Kay, J. (2010). RECON: A reciprocal recommender for online dating. *Proceedings of the 4th ACM Conference on Recommender Systems*, 207–214. [https://doi.org/10.1145/1864708.1864747](https://doi.org/10.1145/1864708.1864747).

8. Howard, R. A. (1966). Information value theory. *IEEE Transactions on Systems Science and Cybernetics, 2*(1), 22–26. [https://doi.org/10.1109/TSSC.1966.300074](https://doi.org/10.1109/TSSC.1966.300074).

9. Saar-Tsechansky, M., Melville, P., & Provost, F. (2009). Active feature-value acquisition. *Management Science, 55*(4), 664–684. [https://doi.org/10.1287/mnsc.1080.0952](https://doi.org/10.1287/mnsc.1080.0952).

10. Wilder, B., Dilkina, B., & Tambe, M. (2019). Melding the data-decisions pipeline: Decision-focused learning for combinatorial optimization. *Proceedings of the AAAI Conference on Artificial Intelligence, 33*(1), 1658–1665. [https://doi.org/10.1609/aaai.v33i01.33011658](https://doi.org/10.1609/aaai.v33i01.33011658).