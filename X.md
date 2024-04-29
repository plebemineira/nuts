NUT-X: 
==========================

`optional`

---

This NUT describes a mechanism for reward distribution based on blind signatures, to be used by Bitcoin mining pools. The mining pool is also an ecash mint. Miners submit shares along with blinded messages. After the block is found by the pool, miner get paid (per share) in ecash, while shares are priced proportionally to:
- the total coinbase reward
- the share's difficulty target
- the total pooled hashrate

This proposal aims to provide a pool architecture that:
- caters for small miners, which can combine their hashrate (as an alternative to solo/lottery), get some sats-backed ecash in return for each block the pool finds, and easily withdraw their rewards via LN (after they accumulate enough ecash to cover fees)
- minimize the trust between pool and miner (pool blindly exchanges shares for ecash)

## Hashrate estimation

Share hashes can be interpreted as unsigned 256 bit integers (`U256`).

A target is established as a hash, which when interpreted as a `U256` acts as an upper bound integer. Valid shares must have a hash whose integer interpretation is smaller than the target.

The maximum target is established as:

$T_{max}$ = `0x00000000FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF`

which can also be interpreted as:

$U256(T_{max}) = 26959946667150639794667015087019630673637144422540572481103610249215$

For some given $T_x$, we calculate difficulty threshold $D_x$ as:

$$D_x = \frac{U256(T_{max})}{U256(T_x)}$$

Let's define:
- The mining epoch $e$ as the time window between two blocks being found on the network
- $M_e = \{ M_1^e, M_2^e, ..., M_N^e \}$ as the set of miners working on the pool during epoch $e$
- $h^e = \{ h_1^e, h_2^e, ..., h_N^e \}$ as the set of miner hashrates during epoch $e$ (the unit here is `Hashes/s`).
- $\hat{h}_x^e = \{ \hat{h}_1^e, \hat{h}_2^e, ..., \hat{h}_N^e \}$ as the set of estimated miner hashrates during epoch $e$
- $d$ as the difficulty of each share
- $D_x$ as the difficulty threshold of the communication channel between pool and miner $M_x^e$ during epoch $e$
- $S_x^e$ as the set of valid shares submitted by miner $M_x^e$, under difficulty target $D_x$, during epoch $e$

In order to optimize bandwidth, the pool opens communication channels with lower difficulty threshold for small miners, while larger miners get channels with higher difficulty thresholds.

A share is considered valid if $d \geq D_x$. The probability of a valid share is given by [1]:
$$P(d \geq D_x) = \frac{2^{16}-1}{2^{48}D_x} \approx \frac{1}{2^{32}D_x}$$

If the pool collects shares from miner $M_x^e$ for a time window $t_e$ (in seconds), then the total number of valid observed shares $|S_x^e|$ is: 

$$|S_x^e| = \frac{h_x^et_e}{2^{32}D_x}$$

so we can estimate hashrate $\hat{h}_x^e$ as:

$$ \hat{h}^e_x = \frac{|S_x^e|2^{32}D_x}{t_e}$$

if $t_e$ is sufficiently large, then the expectation $E[\hat{h}_x^e] \to E[h_x^e]$, and we can assume $\hat{h}_x^e$ is a reliable estimation of the hashrate $h_x^e$.

## Reward distribution

The proposal here is similar to Pay-Per-Last-N-Shares (PPLNS).

The mining pool finds a block with $\theta_B^e$ sats in the coinbase (subsidy + fees).

The entire reward of $\theta_B^e$ on-chain sats is used to mint an equivalent $\theta_C^e$ e-cash sats on the pool's mint.

Everytime new ecash is minted, a LN invoice fee ($F_{M}^e$) is paid by the mint. So the ecash block reward has $\theta_C^e = \theta_B^e - F_M^e$ sats.

Based on the set of estimated hashrates $\hat{h}^e$, each miner $M_x^e$ is rewarded $R_x^e$ ecash sats under the following formula:

$$ R_x^e = \frac{\hat{h}_x^e}{\sum_{i=1}^{N} \hat{h}^e_i} * \theta_C^e = \frac{\frac{|S_x^e|2^{32}D_x}{t_e}}{\sum_{i=1}^{N}\frac{|S_i^e|2^{32}D_i}{t_e}} * \theta_C^e =  \frac{|S_x^e|D_x}{\sum_{i=1}^{N} |S_i^e|D_i} * \theta_C^e $$

For example, let's imagine 3 miners contributed to finding this block.
- Alice submitted the set $S_A^e$ of shares with difficulty above $D_A$
- Bob submitted the set $S_B^e$ of shares with difficulty above $D_B$
- Carol submitted the set $S_C^e$ of shares with difficulty above $D_C$

The reward distribution is:

- $R_A^e = \frac{\hat{h}_A^e}{\hat{h}^e_A + \hat{h}^e_B + \hat{h}^e_C} * \theta_C^e = \frac{|S_A^e|D_A}{|S_A^e|D_A + |S_B^e| D_B + |S_C^e| D_C} * \theta_C^e$ [ `ecash sats` ]

