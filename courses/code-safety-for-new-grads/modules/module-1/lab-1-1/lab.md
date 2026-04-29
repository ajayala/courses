# How Secrets Leak and Why It Is Your Problem

In this lab you will learn what counts as a secret in a software system, the surprisingly varied ways secrets reach the public internet, and what the consequences are — for users, for the company, and for you personally. This is not abstract security theory: secrets leaking through code is one of the most common and most preventable causes of data breaches, and new graduates are disproportionately responsible because nobody taught them to think about it.

**Prerequisites:** Basic programming experience. A terminal and a text editor. No security background required.

---

## Step 1 — What Counts as a Secret

Not all sensitive data is obvious. A secret is any value that, if known by an unintended party, could be used to cause harm — financial, reputational, legal, or operational.

```
Category             Examples
────────────────────────────────────────────────────────────────────
API keys             STRIPE_SECRET_KEY, OPENAI_API_KEY, TWILIO_AUTH_TOKEN
Database credentials DB_PASSWORD, connection strings with embedded passwords
Cloud credentials    AWS_SECRET_ACCESS_KEY, GCP service account JSON files
Authentication       JWT signing secrets, OAuth client secrets, session keys
Private keys         SSH private keys (~/.ssh/id_rsa), TLS certificates (.pem, .pfx)
Personal data        Passwords, social security numbers, payment card numbers
Internal URLs        Admin panel URLs, internal service addresses with auth tokens
Tokens               GitHub PATs, Slack tokens, webhook secrets
Config values        Encryption keys, salt values, HMAC secrets
```

> **The test for "is this a secret?"** Ask: *If a stranger on the internet had this value, could they do something I would not want?* If yes — it is a secret.

Create your safety journal:

```bash
mkdir safety-journal && cd safety-journal
cat > secret-classification.md << 'EOF'
# Secret Classification Reference

## What Counts as a Secret
My definition:

## Categories I Will Encounter in My First Job

### High Severity (leaked = immediate breach risk)
Examples from above:
- 

### Medium Severity (leaked = significant risk but limited window)
Examples:
- 

### Lower Severity (leaked = bad but recoverable quickly)
Examples:
- 

## The Test I Will Apply
"If a stranger had this value, could they..."

EOF
```

---

## Step 2 — How Secrets Actually Reach the Public

Most leaks are not malicious insiders — they are well-intentioned developers making easy mistakes. Here are the most common vectors, with real patterns you will encounter.

**Vector 1: Committed directly to a public repository**

```python
# app.py — pushed to a public GitHub repo
import stripe
stripe.api_key = "sk_live_4xKqR2mN8pLzT7vW3cY9oJ..."   # ← directly in code

DATABASE_URL = "postgresql://admin:SuperSecret123@prod-db.internal:5432/myapp"
```

GitHub scans public repositories for known secret patterns and notifies providers — but by the time the notification arrives, the key has often already been harvested by automated scanners that watch GitHub's public event stream in real time.

**Vector 2: Committed in a config file**

```bash
# config/settings.yml — developer assumed this would be gitignored but it wasn't
production:
  secret_key_base: "a1b2c3d4e5f6g7h8i9j0..."
  aws_access_key_id: "AKIAIOSFODNN7EXAMPLE"
  aws_secret_access_key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

**Vector 3: Left in git history after "deletion"**

```bash
# Developer adds secret, realises mistake, deletes it in next commit
git add config.yml    # commit A — secret added
git commit -m "Add config"

# Realises mistake...
# Removes the secret from config.yml
git add config.yml    # commit B — secret "removed"
git commit -m "Remove secret from config"
```

The secret is gone from the file — but commit A is still in the history forever. `git log -p` reveals it instantly.

**Vector 4: Printed to logs**

```python
def authenticate(request):
    logger.debug(f"Authenticating request: headers={request.headers}")
    #                                                 ↑
    # Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR...
    # This goes into your log aggregation system (Datadog, Splunk, CloudWatch)
    # which is often less tightly controlled than your code repository
```

**Vector 5: Exposed in error messages**

```
Internal Server Error

ValueError: Invalid database URL: postgresql://admin:SuperSecret123@prod-db.internal:5432/myapp
  File "app.py", line 42, in connect_db
    engine = create_engine(DATABASE_URL)

Environment:
  DATABASE_URL = postgresql://admin:SuperSecret123@...
```

Error pages that include environment variables are a surprisingly common production misconfiguration.

```bash
cat >> secret-classification.md << 'EOF'

