# How BLS works

## Basic Formula

```text
Private key:  sk (secret number)
Public key:   PK = sk * G  (G is a fixed point)
Signature:    σ = sk * H(message)  (H hashes message to curve point)
```

Example:

```text
Alice signs:    σ₁ = sk₁ * H(message)
Bob signs:      σ₂ = sk₂ * H(message)
Charlie signs:  σ₃ = sk₃ * H(message)

Combined signature: σ_total = σ₁ + σ₂ + σ₃
Combined public key: PK_total = PK₁ + PK₂ + PK₃
```

## Use case

### Scenario: Blockchain Consensus

```text
textMessage: "Block #1000 is valid"
```

**Each Validator Signs**

```text
textValidator A: σ_A = sk_A *H("Block #1000 is valid")
Validator B: σ_B = sk_B* H("Block #1000 is valid")  
Validator C: σ_C = sk_C * H("Block #1000 is valid")
```

**Network Combines Signatures**

```text
textFinal signature: σ_final = σ_A + σ_B + σ_C
Final public key: PK_final = PK_A + PK_B + PK_C
```

## Benefits

1. Traditional Signatures

```text
100 validators = 100 separate signatures to store and verify
Takes more space and computation
```

2. BLS Signatures

```text
100 validators = 1 aggregated signature
Proves all 100 signed without storing 100 signatures
```
