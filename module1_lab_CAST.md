# The cast: Acme Industrial

Every name in the labs is a person with a job. Learners should be able to ask "who wrote this row and why" of any
record in the environment. Names are fictional; any resemblance to real Acme staff is unintentional.

## Security operations (they write the tickets)

| Name | Username | Role | Where they appear |
| --- | --- | --- | --- |
| Priya Nair | pnair | SOC lead | Sends the Lab 1 brief. Closes every incident (`closed_by` in tickets.csv). |
| Callum Reid | creid | Tier 2 analyst | Raises incidents (`raised_by`). Runs containment; his commands land in 4688 logs. |
| Ifeoma Eze | ieze | Tier 2 analyst | Raises incidents, runs containment. |
| Marcus Lindberg | mlindberg | Tier 1 analyst | Raises incidents. |
| Sunita Rao | srao | Detection engineer | Consumes the Lab 3 artefact. Will write the Sigma rules in Module 8. |

## Everyone else in the scenario text

| Name | Role | Where they appear |
| --- | --- | --- |
| Anita Krishnan | CISO | The deck in Lab 2 is going to her. |
| Daniel Okafor | Data scientist | Built the leaky baseline the learner audits in Lab 2. |
| Meera Iyer | Platform engineering lead | Her team runs the Lab 3 artefact nightly and will page you when it breaks. |
| Sam Whitfield | IT operations | Owns the server estate and the `svc_backup` service account the red team abuses. |

## Staff in the telemetry

25 named employees across Finance, Engineering, Plant Operations, Sales and IT Operations own the 40 Windows hosts.
`data/w1/asset_inventory.csv` is the CMDB export: hostname, IP, owner name and username, department, VLAN, OS build,
criticality. Their usernames appear as `SubjectUserName` and `TargetUserName` in the 4688, 4624 and 4625 events.

Two things about the inventory matter for the labs:

1. It replaces the hostname convention as the honest way to map an IP to a host. `common.host_for_ip` still exists for
   the graders, but learners should join on the CMDB, because that is what they will have in production.
2. The red-team workstation is not in it. Joining inventory to flows leaves `department = "unmanaged"` on exactly the
   attacker's rows, which is a proxy leak with a human explanation: the CMDB does not know about a machine IT never
   provisioned. It is in the Lab 2 pool for that reason, and the probe expects it dropped.

## Red team engagements

| Week | Engagement | Operator workstation | Service account abused | Tooling signature |
| --- | --- | --- | --- | --- |
| Week 1 (public) | Whitaker | 10.40.7.99, TTL 64, VLAN 40 | svc_backup | SMB and RDP lateral movement, ports 445 and 3389 |
| Week 2 (private) | Halloran | 10.12.3.44, TTL 128, VLAN 12 | svc_deploy | WinRM and SSH lateral movement, ports 5985 and 22 |

Two different contractors, two different playbooks. That is why a week-1 model that learned TTL 64, sensor-B, port 3389
or `svc_backup` scores near zero on week 2, and it is the whole point of the gate.