## Leak Vectors I Need to Guard Against
| Vector | How it happens | How to prevent it |
|--------|---------------|-------------------|
| Direct commit | | |
| Config file commit | | |
| Git history | | |
| Logs | | |
| Error messages | | |
| CI/CD output | | |
| Tool output / linting | | |
EOF
```

---

## Step 3 — The Attacker's Timeline

You might assume that a leaked key would be noticed quickly and that the window for misuse is short. The reality is much more alarming.

**Typical timeline after a key is pushed to a public GitHub repo:**

```
T+0:00    Developer pushes commit containing API key to public repository
T+0:02    Automated scanners watching GitHub's public event stream detect the key
T+0:05    Key is tested against the provider API to confirm it is valid and live
T+0:15    Key is used to make first unauthorised API calls
T+0:60    Developer notices the mistake and deletes the file
          (Key is still in git history — still usable)
T+4:00    Developer rotates the key
          (Attackers have now had 4 hours of access)
T+unknown Historic usage is audited — if logs were not retained, the blast radius is unknown
```

Research by GitGuardian found that **leaked secrets are used within minutes** in many cases. The assumption that "nobody will find it that fast" is consistently wrong.

> **The 2022 Toyota leak:** A contractor accidentally published a GitHub repository containing an access key to Toyota's T-Connect service. The key was exposed for nearly five years before being discovered. 296,000 customers' data was potentially accessible during that window.

---

## Step 4 — The Consequences: Personal, Legal, and Regulatory

Leaked secrets are not just a company problem. As the developer who committed the secret, you may have direct personal exposure.

**For users:**

```
A leaked database credential → attacker dumps the user table
→ names, email addresses, hashed passwords, PII exposed
→ GDPR breach notification required within 72 hours
→ potential fines up to 4% of global annual revenue
→ users receive phishing emails using their real data for years
```

**For the company:**

```
A leaked AWS key → attacker spins up 500 GPU instances for crypto mining
→ $47,000 AWS bill in 6 hours (real incident, commonly reported)
→ Company news, customer trust damage, regulatory scrutiny
```

**For you personally:**

| Scenario | Potential consequence |
|----------|----------------------|
| Accidental commit of a secret | Disciplinary action (first offence, usually) |
| Ignoring a known risk | Performance management |
| Knowingly exposing secrets | Gross misconduct, termination, legal action |
| Leaking personal data | Personal liability under data protection law in some jurisdictions |

> **The law matters here.** Under GDPR (Europe), CCPA (California), and similar regulations, organisations and in some cases individuals have legal obligations around personal data protection. "I didn't know" is not a defence when the risk was foreseeable and basic precautions were not taken.

---

## Step 5 — Why This Hits New Graduates Harder

New graduates are statistically the most common source of accidental secret leaks — not because they are careless, but because they have specific blind spots that university does not address.

**Blind spot 1: Working in public by default**

University trains you to push code to public repositories for grading, portfolio-building, and open source contribution. The habit of "everything is public" does not automatically switch off when you start work.

**Blind spot 2: Tutorials use real-looking fake keys**

Every API tutorial shows code like:
```python
api_key = "YOUR_API_KEY_HERE"  # Replace with your key
```
Graduates learn the pattern of putting keys in code files — just with a placeholder. In the next step, they replace the placeholder with the real key and commit it.

**Blind spot 3: No one tested the `.gitignore`**

A `.gitignore` entry for `.env` only ignores files literally named `.env`. A file named `.env.local`, `config.env`, or `settings.env` is not ignored unless explicitly added.

**Blind spot 4: "I'll fix it later" becomes "it's in the history forever"**

Deleting a committed secret from a file does not remove it from history. Without `git filter-repo` or equivalent, the secret is permanent.

```bash
cat >> secret-classification.md << 'EOF'

## My Personal Risk Checklist
Things I will audit before every `git push`:
- [ ] No API keys, passwords, or tokens in any file I am committing
- [ ] No database connection strings with embedded credentials
- [ ] .env and config files containing secrets are in .gitignore
- [ ] I have not added any new file types that could contain secrets
- [ ] My commit does not include log files or tool output that might capture env vars

## Questions I Will Ask on Day One
- Where does the team store secrets? (Vault, AWS Secrets Manager, 1Password Teams?)
- Is there a secret scanning tool on the repository?
- What is the procedure if I accidentally commit a secret?
- Are there pre-commit hooks I should install?
EOF
```

---

## Step 6 — Complete Your Secret Classification Reference

Fill in every section of `secret-classification.md`. This document is your quick reference for the rest of the course and for your first weeks on the job.

```bash
cat secret-classification.md
```

Your file should contain:
- Your own definition of "secret"
- The severity classification table with real examples in each tier
- The leak vectors table with prevention strategies filled in
- Your personal pre-push checklist
- The Day One questions list

Here is an example completed row for the leak vectors table to guide your style:

```markdown
| Logs | A logger captures the full request object including Authorization headers;
         the log is shipped to a third-party aggregation service with wider access
         than the codebase itself | Never log raw headers or request bodies;
         use structured logging with explicit field allowlists |
```

In the next lab you will build the core habit that prevents the majority of leaks: separating secrets from code using environment variables and understanding exactly where that separation can fail.
