---
title: "Solving ZK Hack IV puzzle 1"
date: 2024-01-18
last-update: 2024-01-22
draft: false
tags: ["cryptography", "puzzle", "zk"]
categories: ["cryptography"]
---
# Solving ZK Hack IV puzzle 1
![](/home/thor/projects/blog/static/photos/2024-01-18-Solving ZK Hack IV puzzle 1.md/twisted-elliptic-curve-moon-puzzle-1.jpeg)

*What follows is a solution to [ZK Hack IV, Puzzle 1](https://zkhack.dev/zkhackIV/puzzleF1.html). ZK Hack is a series of zero-knowledge cryptography CTFs, workshops, hackathons, and study groups. Thanks to the ZK Hack organizers for creating this CTF, Geometry for this puzzle, and the ZK Hack community. Thanks to Paul Gafni and Daira Hopwood for discussion.*

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

Suppose $pk = sG = (x,y)$, and nullifier $N=H(sG.x)$. Recall that nullifiers must be unique to avoid double-spends. We give a constructive algorithm to obtain $s'$ such that $N=H((s'G).x)$.

Let $s'G=(x,-y)$, *assuming such a point exists on our subgroup*. Let the order of generator $G$ be denoted $n$. Then:
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

One hiccup. We assumed (hoped) that $(x,-y)$ lay on the curve. On Jubjub, it does not. Why does it exist on `MNT_753_4`? It's time to put our twisted Edwards hats on.

## `puzzle.deets()`
*extended background on twisted Edwards elliptic curves, Zcash lemma 5.4.7*