- $R_B^e = \frac{\hat{h}_B^e}{\hat{h}^e_A + \hat{h}^e_B + \hat{h}^e_C} * \theta_C^e =  \frac{|S_B^e|D_B}{|S_A^e|D_A + |S_B^e| D_B + |S_C^e| D_C} * \theta_C^e$ [ `ecash sats` ]

- $R_C^e = \frac{\hat{h}_C^e}{\hat{h}^e_A + \hat{h}^e_B + \hat{h}^e_C} * \theta_C^e = \frac{|S_C^e|D_C}{|S_A^e|D_A + |S_B^e| D_B + |S_C^e| D_C} * \theta_C^e$ [ `ecash sats` ]

- $R_A^e + R_B^e + R_C^e = \theta_C^e$


## Pricing shares

We establish $T = \{ T_1, T_2, ..., T_{} \}$ as the set of targets where:
- $T_1$ = `"0x0000000000000000000000000000000000000000000000000000000000000001"`
- $T_2$ = `"0x0000000000000000000000000000000000000000000000000000000000000003"`
- $T_3$ = `"0x0000000000000000000000000000000000000000000000000000000000000007"`
- $T_4$ = `"0x000000000000000000000000000000000000000000000000000000000000000F"`
- $T_5$ = `"0x000000000000000000000000000000000000000000000000000000000000001F"`
- $...$
- $T_{max}$ = `"0x00000000FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF"`

The genesis hash (maximum possible target in the protocol) most significant bit is the 224th, so targets are composed so that a `1` is progressively incremented, starting from the least significant ($T_1$) bit up until the 224th bit ($T_{max}$). So $|T| = 224$.

The connection between miner $M_x^e$ and pool goes through an initial setup where Miner $M_x^e$ performs a Blind Diffie-Hellmann Key Exchange (BDHKE) with the pool's mint. 

All shares will be submitted along with blinded messages. This will allow miner to redeem ecash in the future, in case the pool finds a block.

The difficulty target for each miner is adjusted dinamically, until it converges to a stable value.

As already established, if the pool finds a block during epoch $e$, it mints a total $\theta_C^e$ ecash sats from the coinbase, and miner $M_x^e$ has the right to claim an ecash sats reward according to:

$$ R_x^e = \frac{|S_x^e|D_x}{\sum_{i=1}^{N} |S_i^e|D_i} * \theta_C^e $$

where:
- $S_x^e$ is the set of shares (valid under $T_x$) submitted by miner $M_x^e$ on the last mining epoch $e$.
- $D_x$ is the difficulty threshold for the connection with miner $M_x^e$, calculated as $D_x = \frac{U256(T_{max})}{U256(T_x)}$

Therefore, each share is priced as:

$$ \beta^e (s_{x,j}^e) = \frac{R_x^e}{|S_x^e|} = \frac{D_x}{\sum_{i=1}^{N} |S_i^e|D_i} * \theta_C^e$$



The ecash supply coming from the coinbase is $\theta_C^e$, and we keep this supply fixed as:

$$ \theta_C^e = \sum_{j}^{} \sum_{i}^{} \beta(s^e_{i,j})$$

Where $s^e_{i,j}$ is the set of all share receipts signed by the pool/mint during epoch $e$.

In case a block is found on the network (by someone else, not the pool), then the pool starts a new mining epoch $e+1$ by:
- never issuing any ecash for past shares
- discarding shares from db

Also, the miner will discard all secrets and blinding factors they used during epoch $e$, since now they know those are not going to be redeemable for anything in the future.

## Privacy Limitations

As a requirement for bandwidth optimization (and part of the Stratum protocol), the pool needs to establish individual connections with miners under fixed difficulty targets.

If there's only one single miner under target $T_x$, then every share $s_{x,j}^e$ can be traced back to this specific miner.

In case more than one miner has a connection under the same $T_x$ difficulty target, then the annonimty set grows, and this issue is partially mitigated.

## Economic limitations

This reward distribution strategy puts the variance risk on the side of the miner.

## SV2 protocol extension

The `StratumV2` protocol allows for [Protocol Extensions](https://stratumprotocol.org/specification/09-Extensions/)

These new messages are proposed as an extension to the [Mining Protocol](https://stratumprotocol.org/specification/05-Mining-Protocol/):
- `OpenExtendedEcashMiningChannel` (Client -> Server)
- `OpenExtendedEcashMiningChannel.Success` (Server -> Client)
- `SubmitSharesEcashExtended` (Client -> Server)
- `SubmitSharesEcashExtented.Success` (Server -> Client)
- `SubmitSharesEcashExtended.Error` (Server -> Client)
- `RedeemSharesEcash` (Client -> Server)
- `RedeemSharesEcash.Success` (Server -> Client)
- `RedeemSharesEcash.Error` (Server -> Client)

The Blind Diffie Hellman Key Exchange (BDHKE) happens via 
`OpenExtendedEcashMiningChannel*` messages.

todo: check if only these extra messages will be sufficient for BDHKE

Shares are submitted along with blinded messages via `SubmitSharesEcashExtended*` messages.

Once the block is found, miner redeems their ecash (the blinded signatures) (1 for each share) via `RedeemSharesEcash*`.

This could be some functionality added to the proxy role (SV2+Translator).

## APIs

todo

## References

1: https://arxiv.org/abs/1112.4980