---
title: "Solving ZK Hack IV puzzle 2 - Supervillain"
date: 2024-01-24
last-update: 2024-01-24
draft: false
tags: ["cryptography", "puzzle", "zk", "pairing", "zkhack"]
categories: ["cryptography"]
---
# Solving ZK Hack IV puzzle 2 - Supervillain
![](/photos/2024-01-24-Solving/supervillain-zkhack-puzzle.jpeg)

*What follows is a solution to [ZK Hack IV, Puzzle 2](https://zkhack.dev/zkhackIV/puzzleF2.html). ZK Hack is a series of zero-knowledge cryptography CTFs, workshops, hackathons, and study groups. Thanks to the ZK Hack organizers for creating this CTF, Geometry for this puzzle, and the ZK Hack community. You might also be interested in [solution for puzzle 1](https://thork.net/posts/2024-01-18-solving-zk-hack-iv-puzzle-1/). If reposting elsewhere, find this post on my blog [here](http://thork.net/posts/2024-01-24-solving-zk-hack-iv-puzzle-2-supervillain/)*.


You might be interested in this post if you're interested in:
 - a smattering of zk-related trivia about elliptic curve pairing assumptions and proofs of knowledge
 - The process of (cryptographic) puzzle solving

*Recommended reading order*:
- read this post top to bottom for a concise description and solution of the puzzle, then some mathematical detail nuggets to hold onto, and finally the stream of consciousness log of problem solving.
- read this post back to front for the order in which it was actually written (problem solving log, then cleaned up and bloggified knowledge nuggets).

## `puzzle.solve()`
### Background
*all the knowledge nuggets you need to solve the puzzle*

In this puzzle we are tasked with exploiting a BLS Signature scheme with a home-baked proof of knowledge argument. 

#### BLS background
The BLS signature scheme is a simple signature aggregation scheme, relying on a elliptic curve pairing. It consists of 3 algorithms, (sign, aggregate_signatures, verify), where verify checks either any individual's signature, or the whole aggregated signature.
- **Individual signing** - Each individual signature $\sigma_i=sk_i\cdot H(M)$ is curve point on $\mathbb G_2$. Each public key $pk_i=sk_i\cdot G_1$ is a curve point on $\mathbb G_1$. 
- **Individual signature verification** and **aggregated verification** - Verification for any individual signature $\sigma_i$ is the same as for the entire signature, so we drop the indices: 
$$e(pk, H(M)) = e(G_1,\sigma)$$
Note that we don't typically verify the individual signatures, only the final aggregated signature. The point of a signature aggregation scheme is that *the aggregated signature is valid if and only if each component signature is valid*.

- **Signature aggregation** - Simply compute the sum of the signatures and sum of public keys:

$$\begin{aligned}
\sigma_{\text{agg}} &= \sum\limits_{i=1}^{n} \sigma_i=(\sum\limits sk_i)H(M) \\\
pk_{\text{agg}}&=\sum\limits pk_i=(\sum\limits sk_i)G_1
\end{aligned}$$
#### Proof of knowledge argument background
As noted above, we don't verify the individual signatures. A malicious party could therefore choose public key without even knowing the corresponding secret key. The proof of knowledge intends to prevent that.

Instead, we require each party $i$ to provide a proof of knowledge $\pi_i$ that the party knows their secret key. The proof relies on a pairing check:

$$e(pk_i, R_i)= e(G_1, \pi_i)$$
Where $R_i$, is randomly generated within $\mathbb G_2$. That is, we check that the party knows $sk_i$ such that they can compute:

$$\pi_i = sk_i*R_i$$
#### Back to the problem at hand
In the case of this problem, the implementer modifies the BLS signature verification algorithm, negating the message hash:

$$\begin{aligned}
e(pk, -H(M)) &= e(G_1,\sigma) \\\
e\bigg(\sum\limits(sk_i)G_1, -H(M)\bigg) &= e\bigg(G_1,\sum\limits(sk_i)H(M)\bigg)
\end{aligned}$$
This constrains the choice of our secret key $sk'$ such that (note that we start with 8 given public keys, and we are to choose the 9th):

$$sk = sk' + \sum\limits^7_0 sk_i$$
and 

$$-sk * e(..) = sk * e(..)\implies 2 * sk = p$$

where $p$ is the prime modulus of our field. Since $p$ is odd, the group $sk=0$ and $pk= sk * G = \mathcal O$. We don't have the others' secret keys, but we do have their public keys, from which we may construct our own key $pk'$:

$$pk'= \mathcal O - \sum\limits_0^7 pk_i$$
And the aggregate signature: 

$$\sigma_{agg} = sk_{agg} * H(M) = 0 * H(M) = \mathcal O$$

However, we do not have a way obtain our own secret key $sk'$ without breaking the discrete log assumption, leads to issues with the proof-of-knowledge if the PoK is [correctly implemented](https://github.com/thor314/puzzle-supervillain/blob/main/src/main.rs#L27C1-L44C2).
```rust
/// generate random point R
fn derive_point_for_pok(i: usize) -> G2Affine {
    let rng = &mut ark_std::rand::rngs::StdRng::seed_from_u64(20399u64);
    G2Affine::rand(rng).mul(Fr::from(i as u64 + 1)).into()
}

/// verify proof of knowledge
fn pok_verify(pk: G1Affine, i: usize, proof: G2Affine) {
    assert!(Bls12_381::multi_pairing(
        [pk, G1Affine::generator()],
        [derive_point_for_pok(i).neg(), // -R_i, not R_i
        proof]
    ).is_zero());
}
```

Note that instead of:

$$e(pk_i, R_i) = e(G_1, \pi_i)$$
We have:

$$e(pk_i, -R_i)= e(G_1, \pi_i)$$
So we actually need $\pi = -sk_i * R_i$, not $\pi=sk_i*R_i$. Take a moment consider whether this changes anything.[^1]

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
Let's revisit that randomness algorithm.
```rust
/// generate random point R
fn derive_point_for_pok(i: usize) -> G2Affine {
    let rng = &mut ark_std::rand::rngs::StdRng::seed_from_u64(20399u64);
    G2Affine::rand(rng).mul(Fr::from(i as u64 + 1)).into()
}
```

That is, let $R\gets\mathbb G_2$, then:

$$R_i = (i+1)R$$
ₒₕ ₙₒ

You thought this would be harder didn't you? I certainly did. Well, a sanity-check test is in [order](https://github.com/thor314/puzzle-supervillain/blob/8b967f9e8640c25fb26e5df13c0cdd50c4d503fc/src/main.rs#L167C1-L183C2):
```rust
#[test]
fn test_rng() {
    // let G = G1Projective::generator().into_affine();
    let rng = &mut ark_std::rand::rngs::StdRng::seed_from_u64(20399u64);
    let R = G2Affine::rand(rng);
    let R_1 = R.mul(Fr::from(2));
    let R_9 = R.mul(Fr::from(10));
    assert_eq!(derive_point_for_pok(1), R_1); // pass
    assert_eq!(derive_point_for_pok(9), R_9); // pass
    use ark_ff::Field;
    let five_inv = Fr::from(5u64).inverse().unwrap();
    let R_9_to_1 = R_9.into_affine().mul(five_inv).into_affine();
    assert_eq!(R_1, R_9_to_1); // pass
}
```
*a test on the malleability in the randomness*

This shows that, by the malleability of the randomness, we can solve for our $\pi_9$ by a little algebra on $\{\pi_i\}$.[^2] We have:

$$\begin{aligned} 
\pi_9 &= sk_9 \cdot R_9 \\\
&= (-\sum\limits_1^8 sk_i)\cdot((1+9)R) \\\
&= -10*\sum\limits_1^8 sk_i\frac{R_i}{i+1}\qquad \text{as}\ R=R_i/(i+1)\\\
&= -10*\sum\limits_1^8 \frac{\pi_i}{i+1}\qquad \text{as}\ \pi_i=sk_i\cdot R_i\\\
 \end{aligned}$$
 Which we may then simply [implement](https://github.com/thor314/puzzle-supervillain/blob/8b967f9e8640c25fb26e5df13c0cdd50c4d503fc/src/main.rs#L142C1-L161C1):
 ```rust
 fn extract_proof(new_key: &G1Affine, pis: Vec<G2Affine>) -> G2Affine {
    use ark_ff::Field;
    let pi_sum = pis
        .into_iter()
        .enumerate()
        .map(|(i, pi_i)| {
            let coef = {
                debug!("i+1: {}", i + 1);
                let inv = Fr::from(i as u32 + 1u32).inverse().unwrap();
                Fr::from(10) * inv
            };
            pi_i * coef
        })
        .sum::<G2Projective>()
        .into_affine();

    // pi_9 = -sum(pi_i * (10/(i+1))
    -pi_sum
}
```

And then enjoy a pamplemousse sparkling water to celebrate our triumph.
![](/photos/2024-01-24-Solving/pamplemousse_glory.jpeg)
*dalle, upon being asked for triumphant pamplemousse*

## `puzzle.deets()` 
*extended background on the B-KEA and KOSK assumptions*

The puzzle gives a pair of resources that I didn't mention. Resources like these tend to assume things like "you won't stick a knife in your gut on purpose by choosing a malleable source of randomness". But we did do that, so the resources are maybe only helpful for a little background context for the problem. 

Therefore, the following notes are short and somewhat irrelevant to solving the problem, but may be interesting tidbits to take with you.

Mary Maller's paper gives the Bilinear Knowledge of Exponent Assumption (B-KEA).  The B-KEA can be succinctly defined as follows: Given a bilinear group $(G_1, G_2, G_T)$ with a bilinear map $e: G_1 \times G_2 \rightarrow G_T$, and elements $G \in G_1, H \in G_2, xH \in G_2$ for some unknown $x$, if there exists an algorithm that outputs $yG \in G_1$ and $T \in G_T$ such that $e(yG, H) = T$, then there exists an extractor that can compute $y$ given $yG$ and $T$. This assumption implies that the algorithm "knows" the exponent $y$ used to compute $yG$. That is, the B-KEA assumption says that, unless we can break the DLP, we're not going to be able to extract our secret key $sk'$ from our chosen public key $pk'$. 

This is maybe a bummer for hacking the problem, but on the whole, net good that we can use pairings in cryptography and not break all our private keys.

The second paper gives the Knowledge of Secret Key (KOSK) assumption, that it is assumed that the possession and use of a secret key is proof of the identity of the user. 

Formally, let $SK$ be a secret key and $PK$ be the corresponding public key. The KOSK Assumption states that if an entity can perform cryptographic operations requiring $SK$ (like decrypting a message encrypted with $PK$ or signing a message verifiable with $PK$), then this entity possesses $SK$.

In other words, if someone can use $SK$, they are assumed to be the legitimate owner of $SK$.

The remainder of Paper 2 gives a series of remarks on multiparty signature schemes with a more in depth coverage of BLS' security than is absolutely necessary to be familiar with.

![](/photos/2024-01-24-Solving/algebra_hammer.jpeg)

## `puzzle.log()`
*My puzzle solving log, lightly edited for readability. You might be interested in this as a map of my state of mind while thinking about the puzzle. Keeping a log helps navigate overwhelming context dumps, and to avoid doing wrong and/or stupid things repeatedly. I use Obsidian to take these notes, for which I have written [an extensive setup and usage guide](https://github.com/thor314/obsidian-setup). My repo can be found [here](https://github.com/thor314/puzzle-supervillain/blob/main/src/main.rs). For taking math notes, I generally use a whiteboard or ipad for my scratch space before writing down my clean notes here, which helps to think n scratch faster.*

### initial problem description
We are tasked with forging a signature.
> Bob has been designing a new optimized signature scheme for his L1 based on BLS signatures. Specifically, he wanted to be able to use the most efficient form of BLS signature aggregation, where you just add the signatures together rather than having to delinearize them. In order to do that, he designed a proof-of-possession scheme based on the B-KEA assumption he found in the the Sapling security analysis paper by Mary Maller [1]. Based the reasoning in the Power of Proofs-of-Possession paper [2], he concluded that his scheme would be secure. After he deployed the protocol, he found it was attacked and there was a malicious block entered the system, fooling all the light nodes...

1: https://github.com/zcash/sapling-security-analysis/blob/master/MaryMallerUpdated.pdf

2: https://rist.tech.cornell.edu/papers/pkreg.pdf

noted keywords:
- delinearize bls signatures
- B-KEA - 1
- proof of possession - 2

 Immediate candidates for bugs:
- `pok_verify` - perhaps there's a corner case that `pok_verify` doesn't check
- aggregation and verification - does it match the BLS spec aggregation step? 

The PoK seems more suss than bls, since the we've been informed that the implementer designed their own PoK scheme.

### initial things to look at, after skimming code
- aggregates public keys correctly? - yes
- aggregate_signature - huh, we construct this, instead of just publishing our individual signature? that's one vector for not actually knowing our secret, which leaves `pok_verify` as the only other measure
- `bls_verify` verifies sig agg correctly? - yes
- `pok_verify` verifies individually proofs correctly? - yes
### notes on BLS signatures
*a good portion of this comes from copy-pasting my existing notes (e.g. on BLS signatures), and/or telling chatGPT to write a good note on topic for me and editting it*

BLS signature aggregation alg:
#### Individual signing
Each individual signature $\sigma_i=sk_i\cdot H(M)$ is curve point on G2. Each public key $pk_i=sk_i\cdot G$ is a curve point on G1. 
#### Individual verification and BLS verification
Verification for any individual signature $\sigma_i$ is the same as for the entire signature, so we drop the indices: 
$$e(pk, H(M) = e(G_1,\sigma)$$
#### Signature aggregation
Simply compute the sum of the signatures and sum of public keys:
$$\sigma_{\text{agg}} = \sum\limits_{i=1}^{n} \sigma_i=(\sum\limits sk_i)H(M)$$
$$pk_{\text{agg}}=\sum\limits pk_i=(\sum\limits sk_i)G$$

### BLS code check
We don't test the individual signatures for correctness. This seems weird, as noted above.

#### public keys look correctly aggregated
```rust
  let aggregate_key =
    public_keys.iter().fold(G1Projective::from(new_key), |acc, (pk, _)| acc + pk).into_affine();
```

But we get to choose the aggregated signature, rather than submitting an individual signature, which seems like an attack vector for sure.

#### bls signing check
```rust
#[allow(dead_code)]
fn bls_sign(sk: Fr, msg: &[u8]) -> G2Affine { hasher().hash(msg).unwrap().mul(sk).into_affine() }
```
matches signing algorithm given above, $\sigma_i=sk_i*H(M)$
#### bls verify check 
recall that we don't verify individual signatures. We have aggregated $pk=G * sk$ and $\sigma=sk * H(M)$. We compute 2 pairings to check:

$$e(pk,H(M))= sk* e(G,H(M))=e(G,\sigma)$$

bls verification code:
```rust
/// multi_pairing: vec<g1>.zip(vec<g2>).map(pairing) and then subtracts one pairing from the other
fn bls_verify(pk: G1Affine, sig: G2Affine, msg: &[u8]) {
// what's deal with `neg`? Looks off
  assert!(Bls12_381::multi_pairing([pk, G1Affine::generator()], [
    hasher().hash(msg).unwrap()
    .neg(), // what? 
    sig
  ])
  .is_zero());
}
```

Looks like we're taking a negative for some reason. That means our pairing looks like:
$$e(pk, -H(M)) = e(G,\sigma)$$
but $\sigma=sk * H(M)$, not $-sk * H(M)$, so there's only a few values $sk$ could have:
$$sk = -sk \mod N \implies 2sk = N $$
where $N$ is the modulus of field `Fr` that $sk$ lies in. Since $N$ is an odd prime (has no 2 divisor), we have that the aggregate $sk=0$. Our agg sig is then:
$$\sigma= sk*H(M) = 0 * H(M)=\mathcal O$$
our shared public key is also just $\mathcal O$, so we choose our individual public key:
$$pk_9 = \mathcal O - \sum\limits_0^8 pk_i$$
#### first shot
Do we win?![^3]
```rust
pub fn get_hack(
    pks: Vec<(G1Affine, G2Affine)>,
    msg: &[u8],
) -> (G1Affine, G2Affine, G2Affine) {
    let new_key = pks
        .iter()
        .fold(G1Projective::zero(), |acc, (pk, _)| acc - pk)
        .into();
    // dunno about proof
    let new_proof = G2Projective::generator().mul(Fr::zero()).into_affine();
    let aggregate_signature = G2Affine::zero();
    debug!("new_key: {:?}", new_key);
    (new_key, new_proof, aggregate_signature)
}
```
*[first attempt](https://github.com/thor314/puzzle-supervillain/blob/main/src/main.rs#L198) at soln*

Still need to figure out what to put in proof $\pi$.

Do we at least satisfy the [BLS check](https://github.com/thor314/puzzle-supervillain/blob/8b967f9e8640c25fb26e5df13c0cdd50c4d503fc/src/main.rs#L90C1-L102C60)? 
```rust
/* Enter solution here */
let (new_key, new_proof, aggregate_signature) = get_hack(public_keys.clone(), message);
/* End of solution */

// pok_verify(new_key, new_key_index, new_proof); // not happy
debug!("pok verified successfully"); 

let aggregate_key = public_keys
    .iter()
    .fold(G1Projective::from(new_key), |acc, (pk, _)| acc + pk)
    .into_affine();
debug!("agg key created successfully");
bls_verify(aggregate_key, aggregate_signature, message)
warn!("win");
```
*puzzle main, sans pok_ver which we haven't gotten yet*

We do. Very cool. Note that we [tried some other stuff](https://github.com/thor314/puzzle-supervillain/blob/8b967f9e8640c25fb26e5df13c0cdd50c4d503fc/src/main.rs#L226C1-L264C2)  to see if we could satisfy `bls_verify` in some other creative ways too.

### but how do we satisfy `pok_verify`
#### pok verify code:
```rust
fn pok_verify(pk: G1Affine, i: usize, proof: G2Affine) {
  assert!(Bls12_381::multi_pairing([pk, G1Affine::generator()], [
    derive_point_for_pok(i).neg(),
    proof
  ])
  .is_zero());
}
```

which corresponds to 
$$e(pk_9, -R_9 = e(G,\pi_9)$$
So we need $\pi_9= -sk_9*R_9$ (note that 9 is the index of my key).

They actually give us a nice convenience helper to write proofs, thanks Kobi![^4]
```rust
#[allow(dead_code)]
fn pok_prove(sk: Fr, i: usize) -> G2Affine { derive_point_for_pok(i).mul(sk).into() }
```

which would imply that $\pi =R*sk$, dropping the negative. Huh. Idk. There's very possibly a negative bug somewhere in my life.

We either need to derive $pk_9$ by means unknown, or else, if the proofs are malleable by some twisted algebra, we win. The proofs don't look malleable, and the rng doesn't look game-able.[^5]

To the reading!
### he looks to the reading
revisit keywords:
- delinearize BLS signatures
    - neither paper uses the word. chatGPT is reluctant and handwavey to define it. only few [mentions](https://github.com/w3f/bls) appear in a search. chat suggests that it involves checking an additional pairing constraint on a secondary hash function, meaning our OG hash function could be underconstrained.
- B-KEA - feed the paper to chat, who says:
    - The Bilinear Knowledge of Exponent Assumption (B-KEA) is defined over group elements and a bilinear map. It states that, if there exists an algorithm that can produce a certain group element in a bilinear group setting, then there also exists an extractor that can compute a related exponent.
    - That basically says, if we can produce the faulty proof $\pi$, then we can extract $sk$ and break the discrete log problem. That seems bearish.
- proof of possession - feed the paper to chat, who says:
    - Proofs of Possession (PoPs) are mechanisms used to demonstrate that a party possesses a secret key corresponding to a given public key, without revealing the secret key itself. PoPs typically involve the party generating a cryptographic proof, such as a signature using their private key, and then presenting this proof alongside their public key. The verifier checks the proof to ensure that it was indeed created with the corresponding private key. This method ensures that a party cannot claim to own a public key without actually possessing the associated private key.
        - that is indeed what we're trying to break. We want $\pi=sk*-R$, which we can only compute if already have `sk`. We have `pk` we want to derive `sk` to break the PoP.

#### Interlude
*I read the papers some more, and found them not entirely helpful. Their proofs asserted that everything was fine, which seemed untrue. So I screwed around on the whiteboard a bit more to commit the equations to heart before going to bed. In the morning, as I lay in bed dreaming of algebra, there was insight. I ran to the algebra machine*.

#### honey, get the randomness malleability hammer
```rust
// on second glance, the random nonces are less random than they once seemed 🙊
#[test]
fn test_rng() {
    // let G = G1Projective::generator().into_affine();
    let rng = &mut ark_std::rand::rngs::StdRng::seed_from_u64(20399u64);
    let R = G2Affine::rand(rng);
    let R_1 = R.mul(Fr::from(2));
    let R_9 = R.mul(Fr::from(10));
    assert_eq!(derive_point_for_pok(1), R_1); // pass
    assert_eq!(derive_point_for_pok(9), R_9); // pass
    use ark_ff::Field;
    let five_inv = Fr::from(5u64).inverse().unwrap();
    let R_9_to_1 = R_9.into_affine().mul(five_inv).into_affine();
    assert_eq!(R_1, R_9_to_1); // pass
                               // oh fun, we nearly done
                               //
                               // we can construct stuff.
}
```
*[not so random](https://github.com/thor314/puzzle-supervillain/blob/8b967f9e8640c25fb26e5df13c0cdd50c4d503fc/src/main.rs#L166C1-L183C2) anymore!*

described in greater depth in `puzzle.solve()` above.

## changelog
- 2024-01-23 - majority of log written
- 2024-01-24 
    - solved the problem while sleeping, woke up with divine insight
    - copy log over, edited for readability
    - wrote solve() and deets()
## footnotes
[^1]: It doesn't. We can choose $\pi$ however we want.
[^2]: Sorry, I play a little fast and loose with whether I start from 1 or 0, so don't pay too much attention to the indices.
[^3]: He did not win.
[^4]: Woe unto thee, Kobi, architect of ledgers and guardian of chains! Kobi you fool, for your help, I shall betray you! Your fields shall burn as I deceive your L2 light clients! We hack-eth your PoK and thus rain fire and brimstone upon nodes! With this hammer, I do decentralize your walls and scatter shards, crypto-Canaan anew. As Pharaoh's heart was hardened, so shall be the fate of thy PoK.
[^5]: Hard enough, he did not look.
