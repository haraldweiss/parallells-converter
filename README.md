Started this but did not finish as it was easier to get the VMs recreated in UTM than to convert the parallels VMS

### Coding-agent credential safety (2026-11-09)

Local Claude permission rules and saved agent transcripts may contain literal
credentials. Keep them outside Git; shared `.claude/settings.json` must use
variable references rather than secrets. `Secret safety` CI rejects tracked local
agent state and scans introduced commits with Gitleaks, including documentation,
example files, and password-bearing commands. Existing history was audited
separately; this CI job is not a claim that historical credentials are safe.
If a credential was committed, revoke or rotate it before considering the leak
resolved. Do not print secret values in scan logs, reviews, or handoffs.
