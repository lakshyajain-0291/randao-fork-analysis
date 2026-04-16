# Forking the RANDAO

So basically ethereum uses something called RANDAO to pick which validators gets to propose blocks. every validator throws in a random value and they all get XORed together, thats how you get the "random" output. the problem is this design has a flaw - if you're a validator and you just... don't publish your block, you've effectively changed what the random output will be. and if you control enough stake you can keep doing this to bias who gets picked next.

previous work (Alpturer & Weinberg 2024) already showed this "selfish mixing" thing where withholding tail-end blocks gives you a small but real edge. like a 20% staker can get roughly +0.7% more slots than they should. not huge but not nothing either.

## What's New Here

The Nagy et al. paper introduces a meaningful escalation over prior work. Rather than simply withholding your own block, the attack involves actively forking out an honest validator's block. The adversary secretly builds on an earlier block, lets the honest proposer publish their block, then releases a private fork to orphan it entirely. This achieves two things simultaneously: it removes the honest validator's RANDAO contribution from the canonical chain, and the attacker captures the orphaned block's transaction fees and MEV.

The attack requires the adversary to hold at least ~20% stake and to occupy a specific slot arrangement around an epoch boundary. When those conditions are met, the gains are substantial — a 28% staker could see their effective share increase to roughly 36%.

## Related Work

Randomness manipulation in blockchain consensus has a reasonably long research history, and this paper sits at the end of a clear lineage.

