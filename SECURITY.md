# Security policy

## Reporting a vulnerability

Please report vulnerabilities privately through GitHub's
[private vulnerability reporting](../../security/advisories/new) for this repository
rather than opening a public issue. We will acknowledge the report and keep you updated
while we work on a fix.

## Known upstream issues

The example agents run on `a2a-sdk`, and two of its open issues shape how they are
deployed. See [Deployment](README.md#deployment):

- [a2a-python #1159](https://github.com/a2aproject/a2a-python/issues/1159): live-task
  owner-isolation bypass. Never expose an agent publicly.
- [a2a-python #1135](https://github.com/a2aproject/a2a-python/issues/1135): run a single
  replica.

## Keeping secrets out of this repository

Never commit credentials, account identifiers, ARNs or internal hostnames. `gitleaks` scans
every push and the full history, and a credential that lands in history cannot be removed
by a later commit.
