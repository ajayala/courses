# When Things Go Wrong — Detecting and Responding to a Leak

In this lab you will learn what to do the moment you discover a secret has been exposed — whether you find it yourself, a teammate spots it, or a security scanner alerts. You will build a personal incident runbook, practise cleaning a secret from git history, and understand how to audit what damage was done. Speed and sequence are everything: the wrong first move can make a bad situation significantly worse.

**Prerequisites:** Lab 3-1 completed; `safety-journal/` and `safe-app/` exist. `git filter-repo` installed (`pip install git-filter-repo`).

---

## Step 1 — Detection: How You Find Out

Leaks are discovered through several channels, each with a different urgency level.

| Discovery channel | Typical urgency | Who tells you |
|------------------|----------------|---------------|
| GitHub Secret Scanning alert | High — the key is on the public internet | GitHub email / Security tab |
| GitGuardian / Trufflehog alert | High — automated scanner | Alert email or Slack |
| Colleague spots it in a code review | Medium — caught before push | PR comment or DM |
| You notice it yourself post-push | High — act immediately | Self-discovery |
| Security team audit | Medium–High | Formal notification |
| Bug bounty / external report | Very high — outsider found it | Email to security@ |
| Cloud provider alert (AWS, GCP) | Critical — provider detected usage | Email or dashboard |

**The mindset when you discover a leak:**

Panic is the enemy of an effective response. Your goal in the first five minutes is not to understand everything that happened — it is to stop the bleeding. Revoke first. Investigate second.

```bash
cd safety-journal
cat > incident-runbook.md << 'EOF'
# Secret Leak Incident Runbook

**Purpose:** Step-by-step response when a secret is found in code, logs, or anywhere unintended.
**Owner:** Fill in your name / team
**Last reviewed:** 2026-04-29

## Severity Classification
| Severity | Definition | Response time |
|----------|-----------|--------------|
| P1 — Critical | Production secret, actively exploitable, publicly visible | Immediate (< 15 min) |
| P2 — High | Production secret, limited visibility, or unknown exposure | < 1 hour |
| P3 — Medium | Non-production secret, internal visibility only | < 4 hours |
| P4 — Low | Placeholder / test value with no real access | Next business day |

EOF
```

---

## Step 2 — The Immediate Response: Revoke First

The sequence matters. Many developers instinctively try to delete the commit or rewrite history first — this is wrong. History rewriting takes time, and while you are doing it the secret is still live. Revoke the credential immediately, then clean the history.

```
CORRECT sequence:
──────────────────────────────────────────────────────────────────
Step 1  REVOKE the secret (< 5 minutes)
        Make the exposed credential invalid before doing anything else
        Even if it means a brief service outage

Step 2  NOTIFY your team / manager (< 15 minutes)
        Do not handle this alone; you need support and they need to know

Step 3  ASSESS blast radius (< 1 hour)
        What did the secret have access to? Are there audit logs?

Step 4  ISSUE a new credential and deploy it
        Only now do you replace the rotated secret with a new one

Step 5  CLEAN history (if applicable)
        Remove from git history; force-push with team coordination

Step 6  POST-MORTEM
        Document what happened and what changes prevent recurrence
```

**Where to revoke common secrets:**

| Secret type | Where to revoke |
|-------------|----------------|
| AWS keys | IAM console → Users / Roles → Security credentials → Deactivate |
| GitHub PAT | Settings → Developer settings → Personal access tokens → Delete |
| Stripe keys | Dashboard → Developers → API keys → Roll key |
| Google Cloud | IAM → Service accounts → Manage keys → Delete key |
| Database password | ALTER USER / RENAME USER / DROP USER depending on database |
| JWT signing secret | Rotate the secret AND invalidate all existing tokens (force re-login) |

```bash
cat >> incident-runbook.md << 'EOF'

## Immediate Response Steps

### Step 1: Revoke (do this FIRST — before anything else)
- [ ] Identify the secret type and which system it grants access to
- [ ] Navigate to the provider's credential management console
- [ ] Revoke / rotate / delete the exposed credential
- [ ] Confirm the old credential no longer works (test an API call)

### Step 2: Notify
- [ ] Message team lead / manager: "I found a [type] in [location]. I've revoked it. Investigating now."
- [ ] If personal data may have been accessed: notify security team immediately (GDPR 72-hour clock starts)
- [ ] Do NOT post the secret value in Slack or email — reference only the type and location

### Step 3: Assess Blast Radius
Questions to answer:
- When was the secret first committed / exposed?
- Was the repository public or private during that time?
- Does the provider have access logs for the credential?
- Were any unusual API calls made using this credential?
- What systems / data could have been accessed with this credential?
EOF
```

---

## Step 3 — Removing Secrets from Git History

After revoking the credential, clean the history to prevent the exposed value from being discovered by future readers (including automated scanners that index git history).

> **Important:** History rewriting changes commit hashes. Anyone who has cloned or forked the repository will have a diverged history. Coordinate with your team before rewriting a shared branch.

**The tool: `git filter-repo`**

`git filter-repo` is the modern, safe replacement for the deprecated `git filter-branch`. It is faster, safer, and recommended by the Git project.

