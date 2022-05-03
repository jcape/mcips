- Feature Name: `janus_mitigation`
- Start Date: 2022-03-28
- MCIP PR: [mobilecoinfoundation/mcips#0000](https://github.com/mobilecoinfoundation/mcips/pull/0000)
- Tracking Issue: (link to tracking issue)

# Summary
[summary]: #summary

We should adjust wallet-side transaction construction standards to mitigate the [Janus attack](https://web.getmonero.org/2019/10/18/subaddress-janus.html). Specifically, set the fog hint's ephemeral key equal to the txout public key's 'base key'.

# Motivation
[motivation]: #motivation

The [Janus attack](https://web.getmonero.org/2019/10/18/subaddress-janus.html) allows users to craft a malicious transaction which will expose whether two subaddresses belong to the same account. Let's start with Alice using her wallet to generate multiple subaddresses (private keys/scalars are lower case, public keys/points are capitalized):

1. **Alice** has a wallet, _a_, with private keys: _a<sub>view</sub>_, and _a<sub>spend</sub>_

2. **Alice** generates their default subaddress _spend_ private key, _d<sub>spend</sub>_:
> _d<sub>spend</sub>_ = _a<sub>spend</sub>_ + BLAKE2B_512("mc_subaddress" || _a<sub>view</sub>_ || `0x0000_0000_0000_0000_0000_0000_0000_0000_0000_0000_0000_0000_0000_0000_0000_0000`)
4. **Alice** generates their default subaddress _view_ key, _d<sub>view</sub>_:
> _d<sub>view</sub>_ = _d<sub>spend</sub>_ * _a<sub>view</sub>_

6. **Alice** generates the corresponding public keys _D<sub>view</sub>_ and _D<sub>spend</sub>_ using the normal method (curve's basepoint, _g_, raised to the corresponding private key/scalar):

> _D<sub>view</sub>_ = _g<sup>d<sub>view</sub></sup>_

> _D<sub>spend</sub>_ = _g<sup>d<sub>spend</sub></sup>_

8. **Alice** now has her subaddress _A<sub>D</sub>_ = (_D<sub>view</sub>_, _D<sub>spend</sub>_), which she publishes as _D_.

9. **Alice** creates a new subaddress pair, _E_, using `0x0000_0000_0000_0000_0000_0000_0000_0000_0000_0000_0000_0000_0000_0000_0000_000f`, instead of all zeroes.

At this point, **Alice** has published two subaddresses: _D_, and _E_. **Chad** suspects that _D_ and _E_ (the information he has) are actually A<sub>D</sub> and A<sub>E</sub>, and would like to test this hypothesis. If **Chad** was undamaged, he would create a transaction using the usual rules for creating a MobileCoin transaction to _E_:

1. **Chad** creates a fog hint encrypting _E_:
> 1. _F_ = RESOLVE_FOG_PUBKEY(_E_)
> 2. 
3. 

Unfortunately, **Chad** wants to invade **Alice**'s privacy, and so he will create a transaction mixing the view and spend keys in the wrong locations:

9. **Chad** constructs a

10. **Alice** generates a second subaddress, _Z_ = SUB(_a<sub>view</sub>_, _a<sub>spend</sub>_, 1).
5An attacker, **Darth**, suspects that _A_ and _Z_ are both subaddresses for the wallet _a_.
6They generate a new transaction which mixes the view key of _A_ and the spend key of _Z_.

7A malicious user suspects two subaddresses (`A` and `B`) with (`a` and `b`) belong to the same wallet.
8The malicious user will construct a transaction output for the recipient:
   1. Build a shared secret using using the public view key of `A`: A<sub>view</sub>
   2. Build the one-time address using the spend key of `B`.
   3. Build the TXO public key using the spend key of `A`.


```
address A: K^{v,A}, K^{s,A}
address B: K^{v,B}, K^{s,B}
sender-receiver shared secret: shared_secret = r_i K^{v,A}
one-time address: K^o = H(shared_secret) G + K^{s,B}
txout public key: r_i K^{s,A}
```

3. The output recipient will interpret this incorrectly:
   1. The shared secret will be scanned and match against the view subaddress `A`
   2. The spend key will match subaddress `B`
   3. The user will conclude the money belongs to `B`, even though it needed both `A` and `B` to find it.

```
sender-receiver shared secret: shared_secret = k^v * r_j K^{s,A}
nominal spend key: K^s_nom = K^o - H(shared_secret) G
if: K^s_nom matches subaddress spend key B in [set of subaddress spend keys]
then: the output is owned by subaddress B
```

3. If the recipient discloses they have received the transaction, then the sender will know that subaddresses A and B belong to the same account (i.e. were constructed from the same private key pair `k^v, k^s`).

There are currently no mitigations to this attack implemented in MobileCoin.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation


When constructing a transaction:

- Instead of generating separate values `r_i` and `r_fog` for the txout private key and fog hint ephemeral key, respectively, generate one value `r_i`. Let the fog hint ephemeral key equal `r_i G`.

> Explain the proposal as if it was already included in the system and you were teaching it to a user. That generally means:
>
> - Introducing new named concepts
> - Explaining the feature largely in terms of examples.
> - Explaining how users should *think* about the feature, and how it should impact the way they use MobileCoin. It should explain the impact as concretely as possible.
> - If applicable, provide sample error messages, deprecation warnings, or migration guidance.
> - If applicable, describe the differences between teaching this to existing users and new users.
>
> For implementation-oriented MCIPs (e.g. for ledger formats), this section should focus on how other contributors should think about the change, and give examples of its concrete impact. For policy MCIPs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

> This is the technical portion of the MCIP. Explain the design in sufficient detail that:
>
> - Its interaction with other features is clear.
> - It is reasonably clear how the feature would be implemented.
> - Corner cases are dissected by example.
>
> The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

> Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

> - Why is this design the best in the space of possible designs?
> - What other designs have been considered and what is the rationale for not choosing them?
> - What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

> Discuss prior art, both the good and the bad, in relation to this proposal.
> A few examples of what this can include are:
>
> - For consensus and fog proposals: Does this feature exist in other systems, and what experience have their community had?
> - For community proposals: Is this done by some other community and what were their experiences with it?
> - For other teams: What lessons can we learn from what other communities have done here?
> - Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.
>
> This section is intended to encourage you as an author to think about the lessons from other systems, provide readers of your MCIP with a fuller picture.
> If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other systems.
>
> Note that while precedent set by other systems is some motivation, it does not on its own motivate an MCIP.
> Please also take into consideration that MobileCoin sometimes intentionally diverges from common cryptocurrency features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

> - What parts of the design do you expect to resolve through the MCIP process before this gets merged?
> - What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
> - What related issues do you consider out of scope for this MCIP that could be addressed in the future independently of the solution that comes out of this MCIP?

# Future possibilities
[future-possibilities]: #future-possibilities

> Think about what the natural extension and evolution of your proposal would
> be and how it would affect the project as a whole in a holistic way. Try to
> use this section as a tool to more fully consider all possible interactions
> with aspects of the project in your proposal. Also consider how this all
> fits into the roadmap for the project and of the relevant team.
>
> This is also a good place to "dump ideas", if they are out of scope for the
> MCIP you are writing but otherwise related.
>
> If you have tried and cannot think of any future possibilities,
> you may simply state that you cannot think of anything.
>
> Note that having something written down in the future-possibilities section
> is not a reason to accept the current or a future MCIP; such notes should be
> in the section on motivation or rationale in this or subsequent MCIPs.
> The section merely provides additional information.

