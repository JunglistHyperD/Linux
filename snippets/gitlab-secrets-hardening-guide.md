# GitLab secrets hardening guide

**Scope:** our team's GitLab group and its projects, developer MacBooks, and how secrets reach our code locally and in CI. **Environment:** GitLab 19.5 EE (Ultimate), team group owned by us inside a parent group, managed macOS laptops (no sudo), AWS access via IAM Identity Center (SSO), AWS CLI v2, AWS Secrets Manager as the only secrets store. **Status:** draft for team review. Items marked ⚠️ need verifying in our environment or need another team's help.

---

## 1. Principles

1. **Secrets never live in files.** Code reads secrets at runtime from AWS Secrets Manager. Repositories hold only references and placeholders.
2. **Defence in depth.** Every layer assumes the one before it failed.
3. **Server-side controls are the real enforcement.** Anything on a laptop can be skipped. Anything in GitLab can't be skipped by a developer alone.
4. **A pushed secret is a leaked secret.** Rotation is the fix. History cleanup is tidying up afterwards.
5. **"Sensitive" means more than credentials.** Internal hostnames, URLs, account IDs and architecture details count too, and they need custom detection rules plus human review.

## 2. The journey at a glance

| Step | Where | Control | Can a developer bypass it? |
| --- | --- | --- | --- |
| 1 | Developer Mac | One-off setup: AWS SSO, pre-commit, global ignore | Yes |
| 2 | AWS Secrets Manager | Personal and team secrets, pulled at runtime | n/a — it removes the problem |
| 3 | Repository | Template, `.env.refs`, local scanning | Yes |
| 4 | GitLab, on push | Secret push protection, push rules | Only by a deliberate, audited skip |
| 5 | GitLab, merge request | CI secret detection, protected branches, policies | No |
| 6 | GitLab, at rest | History scans, remediation runbook | n/a |

---

## 3. Developer Mac — one-off setup

### 3.1 AWS access

Install AWS CLI v2 for your user (official `.pkg`, *Install for me only*), then:

```zsh
aws configure sso
```

Add your defaults to `~/.zshrc`. `SECRETS_USER` must exactly match the username AWS sees in your session tag. ⚠️ Confirm the format with the AWS admins, for example `firstname.lastname`.

```zsh
echo 'export AWS_PROFILE=YOUR_PROFILE' >> ~/.zshrc
echo 'export AWS_REGION=eu-west-2' >> ~/.zshrc
echo 'export SECRETS_USER=your.sso.username' >> ~/.zshrc
source ~/.zshrc
```

Each working day:

```zsh
aws sso login
aws sts get-caller-identity
```

### 3.2 Local tooling (user-level, no sudo)

```zsh
pip3 install --user pre-commit git-filter-repo
```

Optional: install the `gitleaks` and `trufflehog` release binaries into `~/.local/bin`. ⚠️ Confirm that managed-device policy allows downloaded binaries to run.

### 3.3 Global safety nets

A global ignore file covers every repository on the machine:

```zsh
mkdir -p ~/.config/git
cat >> ~/.config/git/ignore <<'EOF'
.env
.env.*
!.env.example
!.env.refs
*.pem
*.key
*.p12
*.pfx
id_rsa*
id_ed25519*
.aws/
EOF
```

Every future clone or `git init` then gets the pre-commit hooks automatically:

```zsh
pre-commit init-templatedir ~/.git-template
git config --global init.templateDir ~/.git-template
```

---

## 4. Secrets in AWS Secrets Manager

### 4.1 Layout

```
dev/personal/<username>/<name>   only you can read or write
team/<teamname>/<name>           everyone in the team reads; named maintainers write
```

Rules:

- Use neutral names, such as `team/myteam/app1-db` rather than `team/myteam/prod-splunk-admin`.
- Use lowercase letters, digits and hyphens only.
- Team secret names form a shared vocabulary, so agree them in the team and record them in each project's `.env.refs`.

### 4.2 Access model ⚠️ *Needs AWS admins*

Enable *attributes for access control* in IAM Identity Center, so each user's username arrives as the session tag `username`.

