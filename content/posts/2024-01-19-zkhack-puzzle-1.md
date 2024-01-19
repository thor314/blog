---
title: "Solving ZK Hack IV puzzle 1"
date: 2024-01-18
draft: false
---

*What follows is a solution to [ZK Hack IV, Puzzle 1](https://zkhack.dev/zkhackIV/puzzleF1.html). ZK Hack is a series of zero-knowledge cryptography CTFs, workshops, hackathons, and study groups. Thanks to the ZK Hack organizers for creating this CTF, Geometry for this puzzle, and the ZK Hack community. Thanks to Paul Gafni for discussion.*

 You might be interested in this post if you're interested in:
 - a smattering of zk-related trivia about nullifiers and elliptic curves
 - The process of (cryptographic) puzzle solving

*Recommended reading order*:
- read this post top to bottom for a concise description and solution of the puzzle first, some mathematical nuggets to hold onto, and finally the messy stream of consciousness log of problem solving.
- read this post back to front for the order in which it was actually written (messy problem solving log, then cleaned up and bloggified knowledge nuggets).

## `puzzle.solve()`
### background
*all the knowledge nuggets you need to solve the puzzle*
- We have a [merkle tree](https://en.wikipedia.org/wiki/Merkle_tree) constructed with the [poseidon hash function](https://eprint.iacr.org/2019/458.pdf) over elliptic curve [MNT_753_4](https://eprint.iacr.org/2014/595.pdf), with public keys stored at the leaves.
- Miyaji–Nakabayashi–Takano (MNT) curves are pairs of elliptic curves suited for [elliptic curve pairings](https://www.zellic.io/blog/what-are-elliptic-curve-pairings/); that is, the (scalar, base) field for MNT_753_4 correspond to the (base, scalar) field for MNT_753_6.
- A [nullifier](https://zips.z.cash/protocol/protocol.pdf#nullifierconcept) in Zcash is a unique hash of the public keys stored at the leaves of the merkle tree. For curve generator point $G$ and secret scalar $s$, the public key $pk$ is computed simply, $pk=sG$. Thus the nullifier may be computed:  $N=H(pk)$.[^1] 
- The prover in the nullifier argument provides a ZK proof of knowledge of $s$ such that the nullifier hash is correct:[^2]
    - $N=_?H(pk)=H(sG)$
- In this puzzle, we use the [Groth16](https://eprint.iacr.org/2016/260.pdf) prover to construct a zk proof of the correctness of the hash.
- Zcash uses the [Jubjub](https://zips.z.cash/protocol/protocol.pdf#jubjub) elliptic curve for their nullifier argument. Jubjub does not support a pairing function. 
- As remarked on in [Zcash spec lemma 5.4.7](https://zips.z.cash/protocol/protocol.pdf#concreteextractorjubjub):
> **Lemma 5.4.7**. Let $P=(u,v) \in \mathbb J^{(r)}.$ Then $(u,-v) \not \in \mathbb J^{(r)}$ (subgroup of Jubjub of order r).  

The lemma lends itself to a storage optimization for nullifiers over Jubjub: if $P=pk=(u,v)$, (note the switch to $u,v$ denotes that we are using [Twisted Edwards](https://en.wikipedia.org/wiki/Twisted_Edwards_curve) form, rather than [Weierstrass form](https://en.wikipedia.org/wiki/Elliptic_curve), expanded on in more detail below). We may choose to only store the $u$ coordinate of $pk$, to avoid redundancy. 

That is, instead of taking the hash $H(pk)$ as suggested above, the Zcash nullifier takes the hash of only one coordinate: $H(pk.x)$. This optimization is particular to Jubjub, not universal to all curves; and in particular, is not true of MNT_753_4.

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

Suppose $pk = sG = (x,y)$, and nullifier $N=H(sG.x)$. Recall that nullifiers must be unique to avoid double-spends. We give a constructive algorithm to obtain $s'$ such that $N=H(s'G.x)$.

Let $s'G=(x,-y)$. Let the order of generator $G$ be denoted $n$. Then:
$$\begin{align}
s'G &= \mathcal O - sG \\
&= nG - sG \\
&= (n-s)G \\
\therefore s'&=n-s
 \end{align}$$

If there exists $s'$ such that $s'G=(x,-y)$ is a point on the curve, then nullifier $N$ may not be unique, that is:
$$H(sG.x) = H((x,y).x) = H(x) = H((x,-y).x)= H(s'G.x)  $$

Which we may implement as follows:
```rust
/// given a leaked secret, return the double spend secret
fn get_hack(leaked_secret: MNT4BigFr) -> MNT4BigFr {
  let n = MNT4BigFr::from(MNT6BigFr::MODULUS);
  n - leaked_secret
}
```
*[github link](https://github.com/thor314/puzzle-gamma-ray/blob/main/src/main.rs)*

## `puzzle.deets()`
*extended background on elliptic curves, Zcash lemma 5.4.7, and working with a library with an elaborate trait system*
### Twisted Edwards and Zcash lemma 5.4.7
*Tbh I still haven't entirely figured out Zcash Lemma 5.4.7, and am leaving my notes and confusion [here](https://hackmd.io/@cryptograthor/BycdetdFT). Please reach out if you have more familiarity with the subject matter.*

## `puzzle.log()`
*My lightly editted puzzle solving log. You might be interested in this as a map of my state of mind while thinking about the puzzle. Keeping a log helps navigate overwhelming context dumps, and to avoid doing wrong and/or stupid things repeatedly. I use Obsidian to take these notes, for which I have written [an extensive setup and usage guide](https://github.com/thor314/obsidian-setup).*

Log was too long to include here, so see [this hackmd](https://hackmd.io/@cryptograthor/ByU_8Sdta).

## footnotes
[1]  Unrelated to this puzzle, the Zcash nullifier argument also includes a nonce, so that a public key can be used more than once: $N=H(pk,r)$.

[2] Unrelated to this puzzle, the Zcash nullifier must prove two other conditions:
- that nullifier $N$ has not already been published--this is not included in the proof for the puzzle, but is checked elsewhere in the puzzle driver; but, would need to be checked in the proof in real life
- the existence of the note(s) (coins) that will be spent, via proof of knowledge of:
    - $v$, the value of the note
    - $a$ the address possessing the note
    - $r$ the random nonce for the note

[3] a little more on Twisted Edwards and Montgomory Curves: 
Every montgomery form elliptic curve has a birational Twisted Edwards equivalent. Edwards curves have fast addition algorithms, and Twisted Edwards curves are nearly as fast, enabling faster point operations for many curves currently in use. The twist lies in the $a$ parameter, without which, many Montgomory curves have no Edwards equivalent. 

$$
\newcommand{Edwards}{u^2+v^2=1+u^2v^2}
\newcommand{TwistedEdwards}{a\Edwards}
$$

$$\text{Edwards form:}\quad \Edwards$$
$$\text{Twisted Edwards form:}\quad \TwistedEdwards$$

where $a$ and $d$ are constants in the field over which the curve is defined, and $a \neq d$. These curves are a generalization of Edwards curves (where $a=1$) and are used in cryptography for efficient arithmetic operations. Typically, $c=1$, and is often excluded.

Montgomery elliptic curves are defined by the equation:
$$By^2 = x^3 + Ax^2 + x$$
where $A$ and $B$ are constants in the field over which the curve is defined, with $B \neq 0$ and $A^2 \neq 4$, over a field $k$ with $\text{char}(k) \ne 2$. These curves are used in cryptography for their efficient point multiplication properties.

[4] Don't check this by hand. The algebra won't work out nicely, or at least mine didn't. I'm not sure why, but I believe the answer has something to do with algebraic geometers mumbling curses, and at this point I'm not interested enough to dig deeper. But if you, dear reader, know what that's about, do reach out, I'd love to know.
