# Secure Multi-Party Computation: An Introduction

*This post is based on the material from Peter Scholl's [introductory lecture](https://simons.berkeley.edu/talks/peter-scholl-aarhus-university-2025-05-22-0) at the Simons Institute for the Theory of Computing, in Summer 2025.*

Imagine you and your friends want to find out who earns the highest salary without anyone revealing their actual income. Or consider how financial institutions might want to collaborate on fraud detection while keeping their customer data completely private. These scenarios are examples of problems addressed by **secure multi-party computation (MPC)**: enabling multiple parties to jointly compute a function over their private inputs without revealing anything beyond the final result.

## What is MPC?

At its heart, MPC seeks to emulate a trusted third party that could collect everyone's private inputs, compute the desired function, and return only the result. But instead of relying on such a trusted party (which may not be reasonable to rely on), MPC uses cryptographic protocols to achieve the same goal through interaction between the parties themselves.

While MPC began with theoretical research in the 1980s, in recent years the pace of research has drastically accelerated, enabling many real-world applications, from secure auctions and private analytics to machine learning on sensitive data and protecting cryptographic keys through threshold cryptography. A [growing list of deployments](https://mpc.cs.berkeley.edu/) maintained by researchers at UC Berkeley demonstrates that MPC has moved from academic theory to practical reality.

## The Security Landscape: When is MPC Possible?


The feasibility and security guarantees of MPC depend heavily on the setting.

**The Adversary Model**: Are we dealing with passive adversaries (who follow the protocol but try to learn extra information) or active ones (who may deviate from the protocol)? Are corruptions static or adaptive?

**Corruption Threshold**: How many out of the $n$ total parties might be compromised? This is referred to as thre *threshold*, $t$. A critical point is often whether $t < n/2$ - called the *honest majority* setting - or $t \ge n/2$, where the majority of the participants (perhaps even $n-1$) may be corrupted.

**Computational vs Information-Theoretic Security**: Some protocols rely on computational assumptions (like the security of symmetric encryption algorithms such as AES, or "public-key" style assumptions like the hardness of factoring or computing discrete logarithms), while others provide perfect security even against computationally unbounded adversaries.

A classic [impossibility result](https://dl.acm.org/doi/10.1145/12130.12168) by Cleve shows that even simple two-party computations like the AND function cannot be computed with perfect security when half or more parties might be corrupted. This pushes us toward either computational assumptions or the honest majority setting.

## The Building Blocks: Secret Sharing

Most MPC protocols are based on **secret sharing** - a way to split sensitive data into pieces such that:
- Any authorized subset can reconstruct the secret
- Any unauthorized subset learns nothing about it

A common way to define what it means for a subset to be authorized is the *threshold* setting: no set of $t$ or fewer parties can learn any information about the secret, while any $t+1$ or more parties can reconstruct.

### Shamir Secret Sharing

**Shamir Secret Sharing** is an elegant solution that supports an arbitrary threshold $t < n$, and is often used in the honest-majority setting. To share a secret $s$ that lies in a finite field $\mathbf F_p$, where $p>n$:
1. Create a random polynomial $f(x)$ over $\mathbf F_p$ of degree $t$ such that $f(0) = s$
2. Give each party $i$, for $i=1, \dots, n$, the value $f(i)$ (**important**: we clearly must not give any party $f(i)$ for $i=0$)
3. Any $t+1$ parties can reconstruct the secret via Lagrange interpolation, but $t$ or fewer learn nothing

A key feature of Shamir sharing lies in its **homomorphic properties**: 
- Adding secrets is as simple as locally adding shares
- Multiplication is also possible by locally multiplying shares, with the catch that now, parties have shares corresponding to a degree $2t$ polynomial. Clearly, the result can only be reconstructed if we have $n > 2t$ parties, which requires an honest majority.

When using Shamir sharing for general-purpose MPC, most protocols employ a *degree reduction* protocol after each multiplication of secret-shared values. This is an interactive protocol that allows the parties to convert a degree-$2t$ sharing of the product $xy$ back into a fresh, degree-$t$ sharing, which can then be used for further homomorphic computations.

State-of-the-art protocols often use the [Damgård-Nielsen](https://www.iacr.org/archive/crypto2007/46220565/46220565.pdf) protocol for degree reduction. One recent example is [ATLAS](https://eprint.iacr.org/2021/833), which optimizes the Damgård-Nielsen protocol and applies clever consistency checking mechanisms to ensure security against an active adversary, with low cost.
The implementation of ATLAS has impressive performance, processing around 2 million multiplications per second for three parties.

### Additive Sharing for Dishonest Majority

When honest majority isn't guaranteed, **additive secret sharing** becomes the tool of choice. Here, a secret $x$ is shared as $x = x_1 + x_2 + \dots + x_n$, with each party holding one random share, sampled subject to this constraint.

While addition remains trivial (add shares locally), multiplication requires interaction through the famous **multiplication triple** (or Beaver triple) technique:
1. In a preprocessing phase, compute additive shares of random values $a$, $b$, together with shares of their product $c = a \cdot b$
2. Open $x+a$ and $y+b$, by having each party send their shares $x_i + a_i$ and $y_i + b_i$ to all other parties
3. Now, the desired product $x\cdot y$ can by obtained via a simple algebraic formula, which is linear in the secret values $x, y, a, b, c$.

## The Preprocessing Model: Separating Offline and Online

The dishonest majority protocol sketched above relies on
the **preprocessing model**, a key aspect of many modern MPC protocols, which separates computation into two phases:

**Preprocessing Phase** (also called, **offline phase**): Generate correlated randomness (like Beaver triples) before the inputs are known. This often use expensive, "public-key" style cryptographic operations.

**Online Phase**: Once inputs arrive, use the preprocessed material to compute efficiently with only lightweight, information-theoretic (or "symmetric" style) operations.

This separation can allow for a very fast online phase - often, just a constant factor slower than evaluating the circuit on plaintext data.
Of course, in the absence of a trusted dealer, the preprocessing phase still needs to be carried out, and this is typically where the heavy lifting is done.

## Achieving Active Security: Authenticated Shares

Moving from passive to active security (where adversaries might deviate from the protocol) requires additional machinery. One useful technique is **authenticated secret sharing**, which augments each secret share with an information-theoretic message authentication code (MAC).

For each wire in the circuit evaluation, the protocol maintains the following shared values over the field $\mathbb F_p$:
- Secret shares: $x_1 + \dots + x_n = x$
- MAC shares: $\gamma_1 + \dots + \gamma_n = \alpha \cdot x$
- Key shares: $\alpha_1 + \dots + \alpha_n = \alpha$

If $\alpha$ is randomly sampled from $\mathbb F_p$, and $p$ is at least $2^\lambda$ (for security parameter $\lambda$), this MAC setup can be used to ensure integrity of the authenticated shares.
This requires a *MAC checking protocol* to be used in conjunction with opening secret-shared values, such as the protocol from the [SPDZ follow-up paper](https://eprint.iacr.org/2012/642).

## Garbled Circuits: Achieving Constant Round Complexity

While secret sharing-based protocols can have a very low computational overhead, one drawback is that they require many rounds of interaction, proportional to the depth of the circuit being computed.
**Garbled circuits** are an important tool used to overcome this and obtain MPC for general functions with constant round complexity.

The rough idea is as follows:

1. **Garbling**: For each wire in the circuit, assign two random labels (one for 0, one for 1), which are viewed as secret, cryptographic keys. For each gate, create an "encrypted truth table" that maps input labels to output labels.

2. **Evaluation**: Starting with input labels corresponding to the actual inputs (which can be transmitted via oblivious transfer), "decrypt" your way through the circuit gate by gate until reaching the output.

3. **Output Decoding**: Given the secret mapping of wire labels to 0/1 values for the output wire, use the decrypted output label to recover the result of the circuit evaluation.

Recent advances have dramatically improved garbled circuit efficiency: through optimizations like point-and-permute, free-XOR, garbled row reduction and half gates, the size of a practical garbled circuit has reduced by an order of magnitude since Yao's original protocol. The current state-of-the-art is the ["three halves"](https://eprint.iacr.org/2021/749) approach by Rosulek and Roy, where XOR gates cost zero communication to garble, and each AND gates requires sending $1.5\lambda$ bits, where $\lambda$ (typically 128) is the security parameter.


## Advanced Techniques and Future Directions

The field of secure computation continues to evolve rapidly, with many active areas of research:

**PCGs and PCFs**: Pseudorandom Correlation Generators (PCGs) and Pseudorandom Correlation Functions (PCFs) offer an exciting approach to "silent" preprocessing, with minimal communication cost. These techniques allow parties to generate correlated randomness (like multiplication triples) using only short, correlated seeds and local computation, dramatically reducing the communication overhead of the preprocessing phase. Recent constructions based on variants of the learning parity with noise (LPN) assumption, in particular, have made this approach increasingly practical.

**Fully Homomorphic Encryption (FHE)**: While not strictly MPC, FHE enables a single party to perform arbitrary computations on encrypted data without ever decrypting it. FHE is typically seen as much more computationally expensive than MPC, however, its main selling point is that it can be used to obtain MPC with *highly succinct communication costs*, that are independent of the complexity of the function being computed.
Recent advances have made FHE increasingly practical for not-too-complex classes of functions.

**Homomorphic Secret Sharing (HSS)**: This technique can be seen as a hybrid of secret sharing and homomorphic encryption, allowing computation over shared secrets without interaction, for certain classes of functions. Parties can locally evaluate functions on their shares, obtaining a secret-sharing of the result.
HSS has opened up new techniques for achieving low-communication MPC without relying on the power of FHE, for instance, using only discrete log or code-based assumptions, instead of lattice-based cryptography.

**Function Secret Sharing (FSS)**: Function secret sharing can be seen as the *dual* of HSS: instead of secret-sharing the inputs and homomorphically evaluating a public function, FSS secret-shares the function itself and evaluates it on public inputs. This enables sublinear communication for certain function classes by distributing the computation description itself.
FSS is typically used for "lower end", simple classes of functions such as point functions and comparison functions, for which we have very efficient constructions based solely on fast, symmetric cryptography.
This has found applications in private information retrieval, secure aggregation, and as a building block in many MPC protocols.

**Advanced Garbling**:
Beyond gate-level optimizations, in recent years, more attention has been given to optimizing garbled circuits for more complex computational patterns. State-of-the-art techniques now support efficient conditional statements and branching logic, while garbled RAM extends the paradigm to handle computations over large datasets, without paying a naive "linear scan" cost for each memory access.Finally, recent advances in garbling using number-theoretic assumptions like DDH have led to high-rate garbling schemes, reducing the garbled circuit size from $O(|C| \lambda)$ to $O(|C|/\log \log |C|)$ bits for an arbitrary layered circuit $C$.

## Conclusion

Secure multi-party computation has matured from a theoretical curiosity to a practical tool for privacy-preserving computation. With continued advances in both theory and implementation, MPC is poised to enable new applications where privacy and collaboration must coexist.

While there are still challenges remaining in reducing the computational and bandwidth overheads of MPC, even today the field offers a rich landscape of techniques suited to a range of real-world scenarios such as privacy-preserving statistics, secure auctions, collaborative fraud prevention across institutions, private set intersection (e.g., contact discovery), threshold key management, and more.

If you want to experiment with MPC for your use-case, the [MP-SPDZ](https://github.com/data61/MP-SPDZ) library is a great choice for rapid prototyping of applications.
Check out our tutorials section for more info.