```bash
# Install
pip install git-filter-repo

# First, create a fresh clone to work in — never rewrite on your working copy
git clone --mirror https://github.com/your-org/your-repo.git repo-mirror
cd repo-mirror
```

**Remove a file entirely from all history:**

```bash
# Remove a file that should never have been committed
git filter-repo --path secrets.yml --invert-paths
```

**Replace a secret value with a placeholder throughout all history:**

```bash
# Create a replacements file
echo 'sk_live_4xKqR2mN8pLzT7vW==>REDACTED_STRIPE_KEY' > replacements.txt
echo 'SuperSecret123==>REDACTED_PASSWORD' >> replacements.txt

git filter-repo --replace-text replacements.txt
```

**Push the rewritten history:**

```bash
# Force-push is required after history rewrite
# This overwrites the remote — coordinate with your team first
git push --force
```

After pushing, team members must re-clone or run:

```bash
git fetch --all
git reset --hard origin/main
```

> **GitHub's cache:** Even after rewriting history, GitHub may cache the old commits for up to 90 days. You can request an immediate cache purge via GitHub Support for confirmed security incidents. The secret should still be treated as permanently compromised — rotation is the real fix.

---

## Step 4 — Scanning History for Existing Secrets

Before a real incident, scan your repository proactively to find any secrets already lurking in history.

**Using `trufflehog`** — scans git history for high-entropy strings and known patterns:

```bash
pip install trufflehog3

# Scan the full history of a repository
trufflehog3 --format json . > trufflehog-report.json 2>/dev/null

# Human-readable output
trufflehog3 --no-history .
```

**Using `gitleaks`** — another widely used scanner with extensive pattern support:

```bash
# Install via the project's release page or:
# brew install gitleaks  (macOS)
# docker run zricethezav/gitleaks:latest

gitleaks detect --source . --report-path gitleaks-report.json
```

Add a scheduled scan to CI so history is scanned on every merge to main:

```bash
cat >> safe-app/.github/workflows/safe-ci.yml << 'EOF'

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # Full history — not just the latest commit

      - name: Scan for secrets in history
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
EOF
```

---

## Step 5 — Post-Mortem: Turning an Incident Into a System Improvement

A well-run post-mortem converts a painful incident into a permanent improvement. The goal is blameless analysis — understanding the system failure, not assigning fault to a person.

```bash
cat > safety-journal/post-mortem-template.md << 'EOF'
# Incident Post-mortem Template

## Summary
One paragraph: what happened, when, how it was detected, and what was done.

## Timeline
| Time | Event |
|------|-------|
| T+0  | Secret committed |
| T+?  | Secret discovered |
| T+?  | Credential revoked |
| T+?  | New credential deployed |
| T+?  | History cleaned |

## Blast Radius Assessment
- Was the repository public? Yes / No / Unknown
- Duration of exposure:
- Systems the secret had access to:
- Evidence of unauthorised access: Yes / No / Unknown
- Personal data potentially affected: Yes / No

## Root Cause
What was the immediate cause?

What system or process failure allowed this to happen?

## Action Items
| Action | Owner | Priority | Due date |
|--------|-------|----------|---------|
| | | | |

## What We Will Change
Prevention (stops this exact incident):

Detection (catches it faster next time):

Response (reduces blast radius if it happens again):
EOF
```

---

## Step 6 — Complete Your Incident Runbook

Finalise `incident-runbook.md` with all remaining steps and personalise it to your team's tooling.

```bash
cat >> incident-runbook.md << 'EOF'

## Step 4: Issue New Credentials
- [ ] Generate a new credential through the provider's console
- [ ] Store the new credential in the team's secret manager (not in a file)
- [ ] Update all environments (staging, production) with the new credential
- [ ] Verify the service is running correctly with the new credential

## Step 5: Clean History (if secret was in a git repository)
- [ ] Coordinate with team — history rewrite affects everyone
- [ ] Clone a fresh mirror of the repository
- [ ] Use git filter-repo to remove or redact the secret
- [ ] Force-push the cleaned history
- [ ] Notify all team members to re-clone
- [ ] Request GitHub cache purge if repository is public (via GitHub Support)

## Step 6: Post-mortem
- [ ] Schedule a blameless post-mortem within 48 hours
- [ ] Complete the post-mortem template
- [ ] Implement action items with owners and deadlines
- [ ] Update team runbooks and onboarding docs with lessons learned

## Contacts
| Role | Name / Channel |
|------|---------------|
| Team lead | |
| Security team | |
| On-call engineer | |
| GitHub Support (for cache purges) | github.com/contact/security |

## Pre-authorised Actions
The following actions are pre-approved without needing manager sign-off:
- Revoking any credential found in a repository
- Force-pushing to feature branches to clean history
- Creating a private incident channel in Slack

Actions requiring manager sign-off first:
- Force-pushing to main / production branches
- Filing a regulatory breach report
- Notifying affected users
EOF
```

```bash
cat incident-runbook.md
```

Congratulations — you have completed the Code Safety for New Graduates course. You can now identify what counts as a secret, understand how secrets leak through code, config files, linting tools, loggers, Docker, and CI pipelines, apply prevention at multiple layers, and respond systematically when something goes wrong. These skills will protect your users, your company, and your career — and the habits you build now are far easier to maintain than trying to add security as an afterthought to an existing project.