### Twisted Edwards, and proof [Zcash Lemma 5.4.7](https://zips.z.cash/protocol/protocol.pdf#concreteextractorjubjub) with added notes
*This lemma was initially confusing; thanks to several folks who helped clarify, including the author of the lemma herself. Historical record of my confusion notes [here](https://hackmd.io/@cryptograthor/BycdetdFT).*

It may be helpful to first recall the twisted Edwards (tE) curve equation:
$$au^2 + v^2 = c^2(1 + du^2v^2)$$

The point addition law:
$$\begin{align}
u_3 &= \frac{u_1v_2 + v_1u_2}{1 + du_1u_2v_1v_2}\\
v_3 &= \frac{v_1v_2 - au_1u_2}{1 - du_1u_2v_1v_2}
 \end{align}$$
 
And the point doubling law:
$$\begin{align}
u' &= \frac{2uv}{1 + du^2v^2}\\
v' &= \frac{v^2 - au^2}{1 - du^2v^2}
 \end{align}$$

Finally, note that on a tE curve, $-P=(-u,v)$, not $(u,-v)$. Further, all of $(\pm u, \pm v)$ lie on the curve, but may or may not not lie on the given subgroup. The point at infinity is denoted $\mathcal O = (0,1)$.

**Lemma 5.4.7**. Let $P=(u,v) \in \mathbb J^{(r)}.$ Then $(u,-v) \not \in \mathbb J^{(r)}$ (subgroup of Jubjub of order r).  

**Proof** 
If $P=\mathcal O_J$ is the point at infinity then $(u,v)=(0,1),$ but $(u,-v)=(0,-1)$ which does not lie on the subgroup. All other points $P$ have odd order. *Because $P$ lies the some subgroup of order $r$, chosen to be an odd prime*.

Further, $v\ne 0$, since if $v=0$:
$$au^2+0^2=1\implies u=\pm \sqrt {1/a}$$
An application of the group doubling law, gives:
$$\begin{align}
u' &= \frac{0}{1 + 0}=0 \\
v' &= \frac{0 - au^2}{1 - 0}=-au^2= -a(1/a)=-1
 \end{align}$$

Which implies that $[2]P=(0,-1)$ (which does not lie on the subgroup, as argued above) then $[2]([2]P)=(0,1)=O$, which obtains $P$ of even order, a further contradiction.

Now, anticipating contradiction, let $P=(u,v), Q=(u,-v)$ be points on the subgroup. By the doubling formula, we have that $[2]Q=-[2]P$:
$$\begin{align}
[2]Q.u &= \frac{2u(-v)}{1 + du^2v^2}=-\frac{2uv}{1 + du^2v^2}=(-[2]P).u\\
[2]Q.v &= \frac{v^2 - au^2}{1 - du^2v^2}=[2]P.v=(-[2]P).v
 \end{align}$$

But also, $[2](-P)=-[2]P$. Therefore either:
- $Q=-P$, a contradiction, since $u=0$ only for $\mathcal O$
- Or, doubling is not injective on the subgroup, which contradict's the subgroup's having odd order.

### Lemma 5.4.7 applied to `MNT_753_4`
Returning to the problem, we might ask why we were able assume that, given $P=(x,y)$ over `MNT_753_4`, why $(x,-y)$ existed? Because the Prover was defined over good old affine, Weierstrass point coordinates, not Twisted Edwards coordinates! 

In the Weierstrass affine coordinate system, we're working over an entirely different group law, and if $(x,y)$ exists on the subgroup, $(x,-y)$ exists too. 

## `puzzle.log()`
*My puzzle solving log, cut short for readability. You might be interested in this as a map of my state of mind while thinking about the puzzle. Keeping a log helps navigate overwhelming context dumps, and to avoid doing wrong and/or stupid things repeatedly. I use Obsidian to take these notes, for which I have written [an extensive setup and usage guide](https://github.com/thor314/obsidian-setup). To see the full log, see [this hackmd](https://hackmd.io/@cryptograthor/ByU_8Sdta).*

After a bit of information gathering, we know a few things:
- curves: we're using the MNT4/6 cycle of curves for our nullifier, which support a pairing function.  Zcash uses the BLS12-381 (pairing-supporting) curves for the nullifier argument. No actually, they use Jub-jub for this, which is pairing-unfriendly.
    - ~~both are pairing friendly~~.
    - MNT embedding degree 4 and 6, ~~while the BLS curves have embedding degree 12. Larger embedding degree means less efficient pairing computation, and a harder DLP~~.
    - ~~BLS has a smaller field size~~.
- nullifier argument:
    - $N:= H(sk,r)$ where N is the nullifier, H the poseidon hash, r a random nonce, and sk the secret key 
        - It does not look like there is a random nonce included included in the hash eval; we see `LEAFH::evaluate(&leaf_crh_params, [leaked_secret])`, where leaf params are just the poseidon config, and leaked secret is the secret key.
    - the zcash-style proof should show:
        - knowledge of (sk, r) such that the nullifier hash is correct
        - knowledge of (v,a,r) such that H(v,a,r)=C - that the coin exists
            - v the note's value, a the address, C a commitment to the coin's existence
        - that nullifier N has not already been published
- Poseidon is an algebraic hash function used in zk circuits. Its parameter selection here looks a little different from the parameters that Zcash selected, though a vuln in Poseidon parameter selection would be surprising to me, given the hints. Two differences noted in param selection, from the zcash spec:
    - partial_rounds = 56 - **diff; partial_rounds = 29**
    - alpha = 5 - **diff; alpha = 17**
    - didn't check the vector of raw values, or the MDS matrix
- Groth16 - the only element of the Groth16 proving system that factors in here is the specification of the SpendCircuit, so it seems unlikely that we'll find any Groth16-related bugs. The circuit checks:
    - outputvar from root
    - paramsvar's for leaf_params and two_to_one_params (which are the same)
    - fpvar witness for secret 
    - outputvar from nullifier
    - g1var constant for g1 generator
    - pathvar witness for proof
    - to construct leaf_g:
        - assign g1var constant for G1 generator (base)
        - construct public key from secret key
        - leaf_g is the public key
    - assert that:
        - secret < MNT6's modulus
        - nullifier hash is correctly calculated
        - leaf_g lies in the merkle tree at index 2
            - recall leaf_g = g1_generator * secret
- Shape of the problem, we have:
    - a merkle tree, with the target leaked secret at index 2, configured to hash with poseidon over the MNT4BigFr Field
    - a nullifier constructed from the hash of the leaked secret
    - we want to construct a cheater-secret such that:
        - secret < MNT6's modulus 
        - nullifier hash is correctly calculated:  
            - `N == LeafHG::evaluate(&params, &[my_secret])?;`
        - leaf_g lies in the merkle tree at index 2
            - recall `leaf_g = (g1_generator * my_secret).x`
    -  in other words, we have: want construct a secret that obtains a hash collision:
        - `h(params, cheat_secret) == h(params, secret)`

``` 
     root
    /    \
h(h0,h1)  h(h2,h3)
|   |     |   |
h0  h1    h2  h3  (this row: leaves.bin)
|   |     |   |
l0  l1    l2  l3
     ^(leaked_secret.bin)
```

### Issue: how are we going to construct a hash collision? 
[hint 1](https://twitter.com/__zkhack__/status/1747574583538925572) updated: see zcash spec lemma 5.4.7 for bob's reasoning on why to only use the x coordinate when storing public keys.

**Lemma 5.4.7**. Let $P=(u,v) \in \mathbb J^{(r)}.$ Then $(u,-v) \not \in \mathbb J^{(r)}$.   
- initial thought: the hint suggests that the x coordinate for the point on the chosen curve may not be unique, i.e. that for $P=(x,y)$, there may exist $y'$ such that $Q=(x,y')$.  Then we would have: 
    - a new nullifier: $N=H(s)$  and $N'=H(s')$ 
    - but with leaf: $l=(s'G).x=(sG).x$, such that the merkle proof verifies correctly.
- by the hint, it seems likely that the point we want is $s'G=(x,-y)$.

Given leaked secret $s$ such that $k=(sG).x$, we want to obtain $s'$ such that:
$$k'=(s'G).x=(x,-y).x = x = (x,y).x = (sG).x =k$$
Is it possible that $(-s)G=(x,-y)$? Seems unlikely. Checks: nope. Let's read the proof very carefully.

It's not immediately obvious how we will peel off $s'$, i.e. solve the discrete log problem. Jubjub, the zcash curve which the lemma describes, does not have a pairing. Can we exploit the pairing? 
$$e(s'*G_1, G_2)=e(G_1, s'*G_2)= s'*e(G_1,G_2)=s'G_T$$
Maybe if solving the discrete log problem in $G_T$ or $G_2$ is easy, but still, that feels unlikely to be it. But Jubjub isn't pairing friendly; mnt4/6 is, so that feels like a big hint that we just haven't figured out how to use yet. Might be worth throwing an hour at the whiteboard.

Still, we've made some progress. The state of the problem is now:
- we want to obtain $s'$ such that $s'G=(x,-y).x$ where $(x,y).x=sG$. Solving for $s'$ directly would involve solving the discrete log problem.
- we're pretty sure that there's an exploit on the pairing somehow to help us solve for $s'$.

### Lemma and proof
**Lemma 5.4.7**. Let $P=(u,v) \in \mathbb J^{(r)}.$ Then $(u,-v) \not \in \mathbb J^{(r)}$ (subgroup of Jubjub of order r).   Proof: 
- if P is the point at infinity (u,v)=(0,1), but -P=(0,-1) which does not lie on the curve.
- all other points P have odd order.
    - further, $v\ne 0$. if v=0, then $au^2=1$ then $[2]P=(0,-1)$ then $[2]([2]P)=(0,1)=O$, which would have even order.
        - *unsure what the $au^2=1$ argument is saying. $a$ is part of the curve equation, $y^2=x^3 +ax + b$*. Check: $0=u^3+au+b$--nope. Maybe there's another curve representation they're working over, something twisted-ed related. Oh, u,v denotes twisted edwards notation.
    - if $v\ne 0$ then $v\ne -v$.
    - Let $Q=(u,-v)$ be a point on the curve.  We will show $Q=-P$ is a contradiction.
        - Then $[2]Q=-[2]P=[2](-P)$, but $Q.v=-P.v$ is contradiction since $-v \ne v$  
            - *what? Should expand on that* 
        - Therefore either $Q=-P$ or else doubling is not injective on the curve, which contradicts the curve's odd order.

Hint 2 released:
> Note that the MNT curves are represented in Weierstrass from in the circuit, and use the fact that both (x,y) and (x,-y) are valid points for spending

Which confirms my suspicion that we want $s'=(x,-y)$. Let's find it.

### algebraic insight!
- $P=sG=(x,y)$
- $-P=O-P=O-sG=s'G=(x,-y)$

We observe that for the order $n$ of generator $G$, we have $nG=O$, therefore:

$$\begin{align}
s'G &= O-sG = nG-sG\\
&=(n-s)G\\
\therefore s'&=(n-s)
 \end{align}$$
However, note that $s$ lives in the `mnt4::Fr` field, but the order $n$ of generator $G$ will live in `mnt4::Fp`, so we'll have to do a type conversion.
## footnotes
[^1]:  Unrelated to this puzzle, the Zcash nullifier argument also includes a nonce, so that a public key can be used more than once: $N=H(pk,r)$.
[^2]: Unrelated to this puzzle, the Zcash nullifier must prove two other conditions:
- that nullifier $N$ has not already been published--this is not included in the proof for the puzzle, but is checked elsewhere in the puzzle driver; but, would need to be checked in the proof in real life
- the existence of the note(s) (coins) that will be spent, via proof of knowledge of:
    - $v$, the value of the note
    - $a$ the address possessing the note
    - $r$ the random nonce for the note
[^3]: A perhaps interesting note dump on Twisted Edwards and Montgomory Curves: 
Every montgomery form elliptic curve has a birational Twisted Edwards equivalent. Edwards curves have fast addition algorithms, and Twisted Edwards curves are nearly as fast, enabling faster point operations for many curves currently in use. The twist lies in the $a$ parameter, without which, many Montgomory curves have no Edwards equivalent. 
$$\begin{align}
\newcommand{Edwards}{u^2+v^2=1+u^2v^2}
\text{Edwards form:}\quad &\Edwards\\
\newcommand{TwistedEdwards}{a\Edwards}
\text{Twisted Edwards form:}\quad &\TwistedEdwards
 \end{align}$$
where $a$ and $d$ are constants in the field over which the curve is defined, and $a \neq d$. These curves are a generalization of Edwards curves (where $a=1$) and are used in cryptography for efficient arithmetic operations. Typically, $c=1$, and is often excluded.

Montgomery elliptic curves are defined by the equation:
$$By^2 = x^3 + Ax^2 + x$$
where $A$ and $B$ are constants in the field over which the curve is defined, with $B \neq 0$ and $A^2 \neq 4$, over a field $k$ with $\char k \ne 2$. These curves are used in cryptography for their efficient point multiplication properties.