The earliest and most well-known analogue is **selfish mining** in proof-of-work Bitcoin, formalised by [Eyal & Sirer (2014)](https://arxiv.org/abs/1311.0243). They showed that a miner controlling ≥25% of hash power can earn disproportionate block rewards by strategically withholding newly mined blocks and releasing them to orphan honest miners' work. The core mechanism — private chain building followed by strategic release — is structurally similar to the RANDAO forking attack, though the target (randomness bias vs. revenue) and protocol context (PoW vs. PoS) differ.

In the Ethereum PoS context, the first serious treatment of RANDAO manipulation came from a 2023 Ethereum Research forum post by [Wahrstätter ("Selfish Mixing and RANDAO Manipulation")](https://ethresear.ch/t/selfish-mixing-and-randao-manipulation/16081). Wahrstätter quantified how strategically missing tail slots — the last slots of an epoch — could shift the RANDAO output and reassign future block proposals. Using simulations and historical beacon chain data, he showed that major pools like Lido had encountered dozens of realistic opportunities to bias RANDAO. This was an important empirical contribution, though the post was informal and did not optimise the attack strategy.

[Alpturer & Weinberg (2024)](https://drops.dagstuhl.de/entities/document/10.4230/LIPIcs.AFT.2024.10) formalised selfish mixing as a Markov Decision Process, computing the theoretically optimal withholding strategy for any stake fraction α. Their key findings: a 20% staker gains roughly +0.68% additional proposals; a 10% staker gains about +0.19%. They also showed repeated manipulation misses ~0.3% of slots, slightly reducing chain throughput. Importantly, their model is restricted to block withholding only — forking out honest blocks was explicitly out of scope.

[Nagy et al. (2025)](https://eprint.iacr.org/2025/037) is the first work to study forking attacks on RANDAO directly. They extend the MDP framework of Alpturer & Weinberg to cover cross-epoch attack strings and introduce the forking strategy as a formal object of study. On the fix side, [EIP-7998 (2025)](https://eips.ethereum.org/EIPS/eip-7998) is a draft Ethereum protocol proposal that would convert `randao_reveal` into a per-slot BLS-based VRF, directly eliminating the bias vector this paper identifies.

The table below summarises the key papers:

| **Paper**                                                                                              | **Year** | **Method**                | **Key Result**                                                                                             | **Limitation**                                                                    |
| ------------------------------------------------------------------------------------------------------ | -------- | ------------------------- | ---------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| [Wahrstätter](https://ethresear.ch/t/selfish-mixing-and-randao-manipulation/16081) (Ethereum Research) | 2023     | Data analysis, simulation | Showed large pools could bias RANDAO by selfish mixing; e.g. Lido had opportunities to gain extra slots.   | Informal forum post; not a formal model; no attack optimisation.                  |
| [Alpturer & Weinberg](https://drops.dagstuhl.de/entities/document/10.4230/LIPIcs.AFT.2024.10)          | 2024     | MDP optimization          | Formulated selfish-mixing as MDP; found 20% stake → +0.7% block gain via optimal withholding.              | Only covers withholding; ignores forking attacks.                                 |
| [Nagy et al.](https://eprint.iacr.org/2025/037) (this paper)                                           | 2025     | MDP + new attack          | Introduced "forking" strategy; doubling bias when combined with selfish mixing; e.g. 28.1% stake → +8.37%. | Requires ≥20% stake and favorable slot pattern; complex cross-epoch state.        |
| [EIP-7998](https://eips.ethereum.org/EIPS/eip-7998) (proposed fix)                                     | 2025     | Protocol upgrade (VRF)    | Use per-slot BLS-VRF instead of simple RANDAO, eliminating the bias vector.                                | Draft proposal (hard fork needed); implementation and upgrade timeline uncertain. |
| [Eyal & Sirer (Bitcoin)](https://arxiv.org/abs/1311.0243)                                              | 2014     | Analysis + simulation     | Selfish mining in PoW: ≥25% hash power → disproportionate block rewards.                                   | Applies to PoW, not PoS.                                                          |

The clearest takeaway from this lineage is that Nagy et al.'s forking attack is not an isolated discovery — it is a direct extension of the selfish mixing line of work, and combining the two strategies yields significantly higher gains than either alone.

## Proposed Solution

Nagy et al. formalise the attack and its analysis using a **Markov Decision Process (MDP)** framework. Each MDP state captures the current "attack string" — the sequence of adversarial and honest slots seen so far in an epoch. The adversary's available actions at each state are to either publish their block honestly or withhold it, and in the forking case, to secretly build a competing chain. Transition probabilities are determined by the adversary's stake fraction α and the honest network's behaviour. The reward function counts how many slots the adversary ultimately controls.

Because the forking attack spans an epoch boundary (the adversary acts across epoch _e_ and epoch _e+1_), the authors extend the standard MDP formulation to what they call an **extended attack string**. This handles the fact that the adversary needs to evaluate RANDAO outcomes across two consecutive epochs simultaneously, which substantially expands the state space. They manage this through a set of reductions and heuristics to keep policy iteration tractable.

The key theoretical result is **Theorem 1** (condition for forking): given the slot ordering AH·A — adversary proposes, then honest proposes, then epoch boundary, then adversary proposes again — and stake α ≥ ~0.20, the adversary can profitably execute the fork. The honest block's RANDAO contribution is excluded from the canonical chain, and the adversary recovers the lost slot reward through the MEV and fees captured from the orphaned block.

Numerically, their policy iteration results show:

- A **28.1% staker** (roughly Lido's share at time of writing) can increase effective slot share from 28.1% to approximately **36.5%** under the combined selfish-mixing + forking strategy.
- A **40% staker** reaches ~47.5% effective share, compared to ~42.6% under selfish mixing alone — roughly doubling the excess gain.
- Even at **45% stake**, Ethereum remains less biasable than Bitcoin's PoW selfish mining (53.78% vs ~70% effective share in the PoW case).

For the fix, **[EIP-7998](https://eips.ethereum.org/EIPS/eip-7998)** proposes turning `randao_reveal` into a per-slot BLS-based VRF. Currently validators sign only the epoch number, which is predictable. The proposal makes them sign a container holding the previous epoch's RANDAO mix and the current slot number — meaning even the proposer cannot predict their own output in advance. This eliminates the bias window entirely and is a prerequisite for Single Secret Leader Election (SSLE). Verifiable delay functions (VDFs) are another direction, forcing randomness to be finalised only after all reveals are in, so late omissions have no effect.

Empirically, the authors analysed Ethereum mainnet data from September 2022 to October 2024 and found **no statistically significant evidence** of these attacks being used in practice. Lido, despite having 737 epochs with four consecutive tail slots, missed at least one tail slot in only 3 of those cases.

## Gaps (rough notes, will expand)

ok so the paper is solid but there are some things that feel undercooked or just worth flagging:

**the ≥20% stake requirement is doing a lot of work.** the attack is basically only relevant for the very largest pools. which yes, those exist (lido is right there), but it means this isnt really a systemic retail-validator problem. the paper could be clearer about who the realistic threat actors actually are.

**the MDP state space problem.** they admit themselves that proving global optimality is infeasible because the state space is too large. so they use reductions and approximations. that's fine practically but it means we dont actually know if the strategies they found are the _worst case_ - there could be something more damaging they didnt find.

**rational actor assumption feels shaky.** the whole model assumes validators are purely profit-maximising. but lido and other large stakers have massive reputational skin in the game. getting caught manipulating RANDAO would be catastrophic for them commercially. so the incentive calculus might be very different in practice than the model suggests.

**no cross-protocol comparison.** the paper benchmarks against bitcoin selfish mining which is useful but a bit dated as a comparison point. would be interesting to know how this compares to other PoS chains with different randomness designs.

**the empirical section is kind of just "we looked and nothing happened."** which is a valid finding but doesnt tell us much about whether thats because the attack is hard to execute, or because validators are just being honest, or because the gains arent worth the risk. hard to know what to conclude.

EIP-7998 is the obvious fix but its a hard fork which means coordination overhead and timeline uncertainty. interim mitigations arent really discussed in depth.