**Team developer permission set:** everyone gets this.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "OwnPersonalSecrets",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:CreateSecret",
        "secretsmanager:PutSecretValue",
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret",
        "secretsmanager:DeleteSecret"
      ],
      "Resource": "arn:aws:secretsmanager:eu-west-2:ACCOUNT_ID:secret:dev/personal/${aws:PrincipalTag/username}/*"
    },
    {
      "Sid": "TeamSecretsRead",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:eu-west-2:ACCOUNT_ID:secret:team/myteam/*"
    },
    {
      "Sid": "GeneratePasswords",
      "Effect": "Allow",
      "Action": "secretsmanager:GetRandomPassword",
      "Resource": "*"
    }
  ]
}
```

**Team secret maintainers:** an extra permission set for a few people.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "TeamSecretsWrite",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:CreateSecret",
        "secretsmanager:PutSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:eu-west-2:ACCOUNT_ID:secret:team/myteam/*"
    }
  ]
}
```

Notes:

- The trailing `/*` is required, because AWS appends a random six-character suffix to every secret ARN.
- `ListSecrets` is deliberately **not** granted. It can't be limited to particular paths, so granting it would reveal the names of everyone's personal secrets. Projects record the names they use in `.env.refs` instead.
- These are design sketches. Test them in a non-production account first.

**How private are personal secrets?** They're private from other developers. Account administrators with broad Secrets Manager rights can still read them, unless a resource policy denies it. ⚠️ Agree with the AWS admins whether to add an owner-only resource policy, without breaking break-glass access.

**Laptop-theft mitigations** ⚠️ *Needs AWS admins*

- Keep the SSO session duration short (for example, one hour).
- Require MFA at sign-in.
- If a laptop is lost, disable the user in Identity Center. Access stops at once, and CloudTrail shows what was read.
- Optional: encrypt `team/` secrets with a dedicated KMS key, which adds a second permission gate.

### 4.3 Creating, generating and reading secrets

Commit this helper as `scripts/secrets.sh` in the template. It can generate a strong secret for you, or prompt for one you already have. The value never appears in shell history or `ps` output, and nothing is shown on screen unless you ask. ⚠️ Test it with a dummy secret first.

```zsh
#!/usr/bin/env zsh
set -euo pipefail
TEAM="${TEAM:-myteam}"
LENGTH=29
GENERATE=false; SHOW=false; COPY=false

usage() {
  print "Usage: $0 set {personal|team} NAME [--generate [LENGTH]] [--show] [--copy]"
  print "       $0 get {personal|team} NAME [--show] [--copy]"
  exit 1
}

[[ $# -ge 3 ]] || usage
ACTION="$1"; SCOPE="$2"; NAME="$3"; shift 3
[[ "$NAME" =~ '^[a-z0-9-]+$' ]] || { print "Name: lowercase letters, digits, hyphens only"; exit 1 }

case "$SCOPE" in
  personal) ID="dev/personal/${SECRETS_USER:?Set SECRETS_USER}/$NAME" ;;
  team)     ID="team/$TEAM/$NAME" ;;
  *)        usage ;;
esac

while (( $# )); do
  case "$1" in
    --generate) GENERATE=true
                if [[ "${2:-}" =~ '^[0-9]+$' ]]; then LENGTH="$2"; shift; fi ;;
    --show)     SHOW=true ;;
    --copy)     COPY=true ;;
    *)          usage ;;
  esac
  shift
done

reveal() {
  if $SHOW; then print -r -- "$1"; fi
  if $COPY; then print -rn -- "$1" | pbcopy; print "Copied to clipboard"; fi
}

case "$ACTION" in
  get)
    VAL="$(aws secretsmanager get-secret-value \
      --secret-id "$ID" --query SecretString --output text)"
    if ! $SHOW && ! $COPY; then SHOW=true; fi
    reveal "$VAL" ;;
  set)
    if $GENERATE; then
      VAL="$(aws secretsmanager get-random-password \
        --password-length "$LENGTH" --require-each-included-type \
        --exclude-characters "\"'\`\\\$" \
        --query RandomPassword --output text)"
    else
      read -rs "VAL?Value for $ID: "; echo
    fi
    if aws secretsmanager describe-secret --secret-id "$ID" >/dev/null 2>&1; then
      print -rn -- "$VAL" | aws secretsmanager put-secret-value \
        --secret-id "$ID" --secret-string file:///dev/stdin >/dev/null
      print "Updated $ID"
    else
      print -rn -- "$VAL" | aws secretsmanager create-secret \
        --name "$ID" --secret-string file:///dev/stdin >/dev/null
      print "Created $ID"
    fi
    reveal "$VAL" ;;
  *) usage ;;
esac
unset VAL
```

Usage:

```zsh
# New personal secret, 29 characters, generated by AWS, not shown
./scripts/secrets.sh set personal app1-api-token --generate

# Same, but 40 characters and shown once on screen
./scripts/secrets.sh set personal app1-api-token --generate 40 --show

# Store a value you already have (prompts, nothing echoed)
./scripts/secrets.sh set team app1-db

# Read a secret back: prints by default, or --copy to clipboard instead
./scripts/secrets.sh get personal app1-api-token
./scripts/secrets.sh get team app1-db --copy
```

Notes:

- **Generated secrets** always contain uppercase letters, lowercase letters, digits and punctuation. Quotes, backticks, backslashes and `$` are excluded so values never break shell scripts, JSON or connection strings. At 29 characters that is roughly 185 bits of randomness.
- **`--show`** puts the value in your terminal scrollback. Prefer `--copy` and clear the clipboard afterwards. Clipboard-history apps may keep a copy.
- **The same secret** is used by `run-with-secrets.sh` locally and by CI builds, so there is one source of truth.
- **Rotation** is the same command with `--generate` again. Anything reading the secret picks up the new value on its next run.

### 4.4 Using secrets in a project

Commit a `.env.refs` file containing **names and references only**. `{me}` is replaced with your `SECRETS_USER` at runtime, so personal references can be committed without anyone's username in them.

```
# aws: = AWS Secrets Manager secret ID; anything else is a plain setting
DB_PASSWORD=aws:team/myteam/app1-db
API_TOKEN=aws:dev/personal/{me}/app1-api-token
LOG_LEVEL=debug
```

Commit this wrapper as `scripts/run-with-secrets.sh`. It resolves the references in memory and then runs your command.

```zsh
#!/usr/bin/env zsh
set -euo pipefail
REFS="${REFS:-.env.refs}"

while IFS='=' read -r name ref; do
  [[ -z "$name" || "$name" == \#* ]] && continue
  if [[ "$ref" == aws:* ]]; then
    id="${ref#aws:}"
    if [[ "$id" == *"{me}"* ]]; then
      : "${SECRETS_USER:?Set SECRETS_USER to your SSO username}"
      id="${id//\{me\}/$SECRETS_USER}"
    fi
    value="$(aws secretsmanager get-secret-value \
      --secret-id "$id" --query SecretString --output text)"
  else
    value="$ref"
  fi
  export "$name=$value"
done < "$REFS"

exec "$@"
```

Usage:

```zsh
./scripts/run-with-secrets.sh npm run build
```

Rules:

- Never use `set -x` in scripts that handle secrets.
- Never print the environment (`env`, `printenv`) in builds.
- Never write resolved values back to a `.env` file.

### 4.5 CI/CD

- **Pipelines read the same team secrets through OIDC.** Jobs request an `id_tokens` JWT and assume an IAM role that can read `team/myteam/*`, so no long-lived AWS keys are stored in GitLab. ⚠️ *Needs AWS admins* to create the IAM OIDC provider and role trust policy, restricted to our group's projects and protected branches.
- **Pipelines never read `dev/personal/*`.**
- **Any GitLab CI/CD variable that must exist** is set to *Masked and hidden* and *Protected*, and is scoped to the environment that needs it.
- **Never echo variables** in job scripts.

---

## 5. Repository — template and local scanning

### 5.1 Pre-commit configuration

`.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: vX.Y.Z            # pin to the latest release tag
    hooks:
      - id: detect-private-key
      - id: detect-aws-credentials
        args: [--allow-missing-credentials]
      - id: check-added-large-files

  - repo: https://github.com/gitleaks/gitleaks
    rev: vX.Y.Z            # pin to the latest release tag
    hooks:
      - id: gitleaks
```

Notes:

- Pin `rev` to exact release tags and update them deliberately, because hooks are third-party code running on your machine.
- If the gitleaks hook can't build on our laptops, switch it to a `repo: local` hook that calls the `gitleaks` binary from `~/.local/bin`.
- TruffleHog can be added the same way as a local hook. Its strength is verifying whether a found credential is live.

### 5.2 Custom detection rules

`.gitleaks.toml` extends the default rules with our own patterns. Replace the examples with our real internal domains and formats.

```toml
[extend]
useDefault = true

[[rules]]
id = "internal-hostname"
description = "Internal hostname"
regex = '''(?i)\b[a-z0-9-]+\.corp\.example\.com\b'''

[[rules]]
id = "aws-account-id-arn"
description = "AWS ARN containing an account ID"
regex = '''arn:aws:[a-z0-9-]+:[a-z0-9-]*:\d{12}:'''
```

Expect some false positives at first. Tune the rules rather than disabling them.

### 5.3 The template repository

Create `team-template` in the group, and use it for every new project. ⚠️ Group-level custom project templates require a templates subgroup to be set in group settings.

```
.gitignore                         patterns from section 3.3
.pre-commit-config.yaml            section 5.1
.gitleaks.toml                     section 5.2
.env.refs                          example references, no values
.env.example                       placeholders only
scripts/secrets.sh                 section 4.3
scripts/run-with-secrets.sh        section 4.4
.gitlab-ci.yml                     includes Secret Detection (section 7.2)
.gitlab/secret-detection-ruleset.toml
CODEOWNERS                         section 7.1
README.md                          setup steps from sections 3 and 4
```

After cloning, each developer runs `pre-commit install`, unless the global template directory from section 3.3 already did it.

A template only sets defaults. The enforcement comes from group push rules (section 6.2) and group policies (section 7.3).

---

## 6. GitLab — blocking at push time (server-side)

### 6.1 Secret push protection (Ultimate)

- **Enable it:** in each project, go to **Secure → Security configuration** and switch on **Secret push protection**. Also enable it in the template project. ⚠️ Check whether our group's settings offer group-wide enablement in 19.5.
- **What it does:** rejects pushes containing known high-confidence secret patterns, before anything is written to the repository.
- **Known gap:** developers can deliberately skip it, using a push option or a skip marker in the commit message. Each skip is recorded in **audit events**, so review those regularly or alert on them.

### 6.2 Group push rules

In **Group → Settings → Repository → Pre-defined push rules**:

- ✅ **Prevent pushing secret files**
- ✅ **Check whether the commit author is a GitLab user**
- **Prohibited file names** (regex):

```
(^|/)(\.env|.*\.pem|.*\.key|.*\.p12|.*\.pfx|id_rsa.*|id_ed25519.*|credentials)$
```

- **Maximum file size:** set a sensible limit, because large blobs are often dumps or exports.

Group push rules are inherited by **new** projects. Check each of the existing six or seven projects and apply the same rules at project level.

---

## 7. GitLab — merge requests and CI

> ⚠️ This layer needs CI runners. Confirm on **Settings → CI/CD → Runners** (group and instance sections) or with a test pipeline.

### 7.1 Branch and merge request settings (every project)

- **Protected branch `main`:** set *Allowed to push and merge* to No one for pushes and Maintainers for merges, and turn *Allowed to force push* off.
- **Merge request settings:** turn on *Pipelines must succeed* and *All threads must be resolved*.
- **`CODEOWNERS`:** require approval from security-minded owners for `.gitlab-ci.yml`, `.pre-commit-config.yaml`, `.gitleaks.toml`, `.env.refs`, `scripts/` and `.gitlab/`, so nobody can quietly weaken the controls.

### 7.2 Pipeline secret detection

Add this to `.gitlab-ci.yml`:

```yaml
include:
  - template: Jobs/Secret-Detection.gitlab-ci.yml
```

GitLab's analyser is gitleaks-based. Custom rules go in `.gitlab/secret-detection-ruleset.toml`. ⚠️ Check GitLab's custom-ruleset documentation for 19.5 to reuse the `.gitleaks.toml` rules.

### 7.3 Group security policies (enforcement even without project config)

In **Group → Secure → Policies**:

- **Scan execution policy:** run secret detection on every pipeline in every project in the group, including new projects and projects whose `.gitlab-ci.yml` leaves it out.
- **Merge request approval policy:** if secret detection finds anything new, require approval from a named security approver before merging.

Policies are stored in a linked security policy project. ⚠️ Confirm that our owner role can create and link one, or whether the parent group already has one that applies to us.

---

## 8. GitLab — repository at rest

### 8.1 One-off baseline scan (now)

Locally, in each project:

```zsh
gitleaks git -v .
trufflehog git file://. --results=verified,unknown
```

Run a full-history scan in CI once by setting this variable on a manual pipeline:

```
SECRET_DETECTION_HISTORIC_SCAN=true
```

### 8.2 Ongoing

- Add a scheduled pipeline, or a scheduled scan execution policy, that runs secret detection weekly.
- Triage **Secure → Vulnerability report** for secret findings.
- Review audit events for push protection skips.

### 8.3 Remediation runbook: a secret was committed

1. **Rotate the secret immediately** with `./scripts/secrets.sh` (and in the issuing system). This is the only step that actually fixes the problem.
2. **Check exposure:** forks, clones, merge request diffs and comments, CI job logs, artefacts, issues, wiki pages and snippets.
3. **Rewrite history locally on a fresh mirror clone:**

```zsh
git clone --mirror git@gitlab.example.com:group/project.git
cd project.git
# Remove a whole file from all history
git filter-repo --invert-paths --path path/to/secret.file
# Or replace secret strings (one per line in replacements.txt)
git filter-repo --replace-text ../replacements.txt
```

4. **Temporarily allow force-push** on the protected branch, then push:

```zsh
git push --force --mirror
```

Turn force-push off again afterwards.

5. **Purge server-side copies:** in **Settings → Repository → Repository maintenance**, use **Remove blobs** (with the blob IDs) and **Redact text**. This clears copies that GitLab keeps in internal references. ⚠️ Follow GitLab's 19.5 documentation for each step.
6. **Erase affected CI jobs**, which removes their logs and artefacts, and edit any issues or comments that contain the secret.
7. **Everyone deletes their local clone and re-clones.** Old clones still contain the secret and could push it back.
8. **Close the finding** in the vulnerability report and record the incident.

To find a blob ID for **Remove blobs**:

```zsh
git rev-list --objects --all | grep "path/to/secret.file"
```

---

## 9. Residual risks and gaps

| Gap | Why it can't be fully closed | Mitigation |
| --- | --- | --- |
| Pre-commit hooks can be skipped or never installed | Client-side, by design | Server-side push and merge controls |
| Push protection can be skipped deliberately | GitLab provides a skip option | Audit-event monitoring; merge request approval policy |
| Scanners miss unknown secret formats and "sensitive context" | Pattern-based detection | Custom rules; code review; CODEOWNERS |
| Pushed secrets survive in clones, forks and logs | Git is distributed | Rotation first; remediation runbook |
| Secrets sit in memory on laptops while in use | Unavoidable at runtime | Short sessions, MFA, FileVault, no persistent `.env` |
| Account admins can read personal secrets | Broad admin rights in the account | Optional owner-only resource policy |
| Secret names are visible in `.env.refs` | References must be shared to be used | Neutral naming; `{me}` placeholder |
| Parent group or instance settings may override ours | Outside our permissions | Confirm with the parent group owners |
| CI layer depends on runners | Infrastructure we don't own | Confirm runner availability |

---

## 10. Rollout plan

**Phase 1 — this week (no dependencies)**

- Enable secret push protection on all projects.
- Configure group push rules, then check the existing projects.
- Set protected branches and merge request settings.
- Run the baseline history scans (section 8.1).

**Phase 2 — secrets store (needs AWS admins)**

- Enable ABAC session tags and create both permission sets.
- Agree the team secret names.
- Each developer completes section 3.1 and tests `scripts/secrets.sh` with a dummy secret.
- Migrate secrets out of local `.env` files into Secrets Manager, and replace those files with `.env.refs`.

**Phase 3 — template and laptops**

- Build `team-template`.
- Each developer completes sections 3.2 and 3.3.
- Add the template files to existing projects.

**Phase 4 — CI and policy (needs runners)**

- Add pipeline secret detection.
- Set up OIDC for CI.
- Add group scan execution and merge request approval policies.
- Add weekly scheduled scans.

### Requests to other teams

| Team | Ask |
| --- | --- |
| AWS / IAM admins | ABAC session tags, two permission sets, short session duration, MFA, CI OIDC role, decision on owner-only resource policy |
| Platform / GitLab owners | Runner availability; security policy project; parent-group settings |
| Corporate IT | Allow-listing for user-level installs (pre-commit, gitleaks, TruffleHog, AWS CLI v2) |
| Security governance | Approval of the team secrets store design |