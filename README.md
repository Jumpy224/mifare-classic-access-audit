# MIFARE Classic Access Card Audit

This is a write-up of two security assessments I carried out on my access card issued by an educational institution. The first assessment was done in May 2026 using a Flipper Zero. The second was a deeper re-audit in June 2026 using a Proxmark3, carried out after obtaining written authorisation.

The card is a MIFARE Classic 1K running on a Fudan FM11RF08S chip. Every sector uses the factory default key. The chip itself has a known static nonce vulnerability published by Quarkslab in 2024. And after dumping the card in the second assessment, I found the access control system does not use the encrypted sectors at all. It authenticates on UID only.

Card details and the organisation name are redacted. The finding is not yet remediated, so identifying the site would be irresponsible.

---

## Overview

**Author:** Logan  
**First assessment:** May 2026, Flipper Zero, read only  
**Re-audit:** June 2026, Proxmark3 (Iceman firmware), written authorisation obtained  
**Status:** Disclosed to the organisation. Acknowledged. No remediation taken at time of writing.

---

## Summary

The card is a MIFARE Classic 1K on a Fudan FM11RF08S chipset. All 16 sectors use the factory default key FFFFFFFFFFFF. The FM11RF08S also has a static nonce vulnerability, meaning full key recovery takes approximately 2 seconds on a Proxmark3 regardless of what keys are set.

The more significant finding is that all data blocks in the dump are empty. The system authenticates on UID alone and never reads the encrypted sectors. The MIFARE Classic cryptographic layer is completely unused. The security is equivalent to a 125 kHz EM4100 card from 1991.

A working clone was produced in under 2 minutes using a Proxmark3 and a blank magic card. The clone is byte-for-byte identical to the original at the protocol level.

---

## Background

MIFARE Classic uses a proprietary stream cipher called Crypto-1. Crypto-1 has been publicly broken since 2008. The standard attacks, darkside, nested, and hardnested, recover sector keys in seconds to minutes on a Proxmark3.

The Fudan FM11RF08S chip makes things worse. It generates the same nonce on every authentication attempt instead of a random one. That is the Quarkslab 2024 finding. The chip also has a hardcoded manufacturer backdoor key that allows any sector to be read regardless of the configured keys.

There are three separate problems here:

1. The system authenticates on UID only. The encrypted sectors are never read. The UID is freely readable by any NFC device and can be written to a blank magic card.
2. All 16 sectors use the factory default key. No attack is needed. Any tool with the default key dictionary reads the card immediately.
3. The FM11RF08S chipset has a static nonce and a manufacturer backdoor. Even with unique strong keys, recovery takes seconds.

Changing the keys only addresses problem 2. Problems 1 and 3 require replacing the card stock and all readers.

---

## Assessment 1, May 2026

**Tool:** Flipper Zero  
**Scope:** Read only, my own card, no cloning, no door testing

The Flipper's built-in key dictionary authenticated to every sector on the first attempt using FFFFFFFFFFFF. The full 1K dump completed in a few seconds. Every sector trailer was identical:

```
Key A:        FF FF FF FF FF FF  (factory default)
Access bits:  07 80 69           (factory default)
Key B:        FF FF FF FF FF FF  (factory default)
```

Data blocks were mostly empty, which suggested UID-only authentication, though that was not confirmed at this stage.

I stopped at the read. Cloning and testing at a door without written authorisation would be an offence under the Computer Misuse Act 1990. The read alone was sufficient to demonstrate the vulnerability.

The finding was reported to the organisation. They acknowledged it and took no action.

---

## Assessment 2, June 2026

**Tool:** Proxmark3 (Iceman firmware)  
**Scope:** Key recovery, full dump, proof-of-concept clone of my own card. Written authorisation obtained before starting. Testing against organisation readers and touching any third-party card were out of scope.

### Commands run

```
hf 14a info          # card identification
hf mf info           # key state, PRNG type, magic detection
hf mf autopwn        # automated key recovery and full dump
hf mf view           # inspect dump contents
hf mf cload          # write dump to magic card
hf 14a info          # verify clone UID matches original
```

### Finding 1: FM11RF08S static nonce

`hf 14a info` flagged this immediately:

```
[#] Static nonce: [REDACTED]
[+] Static nonce: yes
```

