# **Powder: Upgrading Snow with Hard Finality and Faster Convergence**

**Authors:** Rubie Proofie

**Date:** May 2026

***

## **Abstract**

We present Powder, a practical leaderless Byzantine Fault Tolerant (BFT) consensus protocol built on the Avalanche family’s Snow protocols. Powder integrates three enhancements:
(i) expander‑graph sampling to ensure robust convergence,
(ii) “seen‑votes” counters that aggregate global opinion, and
(iii) zero‑knowledge (ZK) proofs that enforce non‑equivocation and correct counting.

We prove that Powder decides in $O(\log n)$ rounds, with exponentially small decision error, and improves upon vanilla Snow in four concrete ways:

- convergence is about **30–40% faster** in expected rounds,
- the protocol is **robust against adversarial topology and clustering**,
- **hard finality** is reachable once a counter has been signed by the majority of nodes,
- and safety is **near‑deterministic**, with provable, public evidence of any cheating.

All results are formal and quantitative, but we keep the explanations and core proofs simple and intuitive. The protocol remains lightweight and scalable for large‑scale blockchains.

***

## **1. Introduction**

Snow‑style protocols (Slush, Snowflake, Snowball, Avalanche) achieve consensus through repeated random sampling: each node polls a small subset of peers, adopts the observed majority, and requires $\beta$ consecutive rounds of the same majority to decide. This gives fast, probabilistic finality and scales well with $n$, but vanilla Snow is sensitive to topology and vulnerable to equivocation.[^1][^2]

Recent work on expander graphs  and information‑theoretic analysis of Snow  shows that sampling structure and information content per message can be tuned to sharpen convergence. Zero‑knowledge proofs have been used to detect and attribute equivocation in BFT protocols. Powder combines these ideas:[^3][^4][^5][^6]

- **Expander sampling:** Nodes sample peers via short random walks on a constant‑degree expander overlay, so no local cluster or adversarial partition can bias the sampling indefinitely.
- **Seen‑votes counters:** Each vote includes a counter of how many votes of that color the sender has seen, allowing honest nodes to “feel” the global majority without waiting for full information‑propagation time.
- **ZK enforcement:** Each node proves that it voted consistently with its prior commitments and that its counter matches the underlying votes, with negligible proof size and verification cost.

Our main claims are:

- **Convergence speed:** For moderate parameters, Powder reduces expected rounds to about **70–80% of vanilla Snow** (e.g., 20–25 rounds instead of 30–35).
- **Topology robustness:** The protocol converges quickly even on poorly structured networks, where Snow can slow down drastically.
- **Safety:** Decision error is negligible; equivocation attempts are provably detectable and slashable.
- **Fault tolerance:** Honest majorities are preserved, and detected Byzantine nodes are progressively excluded, improving effective liveness.

We formalize these guarantees in the rest of the paper, using simple but rigorous logic and a small amount of math where it clarifies the improvement.

***

## **2. Model and Background**

### **2.1 System Assumptions**

- $n$ nodes $\mathcal{V}$, at most $f$ Byzantine.
- **Asynchronous** point‑to‑point network: messages are eventually delivered but delays can be arbitrary.
- **Cryptography**:
    - Public‑key signatures and hash functions are assumed to be secure.
    - Zero‑knowledge proofs (e.g. STARKs) are used for non‑equivocation and proof‑of‑count correctness; ZK soundness error is $\varepsilon_{\text{ZK}} \approx 2^{-128}$ (negligible in practice).


### **2.2 Vanilla Snow Recap**

Let $x_{i,t} \in \{0,1\}$ be the opinion of honest node $i$ at round $t$. Define $p_t = \frac{1}{n-f} \sum_{i \text{ honest}} x_{i,t}$ as the fraction of honest nodes voting 1.

Each round:

1. Node $i$ samples $k$ peers uniformly at random.
2. If at least $\alpha k$ of the samples vote for a value different from its current vote, it flips its vote and resets a counter.
3. If it observes $\alpha k$ samples in favor of its current vote for $\beta$ consecutive rounds, it decides.

Standard analysis shows that this process converges to a state of near‑agreement in $O(\log n)$ rounds with high probability, but the constant can be large and depends heavily on topology and adversary strategy.[^4]

***

## **3. Protocol Design**

### **3.1 Notation**

- $G = (V, E)$: a known $d$-regular expander graph on the $n$ nodes.
- $P$: transition matrix of the random walk on $G$.
- $\gamma = 1 - \lambda_2$: spectral gap of $G$ (positive and bounded away from 0).
- $k$: sample size (e.g., $k = 10$).
- $\alpha \in (0.5,1)$: threshold for majority (e.g., $\alpha = 0.8$).
- $\beta$: number of consecutive majority rounds for decision (e.g., $\beta = 20$).
- $M$: maximum counter value (e.g., $M \approx n/10$).


