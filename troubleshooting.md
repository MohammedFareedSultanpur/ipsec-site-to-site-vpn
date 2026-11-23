# Troubleshooting Log

Faults are injected one at a time on BR-R2, diagnosed with `show` and `debug` commands, fixed, and verified. Every entry uses the same template: change made, symptom, investigation, root cause, fix, verification, lesson.

| # | Fault | Layer affected | Key command | Status |
|---|---|---|---|---|
| 1 | [Wrong pre-shared key](#fault-1-wrong-pre-shared-key) | Phase 1 | `debug crypto isakmp` | Done |
| 2 | Phase 1 parameter mismatch | Phase 1 | `debug crypto isakmp` | Planned |
| 3 | Transform-set mismatch | Phase 2 | `debug crypto ipsec` | Planned |
| 4 | Crypto ACL not mirrored | Phase 2 | `show access-lists` | Planned |
| 5 | Missing NAT exemption | NAT | `show ip nat translations` | Planned |
| 6 | Wrong peer address | Phase 1 | `show crypto isakmp sa` | Planned |

**Lab:** CML Free, IOL routers (IOS XE 17.18). HQ-R1 = 203.0.113.1, BR-R2 = 198.51.100.2. Pre-shared keys are shown as `<PSK>` in this document.

---

## Fault 1: Wrong Pre-Shared Key

### Change made
On BR-R2, the pre-shared key for peer 203.0.113.1 was replaced with a different value. HQ-R1 was left unchanged.

```
BR-R2(config)# no crypto isakmp key <PSK> address 203.0.113.1
BR-R2(config)# crypto isakmp key WrongKey-999 address 203.0.113.1
BR-R2# show running-config | include isakmp key
crypto isakmp key WrongKey-999 address 203.0.113.1
```

The old SAs were cleared with `clear crypto isakmp` and `clear crypto sa`, so the next negotiation used the new key.

### Symptom
The tunnel could not be established. Phase 1 never reached `QM_IDLE`. BR-R2 logged `IKMP_BAD_MESSAGE`, retried the exchange five times and deleted the SA as unauthenticated (about 41 seconds after the first packet).

### Investigation

**1. Is the peer reachable?** Yes. `debug crypto isakmp` on BR-R2 showed IKE packets arriving from HQ-R1 and BR-R2 replying, so routing and the ISP path were working. (HQ-R1 started the negotiation, so BR-R2 acted as the responder.)

**2. Does the Phase 1 policy match?** Yes. BR-R2 compared HQ-R1's proposal against its own policy 10 and accepted it:

```
ISAKMP: (0):found peer pre-shared key matching 203.0.113.1
ISAKMP: (0):Checking ISAKMP transform 1 against priority 10 policy
ISAKMP: (0):      encryption AES-CBC
ISAKMP: (0):      keylength of 256
ISAKMP: (0):      hash SHA256
ISAKMP: (0):      default group 14
ISAKMP: (0):      auth pre-share
ISAKMP: (0):atts are acceptable. Next payload is 0
```

This rules out a Phase 1 parameter mismatch. Note that `found peer pre-shared key matching` only means a key is configured for that peer address. It does not mean the key is correct.

**3. Does the key exchange work?** Yes. Messages 3 and 4 (Diffie-Hellman and nonces) completed:

```
ISAKMP: (0):processing KE payload. message ID = 0
ISAKMP: (0):processing NONCE payload. message ID = 0
ISAKMP-PAK: (1005):sending packet to 203.0.113.1 my_port 500 peer_port 500 (R) MM_KEY_EXCH
```

**4. Where does it fail?** On the first encrypted message (message 5, which carries the peer's identity):

```
ISAKMP-PAK: (1005):received packet from 203.0.113.1 dport 500 sport 500 Global (R) MM_KEY_EXCH
ISAKMP-PAK-ERROR: (1005):reserved not zero on ID payload!
%CRYPTO-4-IKMP_BAD_MESSAGE: IKE message from 203.0.113.1 failed its sanity check or is malformed
```

**5. What happens next?** BR-R2 retransmitted five times, then gave up:

```
ISAKMP: (1005):: incrementing error counter on sa, attempt 1 of 5: reset_retransmission
...
ISAKMP: (1005):: incrementing error counter on sa, attempt 5 of 5: retransmit phase 1
ISAKMP-ERROR: (1005):deleting SA reason "Death by retransmission P1" state (R) MM_KEY_EXCH (peer 203.0.113.1)
ISAKMP: (1005):Deleting the unauthenticated sa
```

(Vendor ID lines and repeated retransmission lines were removed from the excerpts above for readability.)

### Timeline from the debug output

| Time | Event |
|---|---|
| 03:49:55.038 | HQ-R1 sends Main Mode message 1; BR-R2 creates a new SA |
| 03:49:55.039 | Proposal matches policy 10 (`atts are acceptable`) |
| 03:49:55.040 to .044 | Key exchange completes (messages 3 and 4) |
| 03:49:55.049 | Message 5 fails: `reserved not zero on ID payload!` |
| 03:49:56 to 03:50:36 | Five retransmissions, no progress |
| 03:50:36.554 | `Death by retransmission P1`, unauthenticated SA deleted |
| 03:51:36.555 | SA purged from memory |

### Root cause
The two routers held different pre-shared keys. Each router mixes the pre-shared key into the keys that encrypt Main Mode messages 5 and 6. With different keys, each side derived different encryption keys, so BR-R2 could not decrypt HQ-R1's message 5. The result looked like garbage (`reserved not zero`, `malformed`) instead of a clear "wrong password" message, which is the standard fingerprint of a pre-shared key mismatch in Main Mode.

**How to tell this apart from a Phase 1 parameter mismatch:**

| | Wrong pre-shared key | Phase 1 parameter mismatch |
|---|---|---|
| Proposal | Accepted (`atts are acceptable`) | Rejected (no acceptable proposal) |
| Fails at | First encrypted message (message 5) | Message 1 or 2 |
| Typical errors | `reserved not zero`, `IKMP_BAD_MESSAGE` | Proposal rejected in the debug |

### Fix
Restored the matching key on BR-R2, then cleared the SAs on both routers:

```
BR-R2# undebug all
BR-R2# configure terminal
BR-R2(config)# no crypto isakmp key WrongKey-999 address 203.0.113.1
BR-R2(config)# crypto isakmp key <PSK> address 203.0.113.1
BR-R2(config)# end

HQ-R1# clear crypto isakmp
HQ-R1# clear crypto sa
BR-R2# clear crypto isakmp
BR-R2# clear crypto sa
```

### Verification
After the fix:

- `ping -c 5 192.168.20.10` from PC1 succeeded (the first packet can be lost while IKE negotiates).
- `show crypto isakmp sa` showed `QM_IDLE` / `ACTIVE`.
- `show crypto ipsec sa` showed `#pkts encaps` and `#pkts decaps` increasing.
- The configuration was saved with `write memory`.

### Lessons
1. Check the easy layers first (interfaces, routing, peer reachability, policy), then use debug to confirm.
2. In `debug crypto isakmp`, "proposal accepted, then failure at the first encrypted message" points at authentication, which for a pre-shared key setup means the key.
3. The router cannot tell a wrong key from a corrupted packet, so the error message describes the symptom (malformed message), not the cause.
4. Always run `undebug all` afterwards, and clear the SAs on both routers after any key change, or the old SA can hide the fault.