This confirms the FM11RF08S chipset. The static nonce means the card produces the same value on every authentication rather than generating a random one. Combined with the default keys, `hf mf autopwn` completed full key recovery and a card dump in 2 seconds.

### Finding 2: Default keys on all 16 sectors

Every sector trailer in the dump was identical:

```
FF FF FF FF FF FF  FF 07 80 69  FF FF FF FF FF FF
```

Key A and Key B are both FFFFFFFFFFFF across every sector. The cards were issued without any re-keying. This confirms the finding from Assessment 1 with a more capable tool.

### Finding 3: UID-only authentication

Every data block in the dump was zeroed:

```
Sector 0, Block 1:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
Sector 0, Block 2:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
[all 62 data blocks identical]
```

Nothing has ever been written to the card beyond the manufacturer block. The access control system is making entry decisions based on the UID alone. The entire encrypted layer of MIFARE Classic is switched off.

This is not a misconfiguration that can be corrected with a key change. The readers were installed to check UIDs only. Fixing it means replacing every reader in the building.

### Finding 4: Working clone produced in under 2 minutes

```
hf mf autopwn       # 2 seconds, full key recovery and dump
hf mf cload         # dump written to Gen1A magic card
hf 14a info         # UID, ATQA, SAK all match original
```

The clone presents identically to the original at the protocol level. A reader doing UID-only authentication has no way to distinguish between the two.

Time from first read to working clone: under 2 minutes.
Equipment cost: consumer hardware and a blank card.

### Finding 5: Clone carrying an arbitrary NFC payload

As an additional test, a URL was written into the empty data sectors of the clone using NDEF. The card then presented the credential UID to access readers while simultaneously serving the URL to any NFC-capable phone.

In a real attack, that URL could be a phishing page or a malware delivery link. There is no visual or electronic way to distinguish a legitimate card from a clone carrying a malicious payload.

---

## Impact

The following consequences follow from the confirmed findings. Items 1 to 3 were demonstrated within the authorised scope. Items 4 and 5 are inferred from the confirmed findings.

1. Any authorised card can be cloned in under 2 minutes using consumer equipment.
2. The cardholder has no indication their credential was copied. Reading a card is completely passive and silent.
3. A card in a pocket or bag is close enough to capture without physical contact.
4. If every card on site was provisioned the same way, which the uniform default keys strongly suggest, then the entire card estate is exposed.
5. Cloned credentials can carry arbitrary NFC payloads, enabling phishing or malware delivery alongside physical access.

---

## Remediation notes

The findings here sit at different levels. The default keys are a configuration issue and could be addressed without replacing hardware. The UID-only authentication is an architectural decision baked into how the readers were installed and configured. The FM11RF08S static nonce is a chipset-level flaw that exists regardless of how the system is configured.

Re-keying the cards would address the default key finding only. It would not change the fact that the readers are not using the encrypted sectors, and it would not remove the FM11RF08S chipset vulnerability. A motivated attacker with a Proxmark3 could still recover non-default keys against this chip in a few seconds.

The only way to address all three findings together is to replace the card stock and reader infrastructure with a platform that uses mutual authentication, such as MIFARE DESFire EV2/EV3 or HID iCLASS SE. That is a procurement decision, not something that can be patched.

Short of a full replacement, the most impactful single change would be configuring the readers to authenticate on encrypted sector data rather than the UID alone. That would at least mean the cryptographic layer of the cards is being used, even if the underlying Crypto-1 cipher remains weak.

---

## Disclosure timeline

| Date | Event |
|---|---|
| May 2026 | Default key vulnerability found and reported to organisation |
| May 2026 | Organisation acknowledged the report, no action taken |
| June 2026 | Written authorisation obtained for deeper re-audit |
| June 2026 | Re-audit completed, static nonce, UID-only auth, and working clone confirmed |
| June 2026 | Updated findings reported to organisation |
| June 2026 | This write-up published |

---

## References

- Garcia et al., Dismantling MIFARE Classic (2008)
- Quarkslab, FM11RF08S static nonce vulnerability (2024)
- NXP application notes on migrating from MIFARE Classic to DESFire
- Iceman/RfidResearchGroup Proxmark3 firmware documentation
- Proxmark3 and Flipper Zero MIFARE Classic documentation

---

## Licence

CC BY 4.0. Reuse with attribution.
