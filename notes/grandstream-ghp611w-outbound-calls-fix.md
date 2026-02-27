# Grandstream GHP611W — Outbound Calls Failing (FreePBX/Asterisk)

## Symptom
- Phone registers successfully with FreePBX
- Phone can receive calls (audio works both ways)
- Phone cannot make outbound calls — dials, then nothing happens until timeout

## Environment
- Grandstream GHP611W (firmware 1.0.1.93)
- FreePBX 17 with PJSIP
- Phone behind NAT, server in cloud

## Root Cause
**UDP packet fragmentation causing Asterisk to silently drop authenticated INVITE requests.**

When the phone dials out:
1. Phone sends INVITE (no credentials)
2. FreePBX responds with 401 Unauthorized challenge
3. Phone sends ACK, then a new INVITE with Authorization header
4. The authenticated INVITE packet exceeds 1476 bytes due to:
   - Large SDP body with many codecs listed
   - Additional SIP headers (P-Preferred-Identity, Privacy, Proxy-Require)
5. Packet gets fragmented (IP flag `[+]` = more fragments)
6. Asterisk/PJSIP does not reassemble fragmented UDP packets
7. Authenticated INVITE is silently dropped — no response sent
8. Phone retries repeatedly, eventually times out

## Diagnosis Steps
1. `tcpdump` showed INVITE packets arriving at server
2. `sngrep` showed only 3 messages in INVITE dialog (INVITE → 401 → ACK), missing the authenticated retry
3. Deeper `tcpdump -nnvv` revealed packets with `flags [+]` and `length 1476` — indicating fragmentation

## Fix
**Reduce packet size by disabling unnecessary codecs:**

1. Go to **Account 1 → Codec Settings**
2. Disable all codecs except:
   - PCMU (G.711 μ-law)
   - PCMA (G.711 A-law)
   - Optionally G722 for HD audio
3. Disable: Opus, G723, G729, iLBC, G726, and any others
4. Save and Apply

**Also ensure these settings are cleared/disabled:**
- **Proxy-Require**: Must be blank (was auto-filled with domain name)
- **Use P-Preferred-Identity Header**: No/Disabled
- **Use Privacy Header**: No/Disabled

## Verification
After fix, authenticated INVITE packets should be under 1400 bytes with no fragmentation flag, and calls will complete normally.

## Related Issues
- Grandstream forum thread: Similar issue reported with GXP2170 and HT801
- Affects any SIP phone sending large INVITE packets over UDP to Asterisk/PJSIP

## Key Lesson
When outbound calls fail silently after 401 challenge, and `tcpdump` shows packets arriving but Asterisk doesn't respond, check for **UDP fragmentation** — the packet may be too large for Asterisk's SIP parser to handle.