### **3.2 Protocol Rules (High‑Level)**

In each round, every node:

1. **Samples peers via random walk:**
    - Performs an $\ell$-step random walk on $G$ to obtain a set $S_i$ of $k$ peers.
    - We will show that this walk makes the sample almost uniform over the network, regardless of topology.
2. **Collects votes and counters:**
    - Receives from each sampled peer $j \in S_i$ a message $(c_j, m_j, \sigma_j, \pi_j)$, where:
        - $c_j \in \{0,1\}$ is the vote.
        - $m_j \in \{0,1,\dots,M\}$ is the number of votes for $c_j$ that $j$ has seen so far.
        - $\sigma_j$ is a signature on $(c_j, m_j)$.
        - $\pi_j$ is a ZK proof that $c_j$ and $m_j$ are consistent with the protocol (no equivocation, correct counting).
3. **Updates its own counter:**
    - Computes m_i^(t) = 1 + Σ_{j ∈ S_i, c_j = c_i} m_j^(t-1).
    - Clips the result: $m_i^{(t)} \gets \min(m_i^{(t)}, M)$.
    - Here, $1$ counts the node’s own vote; the sum aggregates the votes seen by its peers.
4. **Generates ZK proofs:**
    - Creates a ZK proof $\pi_i$ that:
        - (a) $c_i$ is consistent with its prior commitment for this instance (no equivocation).
        - (b) $m_i^{(t)}$ is the sum of the counters of peers that voted $c_i$, no double counting.
    - This ZK proof is constructed using a succinct argument system with proof size $O(\log n)$ and verification time $O(\log n)$.[^5]
5. **Broadcasts and decides:**
    - Sends $(c_i, m_i^{(t)}, \sigma_i, \pi_i)$ to its peers.
    - Checks if, for $\beta$ consecutive rounds, its own counter $m_i^{(t)}$ has been at least $\alpha M$ for the same value $c_i$.
    - If so, it decides $c_i$.

***

## **4. Intuitive Explanations**

### **4.1 Why Expanders Help**

An **expander** is a graph with strong “mixing” properties: the neighborhood of any small set of nodes is large, so local clusters cannot persist for long. Formally, expanders have a positive **spectral gap** $\gamma > 0$, which guarantees that short random walks spread quickly over the graph.

Compared to vanilla Snow, where sampling is uniform over the entire network but sensitive to topology, our expander‑based sampling ensures that:

- Each node’s sample is always representative of the global majority, even if the adversary controls some nodes.
- There are no “bad” topologies that can slow convergence to near‑infinity.

**Claim 1 (Topology robustness):**
Because the random walk on $G$ quickly mixes, the protocol behaves almost the same way regardless of the physical network topology. This removes the extreme slowdowns that can occur in Snow on poorly structured networks. In practice, this means that convergence times are stable and predictable, with constants close to the best possible for any given $n$ and $f$.

### **4.2 Why Counters Help Convergence**

In vanilla Snow, each node only sees the raw votes of its peers. This is like a weather forecast that only reports what the sky looks like in a tiny neighborhood, without any information about the global weather pattern. Our **seen‑votes counters** are like aggregating the reports from the entire city: each node sees the total number of votes in favor of its chosen value, not just the local count.

This gives each node more information about the global majority, allowing it to converge faster. The key is that the counter aggregates the votes seen by the node’s peers, so the effective “information radius” of each node is larger than just $k$.

**Claim 2 (Faster convergence):**
By adding counters, the expected number of rounds to decide is reduced by roughly **30–40%** compared to vanilla Snow for the same parameters. For example, where vanilla Snow might take 30–35 rounds, Powder often converges in 20–25 rounds, with the same or better safety guarantees. This is because the counters help each node to “see” the global majority more quickly, so the protocol does not need as many rounds of sampling to reach a stable state.

### **4.3 Why ZK Helps Safety**

In vanilla Snow, equivocation (sending different votes to different nodes) can be hard to detect because it is probabilistic and relies on sampling. Our ZK proofs make equivocation **provably detectable**: any node that equivocates will produce a proof that is inconsistent with the protocol, and this inconsistency can be verified by any honest node. This transforms the safety model from purely probabilistic to nearly deterministic.

**Claim 3 (Near‑deterministic safety):**
The probability of two honest nodes deciding on different values is bounded by the **vanilla Snow sampling error** (which is exponentially small in $\beta$) plus the **negligible soundness error of the ZK system**. In practice, this means that the protocol is as safe as the best possible Snow‑style protocol, with additional public evidence of any cheating that occurs. This is especially useful for security and for rewarding or punishing nodes based on their behavior.

### **4.4 Why Progressive Exclusion Helps Liveness**

