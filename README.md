<!-- solve-problem:ec392105-1555-45af-abe9-872e3ea01d61 -->
# Trust-minimized eclipse detection

A [Tangle Solve problem](https://solve.tangle.tools/problems/ec392105-1555-45af-abe9-872e3ea01d61).

## What is the problem?

Consider a cryptocurrency full node (e.g., Bitcoin) connected to a peer-to-peer network. An **eclipse attack** (Heilman et al. 2015; Tran et al. 2020, "Erebus") is one in which an adversary controls all of the victim's network neighborhood, isolating it from the honest network and feeding it a private view of the blockchain (stale blocks, fake transactions, wasted mining power).

The problem: formalize and solve what an **isolated node can detect** about its eclipse, under realistic constraints:

- **Observations available to the node:** authenticated messages only — block/header chains with valid proof-of-work, cryptographic signatures, peer handshake metadata (e.g., Bitcoin's ADDR relay, service bits). The node cannot see the global network.
- **Adversary model:** computationally bounded, controls all the victim's direct peer connections (Erebus shows this needs no BGP hijacking — just a /24-shadowing sybil swarm), and can delay/withhold/replay honest traffic it has itself observed.
- **Clocks:** the victim has bounded-drift, bounded-offset local clocks (no tight global synchronization).

Questions: (1) An **impossibility result**: prove there exists no deterministic (or restricted randomized) detector that distinguishes "eclipsed by adversary A" from "honest network behaving identically," given authenticated observations and bounded clocks alone — quantify exactly what observation distributions can and cannot be separated. (2) A **possibility result**: identify the weakest extra assumption (e.g., one guaranteed-honest intermittently-reachable peer; a weakly synchronized heartbeat; a beacon such as a signed NTP source or randomness beacon) under which eclipse detection becomes possible, and give a detector with provable false-positive/false-negative guarantees.

Ambiguity note: "authenticated observations" and "bounded clocks" are not formally defined in the census; the solver must fix a precise model (see Warnings).

## Why does it matter?

Wallets and light clients currently have no defensible basis for "I am seeing the true chain" — exchanges and merchants silently lose funds to partitioning (Erebus demonstrates stealthy partitioning from a single AS without routing attacks). A rigorous characterization of what isolated nodes can infer turns network-layer alarms from folklore into justified products: detectors with proven detection latency and false-alarm rates, and minimal connectivity requirements that node operators can actually provision. It also connects blockchain security to the classic failure-detector literature (Chandra–Toueg style), giving a clean theory answer to "how much synchrony/connectivity does partition detection require."

## What's already known?

- **Heilman et al. (2015):** demonstrated practical eclipse attacks on Bitcoin via address-book manipulation (addr flooding, feeler connections); proposed countermeasures (deterministic peer selection, anchor connections, stronger addr-eviction). These are heuristic; no detection guarantees.
- **Tran et al. (2020, Erebus):** showed eclipse of any Bitcoin node by a single AS without BGP hijacking, exploiting the /24 address-grouping heuristic; proposed countermeasures (Erebus detection heuristics based on AS-level connectivity of peer sets, extra outbound connections). Again heuristic scoring, no formal model.
- **Failure-detector theory** (Chandra–Toueg; the "eventual leader" / ♢S hierarchy) proves that in purely asynchronous systems with crashes, perfect failure detection is impossible; partition detection requires extra synchrony. This is the closest mature theory, but it models crash/link failures, not a *strategizing adversary presenting a consistent fake world*. Whether the adversary's need to fabricate valid PoW (costly but possible on a private chain) changes the picture is open.
- Related work exists on network-level partition detection in BFT protocols (e.g., views of liveliness) and Bitcoin stale-block statistics as health signals — but no published formal impossibility/possibility theorem for eclipse detection under the above model. The census explicitly types this as an "impossibility" research target, so no complete solution is known.

## What would count as solving it?

1. **Impossibility theorem:** A formal node/adversary model (observation interface, clock model, PoW cost model) and a proof that no detector in that model can distinguish eclipse from honest behavior with non-negligible advantage — ideally via an explicit indistinguishability/simulation argument (adversary simulates the honest world).
2. **Matching possibility result:** A concrete extra assumption (state it minimally: exactly what channel/peer/beacon it provides) and a detector with a theorem: bounded detection latency, bounded false-positive rate, and a lower bound showing the assumption is necessary (no strictly weaker assumption suffices).
3. Optionally: an implementation/evaluation on Bitcoin-core-like networking demonstrating the detector's practical cost.

Partial credit: an impossibility result alone under a well-justified model, or a single new assumption with a working detector, would each be publishable.

## How to start

1. **Read Heilman et al. §2–4 (attack mechanics) and §6 (countermeasures)** to enumerate the victim's actual observation surface; then **Tran et al. §IV–V** for the AS-level view and their detection heuristics — this tells you what "observations" a real node has.
2. **Formalize the model:** define the observation stream (headers, timestamps, addr messages, connection events), the adversary's simulation power (can it mine private PoW chains at rate α?), and the clock model (skew/offset bounds). The crux subtlety: an eclipsing adversary can *itself* observe the honest chain and replay it verbatim, making indistinguishability nearly trivial unless the victim gets something the adversary cannot reproduce or delay below some bound.
3. **Import failure-detector machinery:** adapt Chandra–Toueg-style reductions to show detection requires a "heartbeat"-like channel; the known obstacle is that in asynchrony-with-adversary, replay attacks make any latency-tolerant protocol spoofable.
4. Candidate weak assumptions to analyze: one honest anchor peer with unjammable intermittent connectivity; a signed time/beacon source; outbound-connection entropy (adversary must also win a peer-selection lottery).

## Key references

- Ethan Heilman, Alison Kendler, Aviv Zohar, Sharon Goldberg. *Eclipse Attacks on Bitcoin's Peer-to-Peer Network.* USENIX Security 2015. Demonstrates practical eclipse of Bitcoin nodes via address-manager manipulation and proposes heuristic countermeasures (anchor connections, deterministic eviction). https://www.usenix.org/conference/usenixsecurity15/technical-sessions/presentation/heilman
- Muoi Tran, Ian Tyson, Priyanka Mandal, Pisek Phand, Joshua Shiat, Min Suk Kang, Adrian Perrig. *A Stealthier Partitioning Attack against Bitcoin Peer-to-Peer Network* (Erebus). IEEE S&P 2020. DOI:10.1109/SP40000.2020.00027. Shows a single malicious AS can eclipse any Bitcoin node without BGP hijacking; proposes AS-diversity heuristics.
- Tushar Deepak Chandra, Sam Toueg. *Unreliable Failure Detectors for Reliable Distributed Systems.* JACM 43(2), 1996. Foundational impossibility/completeness framework for failure detection under asynchrony — the theoretical template for this problem. (Standard reference; no link issue.)

(No further references recorded in census; the Chandra–Toueg citation is added by the domain expert — verify applicability.)

## Warnings

- **Model choice dominates the answer.** If the adversary can fully observe and instantly replay honest traffic, detection is trivially impossible; any interesting result requires a cost or latency asymmetry (PoW cost, propagation-delay lower bound, unjammable channel). State this asymmetry explicitly or the impossibility is vacuous.
- **Bounded clocks ≠ synchronized clocks.** Bitcoin's nTime field is adversarially set (median-time-past rules give only weak constraints); don't treat block timestamps as trustworthy observations.
- Dead end: purely statistical detectors on stale-block rates or peer-set heuristics — these exist (Erebus heuristics) and have no adversarial guarantees; the adversary controls the statistics.
- Ambiguity: "weakest useful extra connectivity assumption" is undefined in the census; you must propose and justify a formal minimality criterion (e.g., via reduction between assumptions).
- Beware conflating eclipse *resistance* (harder to attack) with eclipse *detection* (noticing you were attacked) — this problem is the latter.

---
*`BCGT-000269` · flagged for review*

## Acceptance guidance

1. **Impossibility theorem:** A formal node/adversary model (observation interface, clock model, PoW cost model) and a proof that no detector in that model can distinguish eclipse from honest behavior with non-negligible advantage — ideally via an explicit indistinguishability/simulation argument (adversary simulates the honest world).
2. **Matching possibility result:** A concrete extra assumption (state it minimally: exactly what channel/peer/beacon it provides) and a detector with a theorem: bounded detection latency, bounded false-positive rate, and a lower bound showing the assumption is necessary (no strictly weaker assumption suffices).
3. Optionally: an implementation/evaluation on Bitcoin-core-like networking demonstrating the detector's practical cost.

Partial credit: an impossibility result alone under a well-justified model, or a single new assumption with a working detector, would each be publishable.

## Contribute

Contributions, audits, and critiques from humans, agents, or both are welcome. Open a pull request with your work, then [publish the exact revision or artifacts on Solve](https://solve.tangle.tools/problems/ec392105-1555-45af-abe9-872e3ea01d61?contribute=1). Include reviewed agent traces when available. Contributions need not solve the entire problem.

Keep experiment code, reports, and trace artifacts in this repository. Publish only material you have reviewed for public sharing. Solve retains the contribution snapshot and its review history independently of whether a pull request is merged.

Source record: BCGT-000269.

- [Public source catalog](https://drewstone.github.io/problems/#/p/BCGT-000269)
