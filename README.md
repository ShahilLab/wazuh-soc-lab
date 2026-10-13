# Wazuh SOC Home Lab

A home SIEM lab I built to practise SOC analyst skills: collecting logs from two machines, writing a custom detection rule, testing it, and investigating the alert.

**Author:** Mohomed Shahil · **LinkedIn:** www.linkedin.com/in/mohomed-shahil-9643802b1

> **Scope:** everything here ran on machines I own, on a private Host-only network. The "attack" is me typing wrong SSH passwords from my own PC into my own VM. IP addresses, usernames and hostnames are masked in the screenshots.

## What I built

- **Wazuh 4.9.0** (manager, indexer, dashboard) running with Docker Compose on a Kali Linux VM in VirtualBox
- **Two agents** reporting to it: the Kali VM and my Windows 11 PC
- **A custom detection rule** for SSH brute-force attempts (rule `100021`), mapped to MITRE ATT&CK T1110.001

## How it fits together

```
Windows 11 agent ─┐
                  ├─► Manager ──► Indexer ──► Dashboard
Kali Linux agent ─┘   (rules)     (storage)   (alerts)
```

- **Agents** read logs on each machine and send them to the manager.
- **Manager** checks every event against its rules and raises alerts.
- **Indexer** stores and searches the alerts.
- **Dashboard** is where I read them.
- **Docker Compose** starts all three server parts, already connected, from one file.

## Setup summary

1. Cloned the official `wazuh-docker` repo (tag `v4.9.0`, `single-node`).
2. Installed Docker and Compose on Kali.
3. Raised `vm.max_map_count` to 262144, which the indexer needs to start.
4. Generated the TLS certificates (`generate-indexer-certs.yml`), then started the stack with `docker compose up -d`.
5. Enrolled both agents from the dashboard (Endpoints Summary → Deploy new agent).
6. Added a second, Host-only network adapter to the VM so the Windows agent could reach the manager on ports 1514 and 1515.

## The custom rule

File: [`rules/local_rules.xml`](rules/local_rules.xml). It holds three rules:

| Rule | What it is |
|---|---|
| `100001` | Wazuh's default example rule. Not mine. |
| `100020` | New-device (USB) detection for Windows. Included but **not tested**; it needs the Windows audit policy for Plug and Play events. |
| `100021` | **My SSH brute-force rule. Tested.** |

```xml
<rule id="100021" level="10" frequency="5" timeframe="60">
  <if_matched_sid>5760</if_matched_sid>
  <same_source_ip />
  <description>Possible brute-force: 5+ failed logins in 60 seconds</description>
  <mitre>
    <id>T1110.001</id>
  </mitre>
</rule>
```

In plain words: if built-in rule `5760` (sshd authentication failed) fires **5 times within 60 seconds from the same source IP**, raise a level-10 alert tagged MITRE T1110.001 (Brute Force: Password Guessing).

## Testing

**1. Dry run with `wazuh-logtest`.** This feeds the manager made-up log lines and shows which rule would match. It does not touch the dashboard. The IPs below (`203.0.113.x`) are reserved documentation addresses.

| Test | Input | Expected | Result |
|---|---|---|---|
| A | 5 failures from **one** IP | rule 100021 fires | [`04a-logtest-fires.png`](evidence/04a-logtest-fires.png) |
| B | 5 failures from **five different** IPs | only 5760, no 100021 | [`04b-logtest-no-fire.png`](evidence/04b-logtest-no-fire.png) |

Test B shows the rule counts one source knocking five times, not five sources knocking once. That is what `<same_source_ip />` is for. (The "full event" line is shown twice on the last line of each test. That comes from how I pasted the lines and does not change the decoding.)

**2. Real test.** I made repeated wrong SSH passwords from my Windows PC to the Kali VM. Rule 100021 fired once at level 10:

- [`05-real-alert.png`](evidence/05-real-alert.png): the alert in Threat Hunting
- [`06-alert-details.png`](evidence/06-alert-details.png): the five failures behind it
- [`07-alert-json-terminal.png`](evidence/07-alert-json-terminal.png) and [`07-alert-100021.json`](evidence/07-alert-100021.json): the stored alert record

Full write-up: [`docs/incident-report-001.md`](docs/incident-report-001.md).

**3. Windows.** Failed Windows logons were picked up by Wazuh's built-in rules (no custom rule involved): [`08-windows-logon-failures.png`](evidence/08-windows-logon-failures.png).

## Evidence index

See [`evidence/README.md`](evidence/README.md) for what each file proves and how it was produced.

> Dashboard times are local (UTC+5:30). Raw log and JSON times are UTC.

## Problems I hit and how I fixed them

| Problem | Cause | Fix |
|---|---|---|
| `docker: command not found`, then a package-not-found error | Docker was not installed, and I used an Ubuntu package name on Kali | Installed Docker from the Kali repositories |
| Windows agent showed "started" but never connected | Its `ossec.conf` had tags that agent 4.9.0 does not know (`No such tag 'users'`, `'groups'` in `ossec.log`) | Clean reinstall to get a fresh config |
| Windows could not reach the manager | VirtualBox NAT lets the VM reach out but not be reached | Added a Host-only adapter |
| Windows agent became Disconnected after a restart | The Host-only adapter on Kali had lost its IP address, so port 1514 was unreachable | Restored the adapter's address |
| My custom rule never fired | It waited for rule `5716`, but real SSH password failures are rule `5760` | Changed the parent rule and re-tested with real data |
| Rule counted failures from anyone | Missing `<same_source_ip />` | Added it and confirmed with Test B |
| Dashboard search showed "No results" | Time window was only the last 15 minutes, and Windows alerts were mixed in | Widened the time range and filtered by agent |

## What I learned

- A service saying "started" does not mean it works. The agent log told me the real problem.
- Test a detection rule with real data, not only with the sample line.
- Test both directions: it must fire when it should, and stay quiet when it should not.
- When an agent is disconnected, work from the symptom to the cause: log, then ping, then the machine's network card.
- Breaking things taught me more than succeeding

## Limitations and next steps

- Rule `100020` (USB) is untested.
- The lab uses Wazuh's default demo credentials and is meant for a private network only. Change them and do not expose the ports before using anything like this elsewhere.
- Next: a custom rule for Windows failed logons (event ID 4625), email or chat notifications, and more MITRE techniques.