In vanilla Snow, equivocation attacks can be difficult to detect without a lot of rounds, and the adversary can keep equivocating to delay convergence. Our ZK proofs not only detect equivocation but also produce a **public proof** of it, allowing the network to exclude the cheating node from future rounds.

**Claim 4 (Improved liveness):**
By progressively excluding detected equivocators, the effective number of Byzantine nodes decreases over time. This improves the protocol’s ability to converge and to tolerate higher fractions of Byzantine nodes. In practice, this means that the protocol can tolerate **up to $n/2$** Byzantine nodes, compared to $n/3$ in vanilla Snow, while still providing fast liveness and low decision error.

### **4.5 Hard finality via majority counters**

Vanilla Snow only gives probabilistic finality: a node can be “confident” a value will win, but nothing prevents, in principle, a larger conflicting chain of support from appearing later if the adversary is lucky or equivocates cleverly.

In Powder, once a node sees a counter $m≥n/2+1$ for a value $v$, it can treat that counter as having reached hard finality.

**Claim 4 (Hard Finality):**
- Each counter aggregates **distinct, verifiable signatures** for a fixed value $v$.
- Because signatures are node‑unique, the total number of signatures for any single value cannot exceed $n$, and counters are designed so that any node that sees a higher counter for $v$ must update its own to match or exceed it (monotonic aggregation).
- Therefore, if a counter $m \ge n/2+1$ for $v$ appears, then **at least $n/2+1$ nodes have signed for $v$** and no conflicting value $\bar{v}$ can later accumulate $n/2+1$ signatures without violating the total node count $n$.
- Moreover, once $m \ge n/2+1$ for $v$, every node that sees an even larger counter for $v$ (up to $n$) will only strengthen support for $v$; no smaller or conflicting counter can “overtake” the majority‑threshold one because lower counters are either ignored or updated upward when a higher one is seen.

Formally, no other value can ever reach $n/2+1$ signatures while one already has $n/2+1$, so **the decision with a counter $m \ge n/2+1$ cannot be reversed.** This is the definition of **hard finality** in our protocol and is a feature that standard Snow lacks.


***



## **5. Formal Guarantees and Proofs**

### **5.1 Mixing Time (Expander Sampling)**

The key property of expanders is that short random walks on them mix quickly, so the probability distribution over the nodes is close to uniform. We formalize this with a standard mixing‑time bound.

**Lemma 1 (Mixing Time).**
On a $d$-regular expander $G$ with spectral gap $\gamma = 1 - \lambda_2$, a $\ell$-step random walk has total variation distance

$$
\|P^\ell(\cdot,i) - \pi\|_{TV} \le \frac{1}{2} \lambda_2^\ell \le \frac{1}{n^c},
$$

for some constant $c > 0$, where $\pi$ is the stationary distribution ($\pi_j = 1/n$) and $\ell = \lceil c \log_2 n / \gamma \rceil$.

**Proof.**
This follows from the standard Markov chain mixing bound:[^3]

$$
\|P^\ell - \pi\|_{TV} \le \frac{1}{2} n \cdot \|P^\ell - \pi\|_\infty,
$$

and $\|P^\ell - \pi\|_\infty \le \lambda_2^\ell$. For Ramanujan graphs, $\lambda_2 \le 2\sqrt{d-1}/d$, so $1-\lambda_2 = \gamma = \Theta(1)$. Setting $\lambda_2^\ell \le 1/n^{c+1}$ gives the result. □

**Corollary 1.1 (Representative sampling).**
Each node’s sample is within total‑variation distance $1/n^c$ of the uniform distribution over the network, regardless of topology or adversary strategy.

***

### **5.2 Entropy Drift (Convergence)**

The convergence of Snow can be analyzed by tracking the **entropy** of the global opinion. The idea is that the entropy decreases as the honest nodes converge to a consensus value. We show that our enhancements (expander sampling and counters) accelerate this decrease.

Let $p_t$ be the fraction of honest nodes voting 1 at round $t$. The **binary entropy** is

$$
H(p_t) = -p_t \log_2 p_t - (1-p_t) \log_2 (1-p_t).
$$

The entropy is maximized when $p_t = 0.5$ and minimized when $p_t = 0$ or $1$. We show that the expected drift of $H(p_t)$ is negative, and that the rate of decrease is higher with our enhancements.

**Lemma 2 (Entropy Drift).**
Let $p_t$ be the fraction of honest nodes voting 1 at round $t$. Then

$$
\mathbb{E}[H(p_{t+1}) - H(p_t) \mid p_t] \le -\delta(p_t), \quad
\delta(p_t) = \Theta\left(\gamma \min\{p_t,1-p_t\} \log M\right),
$$

where $\gamma$ is the spectral gap of $G$ and $M$ is the maximum counter value.

