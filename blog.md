# Forking the RANDAO - draft v2

So basically ethereum uses something called RANDAO to pick which validators gets to propose blocks. every validator throws in a random value and they all get XORed together, thats how you get the "random" output. the problem is this design has a flaw - if you're a validator and you just... don't publish your block, you've effectively changed what the random output will be. and if you control enough stake you can keep doing this to bias who gets picked next.

previous work (Alpturer & Weinberg 2024) already showed this "selfish mixing" thing where withholding tail-end blocks gives you a small but real edge. like a 20% staker can get roughly +0.7% more slots than they should. not huge but not nothing either.

## whats new here

the Nagy et al paper takes it further. instead of just withholding your own block, what if you actively fork out an honest validators block? so you secretly build on an earlier block, let the honest guy publish, then release your private fork to orphan their block entirely. this does two things - removes their RANDAO contribution AND you can grab their transaction fees. double win for the attacker basically.

the attack needs you to have at least ~20% stake and get lucky with the slot ordering (you need to be scheduled around an epoch boundary in a specific way). if those conditions line up the gains are significant - that same 28% staker could see their effective share jump to ~36%.

## Proposed Solution

Nagy et al. formalise the attack and its analysis using a **Markov Decision Process (MDP)** framework. The core model works as follows: each MDP state captures the current "attack string" — essentially the sequence of adversarial and honest slots seen so far in an epoch. The adversary's available actions at each state are to either publish their block honestly or withhold it, and in the forking case, to secretly build a competing chain. Transition probabilities are determined by the adversary's stake fraction α and the honest network's behaviour. The reward function counts how many slots the adversary ultimately controls.

Because the forking attack spans an epoch boundary (the adversary acts across epoch *e* and epoch *e+1*), the authors extend the standard MDP formulation to what they call an **extended attack string**. This handles the fact that the adversary needs to evaluate RANDAO outcomes across two consecutive epochs simultaneously, which blows up the state space considerably. They manage this through a set of reductions and heuristics to keep policy iteration tractable.

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

**the MDP state space problem.** they admit themselves that proving global optimality is infeasible because the state space is too large. so they use reductions and approximations. that's fine practically but it means we dont actually know if the strategies they found are the *worst case* - there could be something more damaging they didnt find.

**rational actor assumption feels shaky.** the whole model assumes validators are purely profit-maximising. but lido and other large stakers have massive reputational skin in the game. getting caught manipulating RANDAO would be catastrophic for them commercially. so the incentive calculus might be very different in practice than the model suggests.

**no cross-protocol comparison.** the paper benchmarks against bitcoin selfish mining which is useful but a bit dated as a comparison point. would be interesting to know how this compares to other PoS chains with different randomness designs.

**the empirical section is kind of just "we looked and nothing happened."** which is a valid finding but doesnt tell us much about whether thats because the attack is hard to execute, or because validators are just being honest, or because the gains arent worth the risk. hard to know what to conclude.

EIP-7998 is the obvious fix but its a hard fork which means coordination overhead and timeline uncertainty. interim mitigations arent really discussed in depth.