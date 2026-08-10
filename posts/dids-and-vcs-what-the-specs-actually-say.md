---
title: "DIDs and VCs: What the Specs Actually Say"
---

---

You don't own your Twitter username. The company does. If the platform shuts down tomorrow, that identity is gone — you can recreate it somewhere else, but it's not the same. Your followers, your history, your reputation — tied to a string that a corporation controls.

This is a common but mostly ignored problem. And it applies to more than social media. Your GitHub account, your email, your developer identity — all of it lives on servers you don't control, owned by companies that can revoke access, change the rules, or simply disappear.

DIDs — Decentralized Identifiers — fix this by giving you a cryptographic identifier that you control. No company in the middle. No account to get banned. Just a keypair that's yours.

## What is a DID?

A DID is just a string. Three parts: the scheme (`did`), the method (`example`, `hedera`, `key`), and a unique identifier.

The method tells you *where* this DID lives and *how* to look it up. `did:hedera:mainnet:abc123` lives on Hedera. `did:key:abc123` is self-contained in the string itself. Different methods, same concept.

The string alone is useless. What matters is what it *resolves to* — the DID Document. Think of resolution like DNS: you put in a domain name, you get back an IP address. You put in a DID, you get back a document describing who controls it.

That document looks like this:

```json
{
  "id": "did:example:123456789abcdefghi",
  "authentication": [{
    "type": "Multikey",
    "controller": "did:example:123456789abcdefghi",
    "publicKeyMultibase": "z6MkmM42vxfqZQsv4..."
  }]
}
```

The `authentication` field is the important part. It says: *whoever holds the private key matching this public key controls this DID.* That's your proof of ownership — not a password, not a company's database. A cryptographic keypair you hold.

## Verifiable Credentials — a digital passport

A VC is simply a verified claim about you — issued, signed, and portable.

Think of a passport. The government (issuer) says "this person is a citizen of this country" and stamps it with their authority. You (holder) carry it. The border officer (verifier) checks it.

VCs work the same way, just digital:

- **Issuer** — any entity that makes a claim about you. In Hiero's case, Heka issues a VC saying this GitHub account owns this GPG key.
- **Holder** — you. The VC lives in your wallet.
- **Verifier** — whoever needs to check the claim. In Hiero, the GitHub App verifies your VC when you open a PR.

The key difference from a real passport: nobody needs to call the government to verify it. The issuer's signature is enough. Fully offline, fully cryptographic.

**Verifiable Presentations** are what you send to a verifier. You take your VC, wrap it, and sign it yourself. This proves two things at once — the issuer made this claim, and you are the one presenting it right now.

## DID/VC vs existing GitHub/GPG verification

The existing GPG approach signs your commits with a private key. GitHub shows a "Verified" badge if the signing key matches the key you've uploaded to your account. Simple, and it works — until it doesn't.

**Where GPG falls short:**

- Key distribution is manual and fragile. You upload your public key to GitHub with no standard way to discover or verify it elsewhere.
- Identity is still GitHub-anchored. You're trusting GitHub to store your key and verify against it. GitHub is still the trust root.
- AI agents can generate a GPG key, upload it to a fake GitHub account, and pass all verification checks. The badge means "signed by a key on this account" — not "signed by a real human."

**What DID/VC adds:**

- The trust root moves off GitHub. Heka issues a VC binding your GitHub account to a DID anchored on Hedera — so verification doesn't depend on GitHub's database.
- Revocation is built into the model. A compromised credential can be revoked at the issuer level without touching GitHub at all.
- The same DID works across platforms — not just GitHub. Your identity is portable.

**The honest caveat:** this raises the bar, it doesn't eliminate the problem. A sophisticated attacker who compromises Heka or the issuance flow breaks everything. The trust root shifts — it doesn't disappear.

## A real example — Hiero contributor verification

This isn't theoretical. The [Hiero open source ecosystem](https://github.com/hiero-ledger) is building exactly this for GitHub contributor identity, using the [Heka Identity Platform](https://github.com/hiero-ledger/heka-identity-platform).

**Onboarding flow:**

1. You log in with GitHub
2. Heka creates a wallet and a DID for you
3. You link your GPG key to that DID
4. Heka issues a VC: this GitHub account owns this GPG key
5. Done — your identity is now verifiable without Heka being involved in every check

**Verification flow:**

1. You open a pull request
2. A GitHub App webhook fires
3. The App resolves your DID and retrieves your DID document
4. It checks your Verifiable Presentation
5. Reports pass or fail as a GitHub status check

I've been exploring this codebase as part of applying for the LFX mentorship program and built a stub GitHub App that demonstrates the verification flow: [hiero-contributor-verify](https://github.com/Jashk120/hiero-contributor-verify). Currently uses stub data but testnet integration is coming.

## The parts I simplified

- **Key loss** — if you lose the private key, the DID is gone. The problem shifts from platform custody to key management. Recovery requires building rotation logic on top.
- **Revocation** — "fully offline" was misleading. Revocation checks often require online registries or status lists. Freshness is non-trivial.
- **"No middleman" is overstated** — `did:hedera` still depends on Hedera's network and governance. You're shifting infrastructure trust, not eliminating it. Heka as a central issuer means the system layers cryptographic verification on top of existing centralized identity — useful, but different from the pitch.

## Further reading

- [W3C DID Spec](https://www.w3.org/TR/did-1.1/)
- [W3C VC Data Model](https://www.w3.org/TR/vc-data-model-2.0/)
- [Hedera DID Method](https://github.com/hashgraph/did-method/blob/master/hedera-did-method-specification.md)
- [Heka Identity Platform](https://github.com/hiero-ledger/heka-identity-platform)
