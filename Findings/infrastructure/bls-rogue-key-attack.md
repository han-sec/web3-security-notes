# 📌 BLS Rogue Key Attack – A Subtle but Serious Aggregation Risk

When verifying aggregated BLS signatures from a committee, it's easy to assume the public key set is honest. But without proper `proof of possession (PoP)`, a rogue actor can exploit this and silently take over the signature.

## 🧵 Here's how

1. In BLS, signatures from multiple validators can be aggregated into a single signature verified against an aggregated public key:
`e(agg_sig, G2) = e(H(m), agg_pk)`

2. But if one malicious validator crafts their public key as a function of others, like:
`PK_mal = PK_target - (PK_A + PK_B + PK_C)`
then the overall aggregated key becomes just PK_target.

3. This means the attacker can forge a valid signature alone, but make it look like it came from the full committee. No threshold met. Just manipulation.

4. Without enforcing PoP, the protocol can't distinguish between a legitimate committee and a single rogue key mimicking the committee.

🛡️ Mitigation: Always enforce Proof of Possession for BLS public keys during registration. This ensures validators actually hold the corresponding secret keys.

💥 Without it, even the most elegant signature scheme is vulnerable to subtle but catastrophic compromise.

More reading: [Rogue Key Attack](https://xn--2-umb.com/22/bls-signatures/#rogue-key-attack)
