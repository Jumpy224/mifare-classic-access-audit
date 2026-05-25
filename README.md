# Mifare Classic Access Card Audit

A short security write-up. I found that an access card issued to me by a UK further-education college was a Mifare Classic 1K running factory-default keys, disclosed it to the college, and deliberately stopped at reading the card rather than cloning it and testing entry, because doing that without written authorisation would be an offence under the Computer Misuse Act 1990.

This repo documents the finding, the evidence (redacted), the impact, and the fix. It is here as a sample of how I scope, document, and disclose security work.

- Card details (UID, manufacturer block, data blocks) are redacted.
- The college is not named. The finding is unremediated, so naming the site would put it at risk for no benefit.
- The raw card dump and original photos are not included, for the same reason.

---

## Access control weakness in Mifare Classic 1K site cards

**Author:** Logan T.
**Date:** May 2026
**Status:** Disclosed to the issuing organisation; acknowledged, no remediation taken.

### Summary

A site access card issued by a UK further-education college turned out to be a Mifare Classic 1K card protected by nothing but the factory-default keys. Those keys were never changed from the manufacturer default (`FFFFFFFFFFFF`), so any commodity RFID tool can read the card's whole memory with no key-recovery step at all. Once it has been read, the contents can be cloned to a blank card or emulated, and the access system treats the result as a genuine credential.

Low effort, high impact. There is no cryptographic attack to mount. You need a sub-£200 consumer device and a few seconds next to a card.

### Background: why Mifare Classic is a poor choice for access control

Mifare Classic protects its data with a proprietary stream cipher called Crypto-1. Crypto-1 has been publicly broken since 2008. The documented attacks (`darkside`, `nested`, and the newer `hardnested` and static-nonce attacks) recover sector keys in seconds to minutes on consumer hardware such as a Proxmark3, and on a Flipper Zero for many cards.

So even a correctly re-keyed Mifare Classic card is not secure. The cipher itself is the weak point. The card I looked at never even reached that bar, because it still used the default keys.

Two problems are in play, worst first:

1. Default keys, the finding here. No attack is needed. Any reader loaded with the default key dictionary dumps the card straight away.
2. A broken cipher, baked into the platform. Even unique keys can be recovered, so re-keying a Mifare Classic card does not fix the underlying problem.

### Method

I ran the assessment on a card issued to me, read with my own Flipper Zero. I touched no third-party card and attempted no entry with any reproduced data.

1. Placed the card on the reader and ran a standard Mifare Classic read.
2. The tool's built-in key dictionary authenticated to every sector using the default Key A and Key B.
3. The full 1K of card memory dumped across all 16 sectors and 64 blocks.
4. Inspected the dump to confirm the key state and data layout.

#### Evidence (redacted)

The dump confirmed the card type and key state. Identifying values (the UID, manufacturer block, and any data blocks) are redacted here.

```
Device type:         Mifare Classic
Mifare Classic type: 1K
UID:                 [REDACTED]
ATQA:                00 04
SAK:                 08

Block 0:  [REDACTED: UID + manufacturer data]
Block 1:  [REDACTED]
...
Sector trailers (blocks 7, 11, 15, ...):
  FF FF FF FF FF FF   07 80 69   FF FF FF FF FF FF
  Key A = default     access     Key B = default
                      bits = default
```

The repeating sector-trailer pattern is the finding. `FF FF FF FF FF FF` for both Key A and Key B is the factory default, and `07 80 69` is the default access-condition byte set. Seeing this across every sector confirms the card was issued with no re-keying.

Most data blocks were empty (`00`), which suggests the system keys entry on the UID alone rather than on stored encrypted data. If so, that is a further weakness, because the UID is readable by any device and Mifare Classic UIDs can be emulated on writable ("magic") cards.

### Scope and limitations

I deliberately held this assessment to reading a card issued to me, using my own equipment. I did not clone the card, did not write its data to a blank or magic card, and made no attempt to use a reproduced credential for entry.

That boundary was a choice, not a technical wall. Cloning a credential and testing physical entry without prior written authorisation would be unauthorised access under the Computer Misuse Act 1990, whatever the intent behind it. The read on its own demonstrates the weakness. Exercising it would have been an offence and would have added nothing to the finding.

So the Impact section below describes what an attacker could do, reasoned from the confirmed read. It is not a record of anything I carried out.

### Impact

These are the realistic consequences of the confirmed weakness. None of them were carried out; they describe what the read makes possible for an attacker.

- Cloning: copy the card to a blank or magic card. The clone carries the same UID and data, so a UID-based or default-key-based access system cannot tell it apart from the original.
- Unauthorised entry: a cloned credential is accepted by readers, giving site access with no knowledge on the cardholder's part and no tamper indication.
- Scale: if every site card is provisioned the same way, with default keys and the same scheme, the weakness covers the whole estate rather than one card.
- Detectability: reading a card is passive and silent. A card sitting in a back pocket or a bag is close enough to capture in a moment of proximity.

### Remediation

Roughly in order of effectiveness:

1. Migrate to a modern credential. Replace Mifare Classic with Mifare DESFire EV2/EV3 (AES-128, mutual authentication) or Mifare Plus in Security Level 3 (AES-protected). Both defeat the dump-and-clone attack class outright.
2. Stop trusting the UID. Access decisions should rest on cryptographically authenticated card data, never on the freely readable, emulatable UID.
3. If Classic has to stay in the short term, re-key away from the defaults with unique diversified keys per card, while accepting that this only raises the bar a little, because Crypto-1 itself is breakable.
4. Add defence in depth for sensitive areas: a PIN at the reader for two-factor, or supervised access.

Only item 1 is a durable fix. Items 2 to 4 reduce the risk without removing it.

### Disclosure timeline

- Reported to the issuing organisation with the technical detail above.
- Response: acknowledged, with no remediation undertaken at the time of writing.

This write-up redacts the organisation's identity and every card-identifying value. It documents a vulnerability class on a card issued to me, read with my own equipment. No third party's credential was accessed, and no entry was attempted with reproduced data.

### References

- Garcia et al., *Dismantling MIFARE Classic* (2008), the original Crypto-1 cryptanalysis.
- NXP application notes on migrating from Mifare Classic to DESFire.
- Proxmark3 and Flipper Zero documentation on Mifare Classic key dictionaries.

---

## License

The text of this write-up is released under [CC BY 4.0](LICENSE). You may reuse it with attribution.
