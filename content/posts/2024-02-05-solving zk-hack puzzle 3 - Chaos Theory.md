---
title: "Solving ZK Hack Puzzle 3 - Chaos Theory"
date: 2024-02-05
last-update: 2024-02-05
draft: false
tags: ["cryptography", "puzzle", "zk", "pairing", "zkhack"]
categories: ["cryptography"]
---

![](/photos/2024-02-05-solving/puzzle-chaos-swirling.jpeg)

*What follows is a solution to [ZK Hack IV, Puzzle 3](https://zkhack.dev/zkhackIV/puzzleF3.html). ZK Hack is a series of zero-knowledge cryptography CTFs, workshops, hackathons, and study groups. Thanks to the ZK Hack organizers for creating this CTF, Geometry for this puzzle, and the ZK Hack community. You might also be interested in [solution for puzzle 2](https://thork.net/posts/2024-01-24-solving-zk-hack-iv-puzzle-2-supervillain/). If reposting elsewhere, find this post on my blog [here](http://thork.net/posts/2024-02-05-solving-zk-hack-puzzle-3-chaos-theory/)*.

You might be interested in this post if you're interested in:
 - an thought experiment on how building an authenticatino scheme based in elliptic curve pairings can go horribly awry!
 - Some background on ElGamal encryption and BLS signatures
 - The process of (cryptographic) puzzle solving

Of the three ZK Hack puzzles, this was by far the most straight-forward, so this will be a shorter post than the prior two.

## `puzzle.solve()`
### Background
*all the knowledge nuggets you need to solve the puzzle*

In this puzzle, we are presented with a novel scheme for the age-old cryptographic primitive, encryption plus authentication. Our Bob combines ElGamal encryption with BLS signatures. The details of each algorithm don't much matter, but the way Bob re-uses his encryption key for BLS signing very much does. Our goal is to decrypt the ciphertext, without access to the decryption algorithm.

We visit BLS and Elgamal schemes in more depth in the next section, but familiarity with these algorithms is unrequired for working this problem.

Elliptic curve points are denoted in capitals, scalars in lower case. The sender's secret and private keys are denoted $sk, PK=sk*G$, and the receiver's are denoted $sk',PK'$. We have 3 algorithms:
- Encryption (`send`): $c=(PK, sk*PK' + M)\gets E_{sk}(M, PK')$
- Authentication (`authenticate`): $\sigma= sk*H(c) \gets A_{sk}(c)$
- Auth Verification (`check_auth`): $b\in \{0,1\}=e(G,\sigma)=_? e(PK, H(c)) \gets V(PK, c, \sigma)$

Where the final authentication checks satisfies, since:
$$\begin{aligned}
e(G,\sigma)&=e(PK,H(c)) \\\
sk* e(G, H(c)) &= sk * e(G, H(c))
 \end{aligned}$$

We also are given a list of messages that Bob may choose to send.

We now demonstrate the exploit.

==🔴Stop! This is the 🚨puzzle solving police🚨! You have been given everything you need to find your own solution! Take ten minutes, pull out your pen, and reap the joy and mathematical gainz of problem solving🔴==

.

.

.

.

.

.

.

.

.

### the 'sploit
*The puzzle solution in short*

Let's revisit those 3 algorithms:
- Encryption (`send`): $c=(PK, sk*PK' + M)\gets E_{sk}(M, PK')$
- Authentication (`authenticate`): $\sigma= sk*H(c) \gets A_{sk}(c)$
- Auth Verification (`check_auth`): $b\in \{0,1\}=e(G,\sigma)=_? e(PK, H(c)) \gets V(PK, c, \sigma)$

Bob's only innovation is to re-use the encryption secret key for signing. Because we have the list of messages he may have sent, we can try to peel the message off of $c_2$. By testing each possible message $M_i$, we may compute $sk*PK'$:
$$
X_i = sk*PK' = c_2 - M_i = (sk*PK' + M) - M_i
$$

We have a line-up. One of these $A_i$'s is our guy. How will we differentiate them? We haven't done anything creative or unusual yet. But unlike secure encryption+authentication, now we have a pairing at our disposal.

We have that, for some $i$, $X=X_i = sk*PK' = sk * sk' G$. That looks a lot like a public key, corresponding to secret key $x=sk * sk'$. 

Can we shove $X$ into our verification step? 
$$\begin{aligned}
e(G,\sigma)&=_? e(X_i, H(c)) &\gets V(X_i, c, \sigma) \\\
sk* e(G, H(c)) &\not= sk*sk' * e(G, H(c))
\end{aligned}$$
Almost. We need to find a way to squeeze $sk'$ onto the left-hand-side. If we can multiply $G$ or $\sigma$ by $sk'$, we're done, and the former is just the public key, $PK'$. 

