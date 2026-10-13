# Evidence index

IP addresses, usernames and hostnames are masked in the screenshots (black or grey bars). The `203.0.113.x` addresses in the dry-run tests are reserved documentation addresses, not real hosts. The terminal windows were see-through, so I removed the desktop background that showed through them.

| File | What it shows | How it was produced |
|---|---|---|
| `01-containers-running.png` | The Wazuh manager, indexer and dashboard containers are up | `docker compose ps` |
| `02-agents-active.png` | Both agents (Kali, Windows) are active | Dashboard → Endpoints Summary |
| `03-custom-rule.png` | The rules file with rule 100021, as edited in nano | `nano local_rules.xml` |
| `04a-logtest-fires.png` | Dry run: 5 failures from one IP trigger rule 100021 | `wazuh-logtest` |
| `04b-logtest-no-fire.png` | Dry run: 5 failures from five IPs do **not** trigger 100021 | `wazuh-logtest` |
| `05-real-alert.png` | Rule 100021 fired on a real test, level 10, MITRE T1110.001 | Threat Hunting, agent Kali_agent |
| `06-alert-details.png` | The alert's fields: rule, level, frequency, and the five failures | Expanded alert |
| `07-alert-json-terminal.png` | The stored alert record in the terminal | `head -n 60` on the JSON file |
| `07-alert-100021.json` | The same record as a file. IPs replaced with `x.x.x.x`, usernames and hostnames replaced with `lab-user`, and re-indented | `alerts.json`, then `sed` and `python3 -m json.tool` |
| `08-windows-logon-failures.png` | Built-in Windows rules (60122 and others) detecting failed logons | Threat Hunting, agent Windows_Host |

Notes:

- Dashboard times are local (UTC+5:30). Raw log and JSON times are UTC.
- `07-alert-100021.json` has the same content as the stored alert but is **not a byte-for-byte copy** because of the masking and re-indenting.
- `SHA256SUMS.txt` lists a fingerprint for each file. It shows the files have not changed since I made the list. It does not prove where they came from.
  Check with: `sha256sum -c SHA256SUMS.txt`