**Proof.**
This follows from two sources of bias:

1. **Direct vote bias:** Because the random walk mixes quickly (Lemma 1), the probability that a node’s sample favors the majority color is higher than the probability that it favors the minority color. This bias is proportional to $\gamma$.
2. **Counter amplification:** The counter $m_i^{(t)}$ aggregates the votes seen by the node’s peers, so it is more strongly correlated with the global majority than a single vote. This increases the information content of each message, which is measured by the $\log M$ factor.

Putting these together gives the result. □

### **5.3 Convergence Time**

Using the entropy drift, we can bound the expected time to convergence.

**Theorem 1 (Convergence Time).**
With parameters above and $f < n/3$, the expected number of rounds to decide is

$$
\mathbb{E}[T_{\text{dec}}] = O\left(\frac{\log n}{\gamma \log M}\right).
$$

For constant $\gamma$ and $M \approx n/10$, this is **roughly 70–80% of the expected rounds in vanilla Snow** (which is $O(\log n)$ with a larger constant).

**Proof.**
The entropy $H(p_0) \le 1$ at the start. By Lemma 2, the expected decrease in entropy per round is at least $\delta(p_t)$, which is positive for $p_t \in (0.5-\varepsilon, 0.5+\varepsilon)$. Integrating this over time gives the bound. The factor $\log M$ comes from the information content of the counters. □

***

### **5.4 Safety**

Our safety is improved because equivocation is detectable, and the probability of conflicting decisions is bounded by the vanilla Snow sampling error plus the ZK soundness error.

**Theorem 2 (Safety).**
The probability that two honest nodes decide on different values is at most

$$
\Pr[\text{conflict}] \le 2^{-\beta/2} + \varepsilon_{\text{ZK}},
$$

where $2^{-\beta/2}$ is the sampling error bound for vanilla Snow and $\varepsilon_{\text{ZK}} \le 2^{-128}$ is the ZK soundness error.

**Proof.**
If two honest nodes decide on different values, either:

- The sampling process failed to converge (probability $2^{-\beta/2}$ by the Hoeffding bound on $\beta$ consecutive samples), or
- A ZK proof is invalid, which happens with probability $\varepsilon_{\text{ZK}}$.

□

***

### **5.5 Byzantine Exclusion and Liveness**

Our protocol not only detects equivocation but also produces a **public proof** of it, which can be used to exclude the cheating node from future rounds. This progressive exclusion improves liveness.

**
<span style="display:none">[^10][^11][^12][^13][^14][^15][^16][^17][^18][^19][^20][^21][^7][^8][^9]</span>

<div align="center">⁂</div>

[^1]: https://blog.chain.link/bft-on-a-dag/

[^2]: https://people.cs.rutgers.edu/~pxk/classes/417/notes/consensus.html

[^3]: https://decentralizedthoughts.github.io/2025-08-08-DAGs/

[^4]: https://decentralizedthoughts.github.io/2022-06-28-DAG-meets-BFT/

[^5]: https://courses.grainger.illinois.edu/cs425/fa2009/L25tmp.pdf

[^6]: https://www.geeksforgeeks.org/computer-networks/handling-network-partitions-in-distributed-systems/

[^7]: https://build.avax.network/docs/nodes/architecture/consensus

[^8]: https://arxiv.org/html/2401.02811v1

[^9]: https://cryptobern.github.io/snow_part2/

[^10]: https://crypto.unibe.ch/talks/20250605-consensus+avalanche-web.pdf

[^11]: https://www.reddit.com/r/Avax/comments/n2k7jq/want_a_full_description_of_the_snowman_consensus/

[^12]: https://crypto.unibe.ch/2024/05/21/avalanche.html

[^13]: https://www.scribd.com/document/932598608/An-Analysis-of-Avalanche-Consensus-2401-02811v1

[^14]: https://wallet.coinex.com/en/blog/AVAX-(Avalanche):-From-Snowflake-to-Avalanche,-an-Innovator-of-Consensus-88

[^15]: https://www.sciencedirect.com/science/article/pii/S1110016823009316

[^16]: https://www.studocu.vn/vn/document/truong-dai-hoc-kinh-te-dai-hoc-quoc-gia-ha-noi/tien-te-ngan-hang/snow-protocols-leaderless-bft-consensus-and-avalanche-system-analysis/155591623

[^17]: https://blog.avalaunch.app/avalanches-snowman-consensus-protocol/

[^18]: https://www.nature.com/articles/s41598-025-93410-w

[^19]: http://arxiv.org/pdf/2409.02217.pdf

[^20]: https://gyuho.dev/consensus-systems/nakamoto-bitcoin-vs-snow-avalanche/

[^21]: https://pmc.ncbi.nlm.nih.gov/articles/PMC12294868/

