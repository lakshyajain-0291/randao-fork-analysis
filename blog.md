# Forking the RANDAO

So basically ethereum uses something called RANDAO to pick which validators gets to propose blocks. every validator throws in a random value and they all get XORed together, thats how you get the "random" output. the problem is this design has a flaw - if you're a validator and you just... don't publish your block, you've effectively changed what the random output will be. and if you control enough stake you can keep doing this to bias who gets picked next.

previous work (Alpturer & Weinberg 2024) already showed this "selfish mixing" thing where withholding tail-end blocks gives you a small but real edge. like a 20% staker can get roughly +0.7% more slots than they should. not huge but not nothing either.

## whats new here

the Nagy et al paper takes it further. instead of just withholding your own block, what if you actively fork out an honest validators block? so you secretly build on an earlier block, let the honest guy publish, then release your private fork to orphan their block entirely. this does two things - removes their RANDAO contribution AND you can grab their transaction fees. double win for the attacker basically.

the attack needs you to have at least ~20% stake and get lucky with the slot ordering (you need to be scheduled around an epoch boundary in a specific way). if those conditions line up the gains are significant - that same 28% staker could see their effective share jump to ~36%.

## so whats the fix

the cleanest solution people are pointing to is EIP-7998 which would turn the randao_reveal into an actual VRF (verifiable random function). right now validators sign just the epoch number which is predictable. the proposal makes them sign the previous epochs mix + the current slot number, so even the proposer themselves cant predict their own output ahead of time. that closes the bias window entirely.

theres also talk of verifiable delay functions (VDFs) but those have their own scaling issues with ethereums validator set size.

empirically though - nobody seems to actually be doing this attack yet. the authors looked at two years of mainnet data and found no statistical evidence of manipulation. so its a theoretical risk for now.