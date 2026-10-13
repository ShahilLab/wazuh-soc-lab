# Incident report 001: SSH brute-force alert (simulated)

**Status:** closed, true positive for the rule, benign (my own test)
**Environment:** private home lab, Wazuh 4.9.0

## 1. What happened

Wazuh raised rule `100021` ("Possible brute-force: 5+ failed logins in 60 seconds"), level 10, on agent `Kali_agent`.

- **Alert time:** 2026-10-13 06:54:16 UTC (12:24:16 local, UTC+5:30)
- **MITRE ATT&CK:** T1110.001, Credential Access, Password Guessing
- **Log source:** the Kali VM's journal (`sshd`)

## 2. Evidence

Five `Failed password` events for the same account, all from the same source IP (masked):

| # | Time (UTC) | sshd session | Source port |
|---|---|---|---|
| 1 | 06:53:54 | 15689 | 64900 |
| 2 | 06:53:58 | 15689 | 64900 |
| 3 | 06:54:01 | 15689 | 64900 |
| 4 | 06:54:10 | 15848 | 64927 |
| 5 | 06:54:15 | 15848 | 64927 |

- **5 failures in 21 seconds**, inside the rule's 60-second window.
- **Two SSH connections** (two sessions, two source ports), same source IP.
- The rule fired once, about a second after the fifth failure.

Files: `evidence/05-real-alert.png`, `evidence/06-alert-details.png`, `evidence/07-alert-100021.json`.

## 3. How the detection works

1. Each failed SSH password raises built-in rule `5760`.
2. Rule `100021` counts `5760` events: 5 within 60 seconds from the same source IP (`<same_source_ip />`).
3. When the count is reached, it raises the level-10 alert.

The dry-run tests (`evidence/04a` and `04b`) show it fires for one source and stays quiet for five different sources.

## 4. Was it expected?

Yes. I made these attempts myself, from my own Windows PC to my own Kali VM, to test the rule. In a real environment this would be an unexpected source, and I would treat it as a real attempt.

## 5. Impact and what I would do in a real case

- **Impact in this test:** none.
- **Check for success:** look for a successful SSH login from the same source after the failures (a success after many failures is the worrying pattern). In this test: no successful login after the failures.
- **Contain:** block the source IP, and lock or reset the targeted account if a login succeeded.
- **Harden:** key-only SSH, no password logins, rate limiting such as `fail2ban`.

Not performed here, because this was a controlled test.

## 6. Notes on the detection

- The rule is deliberately simple. A real attacker could slow the guessing to stay under 5 failures per minute, so a longer-window rule would be a sensible addition.
- The first version of this rule never fired because it waited for rule `5716`, while real SSH password failures are rule `5760`. I found this by comparing the rule with real logs.