We can't re-use the provided auth verification algorithm, but we can easily write that new auth-verification ourselves:
```rust
pub fn check_auth_evil(s1s2_G: G1Affine, c: &ElGamal, s: G2Affine, receiver_pk: G1Affine) -> bool {
  // let lhs = { Bls12_381::pairing(G1Projective::generator(), s) };
  // (sk_r * G1, sk_s * H(.))
  let lhs = { Bls12_381::pairing(receiver_pk, s) };
  // let lhs = { receiver_pk, s) };

  let hash_c = c.hash_to_curve();
  // (sk_r*sk_s * G1, H(.))
  let rhs = { Bls12_381::pairing(s1s2_G, hash_c) };

  lhs == rhs
}
```
*[repo link](https://github.com/thor314/puzzle-chaos-theory/blob/840635f84c90eb6d39edb5f442fbf4540c202b90/src/main.rs#L148-L159)*

Plug that bad-boy in, and we're done:
```rust
  let sender_pk = blob.sender_pk;
  let c = blob.c.clone();
  let c1 = blob.c.1.clone();
  let s = blob.s;
  let rec_pk = blob.rec_pk;

  for (i, m) in _messages.into_iter().enumerate() {
    let a = (c1 - m.0).into_affine();
    let e = check_auth_evil(a, &c, s, rec_pk);
    if e {
      warn!("found message at index {}", i);
      warn!("win");
      break;
    }
  }
```

![](/photos/2024-02-05-solving/zkhack-puzzle-3-lineup.jpeg)
*line em up*

## `puzzle.deets()`
*Extended background on BLS and ElGamal encryption. This isn't technically required to solve the problem, but the interested reader may enjoy the background context.*

### BLS
The BLS signature scheme is a simple signature aggregation scheme, relying on a elliptic curve pairing. It consists of 3 algorithms, (sign, aggregate_signatures, verify), where verify checks either any individual's signature, or the whole aggregated signature. We don't use aggregation at all in this puzzle.
- **Individual signing** - Each individual signature $\sigma_i=sk_i\cdot H(M)$ is curve point on $\mathbb G_2$. Each public key $pk_i=sk_i\cdot G_1$ is a curve point on $\mathbb G_1$. 
- **Individual signature verification** and **aggregated verification** - Verification for any individual signature $\sigma_i$ is the same as for the entire signature, so we drop the indices: 
$$e(pk, H(M)) = e(G_1,\sigma)$$
Note that we don't typically verify the individual signatures, only the final aggregated signature. The point of a signature aggregation scheme is that *the aggregated signature is valid if and only if each component signature is valid*.

- **Signature aggregation** - Simply compute the sum of the signatures and sum of public keys:

$$\begin{aligned}
\sigma_{\text{agg}} &= \sum\limits_{i=1}^{n} \sigma_i=(\sum\limits sk_i)H(M) \\\
pk_{\text{agg}}&=\sum\limits pk_i=(\sum\limits sk_i)G_1
\end{aligned}$$
### ElGamal
ElGamal encryption is a secure but non-standard public-key cryptosystem based on the Diffie-Hellman key exchange. It was described by Taher Elgamal in 1985. ElGamal can be used for both encryption and digital signatures. The security of the ElGamal encryption system is based on the difficulty of solving the discrete logarithm problem. ElGamal encryption generally produces larger ciphertexts compared to the original plaintext size, effectively doubling the message size. This can be inefficient in terms of bandwidth and storage.

Denoting scalars in lower case, and elliptic curve points in upper case:

**Key Generation:**
- **Private Key:** Select a random number $x$ as the recipient private key.
- **Public Key:** Multiply the generator point $G$ of the curve by $x$ to get $Q = xG$. The recipient public key is $Q$.

**Encryption:** (message, pk) -> ciphertext
- Given a plaintext message $M$ represented as a point on the curve:
- Choose a random number $r$.
- Compute $C_1 = rG$.
- Compute $C_2 = M + rQ$.
- The ciphertext is the pair $C=(C_1, C_2)$.

**Decryption:** (ciphertext, sk) -> message
- Compute $M = C_2 - xC_1$.
- This works because: $C_2 - xC_1 = M + rQ - x(rG) = M + r(xG) - xrG = M$.


## `puzzle.log()`
*My puzzle solving log, lightly edited for readability. Keeping a log helps navigate overwhelming context dumps, and to avoid doing wrong and/or stupid things repeatedly. I use Obsidian to take these notes, for which I have written [an extensive setup and usage guide](https://github.com/thor314/obsidian-setup). My repo can be found [here](https://github.com/thor314/puzzle-supervillain/blob/main/src/main.rs). For taking math notes, I generally use a whiteboard or ipad for my scratch space before writing down my clean notes here, which helps to think n scratch faster. This puzzle in particular may have been faster to just work out on a whiteboard, without a log at all.*

![](/photos/2024-02-05-solving/zkhack-puzzle-3-pirate-log.jpeg)
*it be time for logging*

### Code runthrough
- hasher - same hash as last puzzle it looks like, sha256 based
- `struct ElGamal` representing ciphertext. Over a pair of G1Affine elements, let's tag it with Debug for convenience.
    - `hash_to_curve(self)` to create a G2 element 
    - we're implementing our own hash to curve? check pre-image resistance (not collision resistance based on goal context). Sha256 over 128 bits is still pretty strong, so it seems fine
- `struct Message` over `G1Affine` 
- `struct Sender` over a secret key in `Fr` and a pk 
    - verify that pk is correctly constructed
    - `send` a message to receiver 
    - `authenticate` generate the tag `s` in `G2Affine`: $s = H(c)*sk$ 
- `struct Receiver` gets a pk
- `Auditor` 
    - `check_auth` takes a the sender pk, the cipher-text, and the authentication tag $s$
        - encrypt-then-sign is fine, unless the encryption and authentication are malleable?
        - lhs: $e(G_1, s)$ 
        - rhs: $e(pk, H(c))$ , where $H$ is our sketchy hash function

I think we have everything we need already to work this out. To the whiteboard!

*See above notes for whiteboard observations.*