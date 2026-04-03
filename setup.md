# EC2 Setup Guide: 6 Concurrent Claude Code Instances
### AWS Console Edition (v10 — Envoy Credential Proxy + Sandbox Runtime)

A step-by-step guide using the AWS web console to set up a dedicated EC2 server for running up to 6 Claude Code instances simultaneously, each working on its own branch and pushing to GitHub.

**Daily workflow:** SSH in, type `code`, pick a repo, start coding. Repeat in as many terminals as you need.

**Process isolation:** Each Claude Code instance runs directly in its SSH session. No Docker, no containers.

**Access method:** AWS SSM tunnel + FIDO2 SSH key (YubiKey touch required per connection, no open ports, no public IP).

**Secrets management:** Envoy reverse proxy injects API credentials into outbound requests. Agent processes never see raw API keys — not in environment variables, not in process memory, not in `/proc`. Secrets are fetched from AWS Secrets Manager at boot and stored only in Envoy's root-owned config.

**Sandbox enforcement:** Every Claude Code session runs inside the sandbox-runtime, which enforces OS-level filesystem and network isolation via bubblewrap. Bash commands are confined to the working directory and can only reach `localhost` (via per-slot socat bridges to Envoy's unix sockets) — no direct outbound network access. The sandbox is mandatory; Claude Code refuses to start if the sandbox is unavailable.

**Workspace management:** Git worktrees from a pool of clones. Repos and slot assignments are fully dynamic.

**Slot attribution:** Each slot gets dedicated proxy bridge ports and writes structured session logs, enabling per-slot correlation of Envoy access logs during incident review.

---

## 1. Create the IAM Role (for the EC2 Instance)

This role gives the instance the minimum permissions it needs: SSM connectivity, CloudWatch monitoring, and Secrets Manager access.

1. Go to **IAM → Roles → Create role**
2. Trusted entity type: **AWS service**
3. Use case: **EC2** → Next
4. Search and attach: `AmazonSSMManagedInstanceCore`
5. Click **Next**, name the role `dev` → **Create role**
6. Open the role → **Add permissions → Create inline policy** → JSON tab:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CloudWatchMetrics",
      "Effect": "Allow",
      "Action": "cloudwatch:PutMetricData",
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:YOUR_REGION:YOUR_ACCOUNT_ID:log-group:/cloudwatch/*"
    },
    {
      "Sid": "SecretsManagerRead",
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:YOUR_REGION:YOUR_ACCOUNT_ID:secret:api-keys-*"
    },
    {
      "Sid": "SSMSessionLogsS3",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl"
      ],
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME-ssm-session-logs/*"
    }
  ]
}
```

> Replace `YOUR_REGION`, `YOUR_ACCOUNT_ID`, and `YOUR_BUCKET_NAME` with your actual values (e.g. `us-east-1`, `123456789012`, and `mycompany`).

7. Name it `dev-permissions` → **Create policy**

> Note the role ARN (e.g. `arn:aws:iam::ACCOUNT_ID:role/dev`). You'll need it in step 3.


---

## 2. Create an IAM User for Your MacBook

Create a dedicated user with only the permission to open SSM sessions to the dev server.

1. Go to **IAM → Users → Create user**
2. User name: `dev-access`
3. Do **not** check "Provide user access to the AWS Management Console" (CLI-only)
4. Click **Next → Attach policies directly → Create policy** → JSON tab:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SSMSessionToDevOnly",
      "Effect": "Allow",
      "Action": [
        "ssm:StartSession",
        "ssm:TerminateSession",
        "ssm:ResumeSession"
      ],
      "Resource": [
        "arn:aws:ec2:*:*:instance/REPLACE_WITH_INSTANCE_ID_AFTER_LAUNCH",
        "arn:aws:ssm:*::document/AWS-StartSSHSession",
        "arn:aws:ssm:*:*:document/SSM-SessionManagerRunShell"
      ]
    }
  ]
}
```

5. Name the policy `dev-ssm-only` → **Create policy**
6. Back in the user creation flow, refresh the policy list, search for `dev-ssm-only`, check it → **Next → Create user**
7. Open the user → **Security credentials → Create access key**
8. Use case: **Command Line Interface** (dismiss the warning about alternatives)
9. Save the **Access key ID** and **Secret access key**

> **Important:** After launching the instance in step 6, come back and replace `REPLACE_WITH_INSTANCE_ID_AFTER_LAUNCH` with the actual instance ID (e.g., `i-0abc123def456`).

### Configure the profile on your MacBook

```bash
aws configure --profile dev
# Access key ID: paste it
# Secret access key: paste it
# Default region: your region (e.g., us-east-1)
# Default output format: json
```


---

## 3. Store Your Secrets in AWS Secrets Manager

1. Go to **Secrets Manager → Store a new secret**
2. Secret type: **Other type of secret**
3. Key/value pairs:
   - `ANTHROPIC_API_KEY` = `sk-ant-...`
   - `GITHUB_TOKEN` = `github_pat_...` (fine-grained PAT — see below)
   - `NPM_PRIVATE_TOKEN` = `npm_...` (optional — only if you use a private npm registry)
   - Add any other keys you need
4. Encryption key: leave as default (`aws/secretsmanager`)
5. Click **Next**
6. Secret name: `api-keys`
7. Description: `API keys for coding agents`
8. Under **Rotation**, leave disabled
9. Click **Next → Store**

### Create the GitHub Fine-Grained PAT

1. On GitHub: **Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token**
2. Token name: `dev-agent`
3. Expiration: **90 days** (set a calendar reminder to rotate)
4. Repository access: **Only select repositories** → pick all repos your agents will work on
5. Permissions:
   - Repository → **Contents**: Read and write
   - Repository → **Pull requests**: Read and write
6. **Generate token**
7. Copy the `github_pat_...` value into the Secrets Manager entry above

> When you add new repos later, edit the PAT's repository access on GitHub to include them.

### Set the Resource Policy

1. Open the secret you just created
2. Click **Resource permissions → Edit permissions**
3. Paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::ACCOUNT_ID:role/dev"
        ]
      },
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "*"
    }
  ]
}
```

4. Replace `ACCOUNT_ID` with your AWS account ID
5. **Save**

### Using Secrets Locally on Your MacBook

```bash
dev-secrets() {
  local secret_json
  secret_json=$(aws secretsmanager get-secret-value \
    --secret-id "api-keys" \
    --query 'SecretString' \
    --output text)

  if [ -z "$secret_json" ]; then
    echo "ERROR: Failed to fetch secrets" >&2
    return 1
  fi

  while IFS='=' read -r key value; do
    export "$key"="$value"
  done < <(echo "$secret_json" | jq -r 'to_entries[] | "\(.key)=\(.value)"')
}
# Usage: dev-secrets && claude
```


---

## 4. Create the VPC

### VPC and Subnet

1. Go to **VPC → Your VPCs → Create VPC**
2. Resources to create: **VPC and more**
3. Name tag auto-generation: `dev`
4. IPv4 CIDR block: `10.99.0.0/16`
5. Number of Availability Zones: **1**
6. Number of public subnets: **0**
7. Number of private subnets: **1**
8. NAT gateways: **None** (we'll add one manually)
9. VPC endpoints: **None** (we'll add them manually)
10. **Create VPC**

### Security Groups

**1. Instance security group:**

1. Go to **VPC → Security Groups → Create security group**
2. Name: `dev-sg`
3. Description: `Agent server - no inbound, restricted outbound`
4. VPC: `dev-vpc`
5. Inbound rules: **leave empty**
6. Outbound rules: **delete the default "allow all" rule**, then add:

| Type | Port | Destination | Description |
|------|------|-------------|-------------|
| HTTPS | 443 | `0.0.0.0/0` | GitHub, Anthropic API, npm, pip |
| DNS (TCP) | 53 | `10.99.0.0/16` | VPC DNS resolution |
| DNS (UDP) | 53 | `10.99.0.0/16` | VPC DNS resolution |

> **Why restrict outbound:** The default "all traffic" rule lets agents exfiltrate data on any port. Restricting to port 443 eliminates raw TCP/UDP exfil channels. For tighter control, deploy a transparent HTTPS proxy with domain allowlisting as a follow-up hardening step.

7. **Create security group**

**2. VPC endpoints security group:**

1. **Create security group** again
2. Name: `dev-vpce-sg`
3. Description: `Allow HTTPS from VPC for SSM endpoints`
4. VPC: `dev-vpc`
5. Inbound rules: **Add rule** → Type: **HTTPS**, Source: `10.99.0.0/16`
6. Outbound rules: leave default
7. **Create security group**

### VPC Endpoints for SSM

Create each of these three endpoints:

| Endpoint | Service name |
|----------|-------------|
| SSM | `com.amazonaws.REGION.ssm` |
| SSM Messages | `com.amazonaws.REGION.ssmmessages` |
| EC2 Messages | `com.amazonaws.REGION.ec2messages` |

For each one:

1. Go to **VPC → Endpoints → Create endpoint**
2. Service category: **AWS services**
3. Search for the service name
4. VPC: `dev-vpc`
5. Subnets: check your private subnet
6. Security group: `dev-vpce-sg`
7. Policy: **Full access**
8. **Create endpoint**

> **Why keep these despite having a NAT gateway:** If the NAT gateway goes down, you lose both work connectivity (GitHub, Anthropic API) and management access (SSM) simultaneously. The VPC endpoints give you an independent management plane — you can still SSH in to diagnose and fix the NAT issue. For a single server where SSM is your only way in (no public IP, no bastion), that independence is worth the ~$23/mo.

### NAT Gateway for Outbound Internet

1. Create a **public subnet** for the NAT gateway:
   - Go to **VPC → Subnets → Create subnet**
   - VPC: `dev-vpc`
   - Name: `dev-nat-subnet`
   - CIDR: `10.99.2.0/24`
   - **Create subnet**

2. Create an **internet gateway**:
   - Go to **VPC → Internet gateways → Create**
   - Name: `dev-igw`
   - Attach to `dev-vpc`

3. Create a **route table** for the NAT subnet:
   - Go to **VPC → Route tables → Create**
   - Name: `dev-nat-rt`
   - VPC: `dev-vpc`
   - Add route: `0.0.0.0/0` → `dev-igw`
   - Associate with `dev-nat-subnet`

4. Create the **NAT gateway**:
   - Go to **VPC → NAT gateways → Create**
   - Name: `dev-nat`
   - Subnet: `dev-nat-subnet`
   - Connectivity type: **Public**
   - Allocate an Elastic IP
   - **Create**

5. Update the **private subnet's route table**:
   - Find the route table associated with your private subnet
   - Add route: `0.0.0.0/0` → `dev-nat`


---

## 5. Enable VPC Flow Logs

1. Go to **VPC → Your VPCs** → select `dev-vpc`
2. Click **Actions → Create flow log**
3. Filter: **All**
4. Maximum aggregation interval: **1 minute**
5. Destination: **Send to CloudWatch Logs**
6. Destination log group: click **Create new** → name it `/vpc/dev-flow-logs` → set retention to **30 days**
7. IAM role: click **Create new role** (the wizard generates the correct trust policy and CloudWatch Logs permissions automatically)
8. **Create flow log**


---

## 6. Launch the Instance

1. Go to **EC2 → Instances → Launch instances**

### Name and Tags

- Name: `dev`

### AMI

- Click **Browse more AMIs** → search: `al2023-ami` and filter for **64-bit (Arm)**
- Select the most recent **Amazon Linux 2023 AMI** (arm64, HVM, gp3) owned by **Amazon**

### Instance Type

- `m6g.2xlarge` (8 vCPU, 32 GB RAM) for up to 6 concurrent agents

### Key Pair

- Select **Proceed without a key pair**

### Network Settings

- Click **Edit**
- VPC: `dev-vpc`
- Subnet: the **private** subnet
- Auto-assign public IP: **Disable**
- Select existing security group: `dev-sg`

### Configure Storage

- Size: **200 GiB**
- Volume type: **gp3**
- IOPS: **6000**
- Throughput: **400** MB/s
- Encrypted: **Yes**

### Advanced Details

- IAM instance profile: `dev`
- **Metadata version**: **V2 only (token required)**
- **Metadata response hop limit**: **2**
- Scroll to **User data** and paste:

```bash
#!/bin/bash
set -eo pipefail

dnf update -y

# ── Create users ──
useradd -m -s /bin/bash agent
useradd -m -s /bin/bash admin
echo 'admin ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/admin
chmod 440 /etc/sudoers.d/admin

# ── Envoy system user (no login, no home) ──
useradd -r -s /usr/sbin/nologin envoy

# ── Add agent to envoy group (for unix socket access) ──
usermod -aG envoy agent

# ── Core tools ──
dnf install -y git tmux htop jq unzip gcc gcc-c++ make openssl-devel bubblewrap socat iptables-nft acl psmisc

# Node.js 22
curl -fsSL https://rpm.nodesource.com/setup_22.x | bash -
dnf install -y nodejs

# Claude Code
npm install -g @anthropic-ai/claude-code

# Sandbox Runtime (OS-level filesystem and network isolation for Claude Code)
npm install -g @anthropic-ai/sandbox-runtime

# ── Install Envoy (with integrity verification) ──
# Before deploying, obtain the correct binary URL and SHA256 hash:
#   1. Go to https://github.com/envoyproxy/envoy/releases
#   2. Find the version below and locate the binary for your architecture
#      (artifact names vary across versions — verify the exact filename)
#   3. Download the binary to your MacBook:
#        curl -fsSL -O "https://github.com/envoyproxy/envoy/releases/download/v1.32.3/BINARY_NAME"
#   4. Compute the hash: shasum -a 256 BINARY_NAME
#   5. Paste the hash into the appropriate variable below
ENVOY_VERSION="1.32.3"
ARCH=$(uname -m)
if [ "$ARCH" = "aarch64" ]; then
  ENVOY_ARCH="aarch_64"
  ENVOY_SHA256="REPLACE_WITH_ACTUAL_SHA256_FOR_ARM64"
else
  ENVOY_ARCH="x86_64"
  ENVOY_SHA256="REPLACE_WITH_ACTUAL_SHA256_FOR_X86_64"
fi

# Fail fast if someone forgot to replace the placeholder hash
if [[ "$ENVOY_SHA256" == *"REPLACE"* ]]; then
  echo "FATAL: Envoy SHA256 hash is still a placeholder — replace it with the real hash before deploying" >&2
  exit 1
fi

curl -fsSL -o /usr/local/bin/envoy \
  "https://github.com/envoyproxy/envoy/releases/download/v${ENVOY_VERSION}/envoy-${ENVOY_ARCH}-linux-${ENVOY_ARCH}"

# Verify binary integrity before making it executable
echo "${ENVOY_SHA256}  /usr/local/bin/envoy" | sha256sum -c -
if [ $? -ne 0 ]; then
  echo "FATAL: Envoy binary SHA256 mismatch — possible supply chain compromise" >&2
  rm -f /usr/local/bin/envoy
  exit 1
fi
chmod +x /usr/local/bin/envoy

# ── Envoy directories ──
mkdir -p /etc/envoy /var/log/envoy /var/run/envoy
chown envoy:envoy /var/log/envoy
chown envoy:envoy /var/run/envoy

# ── Agent read access to GitHub access log (for local repo sync) ──
# The GitHub access log contains only request metadata (timestamps, paths,
# response codes) — no secrets. Granting read access lets the agent user
# tail the log over SSH for event-driven local repo fetching (see section 12).
# Only the GitHub log is exposed; Anthropic and admin logs remain agent-inaccessible.
setfacl -m u:agent:x /var/log/envoy
touch /var/log/envoy/github_access.log
chown envoy:envoy /var/log/envoy/github_access.log
setfacl -m u:agent:r /var/log/envoy/github_access.log

# ── Envoy log rotation ──
# Envoy access logs get a JSON entry per API request; without rotation
# they will fill the disk. CloudWatch has the canonical immutable copy;
# local logs are kept 7 days for quick debugging.
cat > /etc/logrotate.d/envoy << 'LOGROTATE'
/var/log/envoy/*.log {
  daily
  rotate 7
  compress
  missingok
  notifempty
  copytruncate
}
LOGROTATE

# ── Shared safe_git wrapper and config poison detection (sourced by all trusted scripts) ──
# Git local config overrides global config, so a malicious repo could set
# core.hooksPath, core.fsmonitor, credential.helper, core.askPass, core.pager,
# core.editor, diff.external, core.sshCommand, url.*.insteadOf,
# url.*.pushInsteadOf, or custom filter commands in its .git/config to execute
# arbitrary code or redirect traffic during git operations in trusted scripts
# (which run outside the sandbox). The safe_git() wrapper passes per-invocation
# -c flags to override the first eight. The check_git_config_poison() function
# detects all families
# (including those that can't be blanket-overridden) before any fetch, checkout,
# or reset.
mkdir -p /usr/local/lib
cat > /usr/local/lib/safe-git.sh << 'SAFEGIT'
# Prevent git from hanging on interactive credential prompts.
# If a socat bridge dies, git fails fast instead of waiting for input.
export GIT_TERMINAL_PROMPT=0

# Restrict git to HTTP protocol only. All legitimate git operations in this
# setup use plain HTTP to localhost socat bridges (which forward to Envoy's
# unix sockets). This blocks git's ext:: transport, which allows arbitrary
# command execution via remote URLs (e.g., ext::sh -c "evil" in .gitmodules
# or crafted refs). Also blocks file://, ssh://, and git:// protocols.
# If you later add direct HTTPS remotes (bypassing the proxy), change to
# GIT_ALLOW_PROTOCOL=http:https.
export GIT_ALLOW_PROTOCOL=http

safe_git() {
  git -c core.hooksPath=/dev/null -c core.fsmonitor= \
      -c credential.helper= -c core.askPass= \
      -c core.pager=cat -c diff.external= \
      -c core.editor=true -c core.sshCommand=false "$@"
}

# Detect git config poisoning in a given directory.
# Checks for: url.*.insteadOf/pushInsteadOf, custom filter commands, and
# executable config overrides (core.pager, core.editor, diff.external, etc.).
# Usage: check_git_config_poison "$REPO_DIR" "backing repo"
check_git_config_poison() {
  local dir="$1"
  local label="${2:-repository}"
  (
    cd "$dir" || return 1

    # Detect insteadOf and pushInsteadOf — these redirect fetch/push URLs
    # to arbitrary endpoints, bypassing Envoy.
    # NOTE: git config --get-regexp compiles patterns as ERE (REG_EXTENDED),
    # so use bare () and | for grouping/alternation — not \(\) and \|.
    if safe_git config --get-regexp 'url\..*\.(insteadOf|pushInsteadOf)' 2>/dev/null | grep -q .; then
      echo "  ERROR: url.*.insteadOf or pushInsteadOf entries detected in $label config." >&2
      echo "  This could redirect git operations to unexpected endpoints." >&2
      echo "  Inspect: cd $dir && git config --list --show-origin | grep -i insteadof" >&2
      return 1
    fi

    # Detect remote.origin.pushurl — git uses this preferentially for push
    # operations, bypassing the remote.origin.url override set via GIT_CONFIG
    # env vars. Only checks the origin remote (the only one used in this setup).
    if safe_git config --get remote.origin.pushurl 2>/dev/null | grep -q .; then
      echo "  ERROR: remote.origin.pushurl detected in $label config." >&2
      echo "  This could redirect push operations to unexpected endpoints." >&2
      echo "  Inspect: cd $dir && git config --list --show-origin | grep pushurl" >&2
      return 1
    fi

    # Detect custom filter commands (clean/smudge/process) — these execute
    # during checkout, reset --hard, and clean. Allowlist git-lfs entries
    # only if the value starts with "git-lfs " — a poisoned config could
    # keep the filter.lfs.* key prefix but substitute a malicious command.
    # NOTE: grep -E for ERE consistency with the git config patterns above.
    local poison
    poison=$(safe_git config --get-regexp 'filter\..*\.(clean|smudge|process)' 2>/dev/null \
      | grep -Ev '^filter\.lfs\.(clean|smudge|process) git-lfs ' || true)
    if [ -n "$poison" ]; then
      echo "  ERROR: custom git filter commands detected in $label config." >&2
      echo "  This could execute arbitrary code during checkout/reset/clean." >&2
      echo "  Entries found:" >&2
      echo "$poison" | sed 's/^/    /' >&2
      echo "  Inspect: cd $dir && git config --list --show-origin | grep filter" >&2
      return 1
    fi

    # Detect config keys that execute arbitrary commands when triggered by
    # common git operations (log, diff, show, commit, rebase, fetch). These
    # are overridden by safe_git() via -c flags, but detection catches them
    # for contexts that might not use safe_git (e.g., if the subshell alias
    # is bypassed via command git or env -i git). Includes all keys that
    # safe_git() overrides (core.hooksPath, core.fsmonitor, core.askPass,
    # core.pager, core.editor, core.sshCommand, diff.external) plus keys
    # that only trigger in interactive contexts (diff.tool, merge.tool,
    # sequence.editor).
    # NOTE: credential.helper is intentionally omitted from detection —
    # it's a multi-valued key that may have legitimate system-level entries
    # (e.g., osxkeychain, libsecret). safe_git() clears it via -c flag
    # regardless, so the runtime override is the primary control.
    local exec_keys
    exec_keys=$(safe_git config --get-regexp \
      '(core\.hooksPath|core\.fsmonitor|core\.askPass|core\.pager|core\.editor|core\.sshCommand|diff\.external|diff\.tool|merge\.tool|sequence\.editor)' \
      2>/dev/null || true)
    if [ -n "$exec_keys" ]; then
      echo "  ERROR: executable config overrides detected in $label config." >&2
      echo "  These could execute arbitrary commands during git operations." >&2
      echo "  Entries found:" >&2
      echo "$exec_keys" | sed 's/^/    /' >&2
      echo "  Inspect: cd $dir && git config --list --show-origin" >&2
      return 1
    fi
  )
}

# Cross-validate meta/github-path against the frozen remote URL in .git/config.
# The .git/config is read-only (mode 0444), so its remote URL is a trusted
# record of the original clone target. If meta/github-path has been tampered
# with, the paths will diverge.
# Usage: validate_github_path "$REPO_DIR" "$GITHUB_PATH"
validate_github_path() {
  local repo_dir="$1"
  local meta_path="$2"
  local config_file="$repo_dir/.git/config"

  [ -f "$config_file" ] || return 0  # fresh clone, no config yet
  [ -n "$meta_path" ] || return 0    # no meta path to validate

  local frozen_url
  frozen_url=$(git config --file "$config_file" remote.origin.url 2>/dev/null || echo "")
  [ -n "$frozen_url" ] || return 0  # no remote in config

  # Extract org/repo from the frozen URL (handles http://host/org/repo.git)
  local frozen_path
  frozen_path=$(echo "$frozen_url" | grep -oP '[^/]+/[^/]+(?=\.git$)' || echo "")
  [ -n "$frozen_path" ] || return 0  # couldn't parse, skip validation

  if [ "$meta_path" != "$frozen_path" ]; then
    echo "  ERROR: meta/github-path ($meta_path) does not match" >&2
    echo "  the frozen remote URL in .git/config ($frozen_path)." >&2
    echo "  This could indicate tampering with the meta directory." >&2
    echo "  Expected: $frozen_path" >&2
    return 1
  fi
}
SAFEGIT
chmod 644 /usr/local/lib/safe-git.sh
chown root:root /usr/local/lib/safe-git.sh

# ── Subshell init file (loaded by the post-Claude subshell) ──
# The post-Claude subshell runs outside the sandbox with outbound HTTPS
# available via the NAT gateway. Without this rcfile, plain `git` would
# honor any config poisoning planted during the sandboxed Claude session
# (e.g., insteadOf/pushInsteadOf in config.worktree). Aliasing `git` to
# `safe_git` gives the user the same -c flag overrides that trusted scripts
# have, so credential.helper, core.askPass, core.hooksPath, core.sshCommand,
# and core.fsmonitor are neutralized even in the interactive subshell.
cat > /usr/local/lib/subshell-rc.sh << 'SUBSHELLRC'
source /usr/local/lib/safe-git.sh
git() { safe_git "$@"; }
export -f git
SUBSHELLRC
chmod 644 /usr/local/lib/subshell-rc.sh
chown root:root /usr/local/lib/subshell-rc.sh

# ── Secrets refresh script (generates Envoy config from Secrets Manager) ──
cat > /usr/local/bin/refresh-proxy-secrets << 'REFRESH'
#!/bin/bash
set -euo pipefail

SECRETS_JSON=$(aws secretsmanager get-secret-value \
  --secret-id "api-keys" \
  --query 'SecretString' \
  --output text 2>/dev/null)

if [ -z "$SECRETS_JSON" ]; then
  echo "ERROR: Failed to fetch secrets from Secrets Manager" >&2
  exit 1
fi

ANTHROPIC_KEY=$(echo "$SECRETS_JSON" | jq -r '.ANTHROPIC_API_KEY')
GITHUB_TOKEN=$(echo "$SECRETS_JSON" | jq -r '.GITHUB_TOKEN')

if [ -z "$ANTHROPIC_KEY" ] || [ "$ANTHROPIC_KEY" = "null" ]; then
  echo "ERROR: ANTHROPIC_API_KEY not found in secret" >&2
  exit 1
fi

if [ -z "$GITHUB_TOKEN" ] || [ "$GITHUB_TOKEN" = "null" ]; then
  echo "ERROR: GITHUB_TOKEN not found in secret" >&2
  exit 1
fi

# Validate key formats
if ! [[ "$ANTHROPIC_KEY" =~ ^sk-ant- ]]; then
  echo "ERROR: ANTHROPIC_API_KEY does not match expected format (sk-ant-...)" >&2
  exit 1
fi

if ! [[ "$GITHUB_TOKEN" =~ ^github_pat_ ]]; then
  echo "ERROR: GITHUB_TOKEN does not match expected format (github_pat_...)" >&2
  exit 1
fi

# Validate character sets to prevent YAML injection.
# Secret values are interpolated into an Envoy YAML config via bash heredoc.
# A value containing YAML metacharacters (quotes, backslashes, newlines, colons)
# could break out of the quoted string context and inject arbitrary Envoy config.
# Anthropic keys and GitHub PATs use restricted character sets in practice, but
# we enforce it explicitly so a malformed value in Secrets Manager cannot become
# a config injection vector.
if ! [[ "$ANTHROPIC_KEY" =~ ^[a-zA-Z0-9_-]+$ ]]; then
  echo "ERROR: ANTHROPIC_API_KEY contains unexpected characters (allowed: a-zA-Z0-9_-)" >&2
  exit 1
fi

if ! [[ "$GITHUB_TOKEN" =~ ^[a-zA-Z0-9_-]+$ ]]; then
  echo "ERROR: GITHUB_TOKEN contains unexpected characters (allowed: a-zA-Z0-9_-)" >&2
  exit 1
fi

# ── Optional: private npm registry token ──
# If NPM_PRIVATE_TOKEN is set in Secrets Manager, Envoy will proxy and
# authenticate requests to the private registry. If not set, the private
# registry listener is omitted from the config.
NPM_PRIVATE_TOKEN=$(echo "$SECRETS_JSON" | jq -r '.NPM_PRIVATE_TOKEN // empty')
NPM_PRIVATE_REGISTRY=$(echo "$SECRETS_JSON" | jq -r '.NPM_PRIVATE_REGISTRY // empty')

if [ -n "$NPM_PRIVATE_TOKEN" ]; then
  if ! [[ "$NPM_PRIVATE_TOKEN" =~ ^[a-zA-Z0-9_-]+$ ]]; then
    echo "ERROR: NPM_PRIVATE_TOKEN contains unexpected characters (allowed: a-zA-Z0-9_-)" >&2
    exit 1
  fi
  # Default to GitHub Packages if no registry specified
  NPM_PRIVATE_REGISTRY=${NPM_PRIVATE_REGISTRY:-npm.pkg.github.com}
  if ! [[ "$NPM_PRIVATE_REGISTRY" =~ ^[a-zA-Z0-9._-]+$ ]]; then
    echo "ERROR: NPM_PRIVATE_REGISTRY contains unexpected characters" >&2
    exit 1
  fi
fi

# Back up current working config (if it exists)
if [ -f /etc/envoy/envoy.yaml ]; then
  cp /etc/envoy/envoy.yaml /etc/envoy/envoy.yaml.bak
fi

# Write to a temp file first
TMPFILE=$(mktemp /etc/envoy/envoy.yaml.XXXXXX)

# ── Build optional private registry listener/cluster (if token exists) ──
NPM_PRIVATE_LISTENER=""
NPM_PRIVATE_CLUSTER=""
NPM_PRIVATE_ACCESS_LOG=""
if [ -n "$NPM_PRIVATE_TOKEN" ]; then
NPM_PRIVATE_LISTENER=$(cat << NPMLISTENER

  # ── Private npm registry proxy (unix socket) ──
  - name: npm_private_listener
    address:
      pipe:
        path: /var/run/envoy/npm-private.sock
        mode: 0660
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: npm_private
          codec_type: AUTO
          normalize_path: true
          merge_slashes: true
          stream_idle_timeout: 300s
          request_timeout: 300s
          access_log:
          - name: envoy.access_loggers.file
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
              path: /var/log/envoy/npm_private_access.log
              log_format:
                json_format:
                  timestamp: "%START_TIME%"
                  method: "%REQ(:METHOD)%"
                  path: "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%"
                  response_code: "%RESPONSE_CODE%"
                  bytes_sent: "%BYTES_SENT%"
                  bytes_received: "%BYTES_RECEIVED%"
                  duration_ms: "%DURATION%"
                  upstream_host: "%UPSTREAM_HOST%"
                  downstream_remote: "%DOWNSTREAM_REMOTE_ADDRESS%"
          route_config:
            name: npm_private_route
            virtual_hosts:
            - name: npm_private
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: npm_private_registry
                  timeout: 300s
                request_headers_to_remove:
                - authorization
                request_headers_to_add:
                - header:
                    key: authorization
                    value: "Bearer ${NPM_PRIVATE_TOKEN}"
                  append_action: OVERWRITE_IF_EXISTS_OR_ADD
                - header:
                    key: host
                    value: "${NPM_PRIVATE_REGISTRY}"
                  append_action: OVERWRITE_IF_EXISTS_OR_ADD
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
NPMLISTENER
)

NPM_PRIVATE_CLUSTER=$(cat << NPMCLUSTER

  # ── Private npm registry upstream ──
  - name: npm_private_registry
    type: STRICT_DNS
    dns_lookup_family: V4_ONLY
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: npm_private_registry
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: ${NPM_PRIVATE_REGISTRY}
                port_value: 443
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        sni: ${NPM_PRIVATE_REGISTRY}
        common_tls_context:
          validation_context:
            trusted_ca:
              filename: /etc/pki/tls/certs/ca-bundle.crt
NPMCLUSTER
)
fi

cat > "$TMPFILE" << ENVOY_EOF
static_resources:
  listeners:

  # ── Anthropic API proxy (unix socket) ──
  - name: anthropic_listener
    address:
      pipe:
        path: /var/run/envoy/anthropic.sock
        mode: 0660
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: anthropic
          codec_type: AUTO
          normalize_path: true
          merge_slashes: true
          stream_idle_timeout: 600s
          request_timeout: 600s
          access_log:
          - name: envoy.access_loggers.file
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
              path: /var/log/envoy/anthropic_access.log
              log_format:
                json_format:
                  timestamp: "%START_TIME%"
                  method: "%REQ(:METHOD)%"
                  path: "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%"
                  response_code: "%RESPONSE_CODE%"
                  bytes_sent: "%BYTES_SENT%"
                  bytes_received: "%BYTES_RECEIVED%"
                  duration_ms: "%DURATION%"
                  upstream_host: "%UPSTREAM_HOST%"
                  request_id: "%REQ(X-REQUEST-ID)%"
                  downstream_remote: "%DOWNSTREAM_REMOTE_ADDRESS%"
          route_config:
            name: anthropic_route
            virtual_hosts:
            - name: anthropic
              domains: ["*"]
              routes:
              - match:
                  prefix: "/v1/messages"
                route:
                  cluster: anthropic_api
                  timeout: 600s
                request_headers_to_remove:
                - x-api-key
                request_headers_to_add:
                - header:
                    key: x-api-key
                    value: "${ANTHROPIC_KEY}"
                  append_action: OVERWRITE_IF_EXISTS_OR_ADD
                - header:
                    key: host
                    value: "api.anthropic.com"
                  append_action: OVERWRITE_IF_EXISTS_OR_ADD
              # ── Deny all other Anthropic API paths ──
              - match:
                  prefix: "/"
                direct_response:
                  status: 403
                  body:
                    inline_string: "Forbidden: only /v1/messages is proxied"
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  # ── GitHub git-over-HTTP proxy (unix socket) ──
  - name: github_listener
    address:
      pipe:
        path: /var/run/envoy/github.sock
        mode: 0660
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: github
          codec_type: AUTO
          normalize_path: true
          merge_slashes: true
          stream_idle_timeout: 600s
          request_timeout: 600s
          access_log:
          - name: envoy.access_loggers.file
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
              path: /var/log/envoy/github_access.log
              log_format:
                json_format:
                  timestamp: "%START_TIME%"
                  method: "%REQ(:METHOD)%"
                  path: "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%"
                  response_code: "%RESPONSE_CODE%"
                  bytes_sent: "%BYTES_SENT%"
                  bytes_received: "%BYTES_RECEIVED%"
                  duration_ms: "%DURATION%"
                  upstream_host: "%UPSTREAM_HOST%"
                  downstream_remote: "%DOWNSTREAM_REMOTE_ADDRESS%"
          route_config:
            name: github_route
            virtual_hosts:
            - name: github
              domains: ["*"]
              routes:
              # ── Git smart HTTP: fetch/clone ──
              - match:
                  safe_regex:
                    regex: "^/[^/]+/[^/]+\\.git/info/refs$"
                route:
                  cluster: github_api
                  timeout: 600s
                request_headers_to_remove:
                - authorization
                request_headers_to_add:
                - header:
                    key: authorization
                    value: "Bearer ${GITHUB_TOKEN}"
                  append_action: OVERWRITE_IF_EXISTS_OR_ADD
                - header:
                    key: host
                    value: "github.com"
                  append_action: OVERWRITE_IF_EXISTS_OR_ADD
              - match:
                  safe_regex:
                    regex: "^/[^/]+/[^/]+\\.git/git-upload-pack$"
                route:
                  cluster: github_api
                  timeout: 600s
                request_headers_to_remove:
                - authorization
                request_headers_to_add:
                - header:
                    key: authorization
                    value: "Bearer ${GITHUB_TOKEN}"
                  append_action: OVERWRITE_IF_EXISTS_OR_ADD
                - header:
                    key: host
                    value: "github.com"
                  append_action: OVERWRITE_IF_EXISTS_OR_ADD
              # ── Git smart HTTP: push ──
              - match:
                  safe_regex:
                    regex: "^/[^/]+/[^/]+\\.git/git-receive-pack$"
                route:
                  cluster: github_api
                  timeout: 600s
                request_headers_to_remove:
                - authorization
                request_headers_to_add:
                - header:
                    key: authorization
                    value: "Bearer ${GITHUB_TOKEN}"
                  append_action: OVERWRITE_IF_EXISTS_OR_ADD
                - header:
                    key: host
                    value: "github.com"
                  append_action: OVERWRITE_IF_EXISTS_OR_ADD
              # ── Deny everything else (API calls, webhooks, etc.) ──
              # The three routes above cover all git smart-HTTP operations.
              # Older git versions occasionally request bare repo.git/ paths
              # expecting a redirect; those will get a 403 here. If you see
              # clone failures in Envoy logs with 403 on a .git/ path, add
              # a route for that specific path — but prefer failing closed.
              - match:
                  prefix: "/"
                direct_response:
                  status: 403
                  body:
                    inline_string: "Forbidden: only git smart-HTTP operations are proxied"
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
${NPM_PRIVATE_LISTENER}

  clusters:

  # ── Anthropic upstream ──
  - name: anthropic_api
    type: STRICT_DNS
    dns_lookup_family: V4_ONLY
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: anthropic_api
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: api.anthropic.com
                port_value: 443
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        sni: api.anthropic.com
        common_tls_context:
          validation_context:
            trusted_ca:
              filename: /etc/pki/tls/certs/ca-bundle.crt

  # ── GitHub upstream ──
  - name: github_api
    type: STRICT_DNS
    dns_lookup_family: V4_ONLY
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: github_api
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: github.com
                port_value: 443
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        sni: github.com
        common_tls_context:
          validation_context:
            trusted_ca:
              filename: /etc/pki/tls/certs/ca-bundle.crt
${NPM_PRIVATE_CLUSTER}

# ── Admin interface on unix socket (root-only access) ──
admin:
  address:
    pipe:
      path: /var/run/envoy/admin.sock
      mode: 0600
  access_log:
  - name: envoy.access_loggers.file
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
      path: /var/log/envoy/admin_access.log
ENVOY_EOF

# Config readable only by envoy user
chmod 600 "$TMPFILE"
chown envoy:envoy "$TMPFILE"

# Validate config before applying
if /usr/local/bin/envoy --mode validate -c "$TMPFILE" 2>/dev/null; then
  mv "$TMPFILE" /etc/envoy/envoy.yaml
  echo "Proxy secrets refreshed at $(date)"
else
  echo "ERROR: Generated Envoy config failed validation" >&2
  rm -f "$TMPFILE"
  # Restore backup if available
  if [ -f /etc/envoy/envoy.yaml.bak ]; then
    cp /etc/envoy/envoy.yaml.bak /etc/envoy/envoy.yaml
    echo "Restored previous working config from backup" >&2
  fi
  exit 1
fi
REFRESH

chmod 755 /usr/local/bin/refresh-proxy-secrets
chown root:root /usr/local/bin/refresh-proxy-secrets

# ── Envoy systemd service ──
cat > /etc/systemd/system/envoy.service << 'ENVOY_SVC'
[Unit]
Description=Envoy Credential Proxy
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=envoy
Group=envoy
ExecStartPre=/usr/local/bin/refresh-proxy-secrets
ExecStart=/usr/local/bin/envoy -c /etc/envoy/envoy.yaml --log-level warn --log-path /var/log/envoy/envoy.log
Restart=always
RestartSec=5
LimitNOFILE=65536
LimitCORE=0

# Hardening
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/log/envoy /var/run/envoy
PrivateTmp=true
CapabilityBoundingSet=
AmbientCapabilities=

[Install]
WantedBy=multi-user.target
ENVOY_SVC

# ── ExecStartPre runs as root (before dropping to envoy user), ──
# ── so refresh-proxy-secrets can call aws CLI and write the config. ──
# ── Override the User for ExecStartPre only: ──
mkdir -p /etc/systemd/system/envoy.service.d
cat > /etc/systemd/system/envoy.service.d/override.conf << 'OVERRIDE'
[Service]
ExecStartPre=
ExecStartPre=+/usr/local/bin/refresh-proxy-secrets
OVERRIDE

systemctl daemon-reload
systemctl enable envoy

# ── The `code` command — the daily driver ──
cat > /usr/local/bin/code << 'CODE_SCRIPT'
#!/bin/bash
set -euo pipefail

source /usr/local/lib/safe-git.sh

SLOTS_DIR="$HOME/workspaces/slots"
REPOS_DIR="$HOME/workspaces/repos"
META_DIR="$HOME/workspaces/meta"
LOCKS_DIR="$HOME/.locks"
SESSION_LOG="$HOME/logs/sessions.jsonl"

mkdir -p "$SLOTS_DIR" "$REPOS_DIR" "$LOCKS_DIR" "$(dirname "$SESSION_LOG")"

ENVOY_ANTHROPIC_SOCK="/var/run/envoy/anthropic.sock"
ENVOY_GITHUB_SOCK="/var/run/envoy/github.sock"
ENVOY_NPM_PRIVATE_SOCK="/var/run/envoy/npm-private.sock"

# ── Verify Envoy sockets exist and are connectable ──
if [ ! -S "$ENVOY_ANTHROPIC_SOCK" ]; then
  echo "  ERROR: Envoy Anthropic socket not found at $ENVOY_ANTHROPIC_SOCK" >&2
  echo "  Ask an admin to check: sudo systemctl status envoy" >&2
  exit 1
fi

if ! curl -sf --unix-socket "$ENVOY_ANTHROPIC_SOCK" http://localhost/ -o /dev/null 2>/dev/null; then
  # Socket exists but might just be rejecting the path (which is expected: 403)
  # Verify we can at least connect
  if ! socat -u OPEN:/dev/null UNIX-CONNECT:"$ENVOY_ANTHROPIC_SOCK" 2>/dev/null; then
    echo "  ERROR: Cannot connect to Envoy Anthropic socket." >&2
    echo "  Ask an admin to check: sudo systemctl status envoy" >&2
    exit 1
  fi
fi

if [ ! -S "$ENVOY_GITHUB_SOCK" ]; then
  echo "  ERROR: Envoy GitHub socket not found at $ENVOY_GITHUB_SOCK" >&2
  echo "  Ask an admin to check: sudo systemctl status envoy" >&2
  exit 1
fi

if ! socat -u OPEN:/dev/null UNIX-CONNECT:"$ENVOY_GITHUB_SOCK" 2>/dev/null; then
  echo "  ERROR: Cannot connect to Envoy GitHub socket." >&2
  echo "  Ask an admin to check: sudo systemctl status envoy" >&2
  exit 1
fi

# ── Check for optional private registry proxy ──
HAS_NPM_PRIVATE=false
if [ -S "$ENVOY_NPM_PRIVATE_SOCK" ]; then
  HAS_NPM_PRIVATE=true
fi

# ── Verify sandbox-runtime is installed ──
if ! command -v srt &> /dev/null; then
  echo "  ERROR: sandbox-runtime (srt) is not installed." >&2
  echo "  Run: npm install -g @anthropic-ai/sandbox-runtime" >&2
  exit 1
fi

# ── Find an available slot (atomic via flock) ──
claim_slot() {
  for i in $(seq 1 6); do
    local lockfile="$LOCKS_DIR/slot-$i.lock"
    eval "exec 200>\"$lockfile\""
    if flock -n 200; then
      # Lock acquired — fd 200 stays open for the session lifetime.
      # The lock is automatically released when the shell exits.
      echo "$i"
      return 0
    fi
    exec 200>&-  # Close fd, try the next slot
  done
  echo ""
  return 1
}

# ── Check if a slot is currently locked ──
is_slot_active() {
  local slot=$1
  local lockfile="$LOCKS_DIR/slot-$slot.lock"
  [ -f "$lockfile" ] || return 1
  # Try to acquire lock in a subshell (doesn't affect our fd table)
  if (flock -n 200 || exit 1) 200<>"$lockfile" 2>/dev/null; then
    return 1  # Lock was available — slot is NOT active
  fi
  return 0  # Couldn't acquire — slot IS active
}

# ── Pick a repo ──
pick_repo() {
  local repos=()
  for dir in "$REPOS_DIR"/*/; do
    [ -d "$dir" ] || continue
    repos+=("$(basename "$dir")")
  done

  if [ ${#repos[@]} -eq 0 ]; then
    echo "No repos available. Add one first:" >&2
    echo "  add-repo yourorg/repo-name" >&2
    return 1
  fi

  if [ ${#repos[@]} -eq 1 ]; then
    echo "${repos[0]}"
    return 0
  fi

  echo "" >&2
  echo "  Pick a repo:" >&2
  echo "" >&2
  for i in "${!repos[@]}"; do
    printf "    %d) %s\n" "$((i + 1))" "${repos[$i]}" >&2
  done
  echo "" >&2

  local choice
  while true; do
    read -rp "  > " choice
    if [[ "$choice" =~ ^[0-9]+$ ]] && [ "$choice" -ge 1 ] && [ "$choice" -le "${#repos[@]}" ]; then
      echo "${repos[$((choice - 1))]}"
      return 0
    fi
    for r in "${repos[@]}"; do
      if [ "$r" = "$choice" ]; then
        echo "$r"
        return 0
      fi
    done
    echo "  Invalid choice. Enter a number or repo name." >&2
  done
}

# ── Start socat bridges for a given slot ──
start_bridges() {
  local slot=$1
  local api_port=$((8080 + slot))   # 8081-8086
  local git_port=$((8090 + slot))   # 8091-8096
  local npm_port=$((8100 + slot))   # 8101-8106

  # Kill any orphaned socat processes from a previous session that exited
  # abnormally (SIGKILL, OOM kill) without running the trap handler.
  # The flock on the slot is already held by us, so any existing listener
  # on these ports is definitionally orphaned residue.
  fuser -k ${api_port}/tcp 2>/dev/null || true
  fuser -k ${git_port}/tcp 2>/dev/null || true
  fuser -k ${npm_port}/tcp 2>/dev/null || true

  socat TCP-LISTEN:${api_port},bind=127.0.0.1,reuseaddr,fork \
    UNIX-CONNECT:"$ENVOY_ANTHROPIC_SOCK" &
  SOCAT_API_PID=$!

  socat TCP-LISTEN:${git_port},bind=127.0.0.1,reuseaddr,fork \
    UNIX-CONNECT:"$ENVOY_GITHUB_SOCK" &
  SOCAT_GIT_PID=$!

  # Optional: private npm registry bridge
  SOCAT_NPM_PID=""
  if $HAS_NPM_PRIVATE; then
    socat TCP-LISTEN:${npm_port},bind=127.0.0.1,reuseaddr,fork \
      UNIX-CONNECT:"$ENVOY_NPM_PRIVATE_SOCK" &
    SOCAT_NPM_PID=$!
  fi

  # Wait briefly for bridges to bind
  sleep 0.3

  # Verify bridges are listening
  if ! kill -0 "$SOCAT_API_PID" 2>/dev/null; then
    echo "  ERROR: Failed to start Anthropic API bridge on port $api_port" >&2
    kill "$SOCAT_GIT_PID" 2>/dev/null
    [ -n "$SOCAT_NPM_PID" ] && kill "$SOCAT_NPM_PID" 2>/dev/null
    return 1
  fi
  if ! kill -0 "$SOCAT_GIT_PID" 2>/dev/null; then
    echo "  ERROR: Failed to start GitHub bridge on port $git_port" >&2
    kill "$SOCAT_API_PID" 2>/dev/null
    [ -n "$SOCAT_NPM_PID" ] && kill "$SOCAT_NPM_PID" 2>/dev/null
    return 1
  fi
  if [ -n "$SOCAT_NPM_PID" ] && ! kill -0 "$SOCAT_NPM_PID" 2>/dev/null; then
    echo "  WARNING: Failed to start npm private registry bridge on port $npm_port" >&2
    SOCAT_NPM_PID=""
    HAS_NPM_PRIVATE=false
  fi

  echo "$api_port"
}

# ── Show status ──
show_status() {
  echo ""

  local repo_list
  repo_list=$(ls "$REPOS_DIR" 2>/dev/null | tr '\n' ' ')
  if [ -n "$repo_list" ]; then
    echo "  Repos: $repo_list"
    echo ""
  fi

  local active=0
  for i in $(seq 1 6); do
    local slot_dir="$SLOTS_DIR/slot-$i"

    if is_slot_active "$i"; then
      local repo branch
      repo=$(cd "$slot_dir" 2>/dev/null && safe_git remote get-url origin 2>/dev/null | grep -oP '[^/]+(?=\.git$)' || echo "?")
      branch=$(cd "$slot_dir" 2>/dev/null && safe_git branch --show-current 2>/dev/null || echo "?")
      printf "  slot %d: \033[32mactive\033[0m  [%s] %s  (ports %d/%d)\n" "$i" "$repo" "$branch" "$((8080+i))" "$((8090+i))"
      active=$((active + 1))
    elif [ -d "$slot_dir" ]; then
      local repo dirty=""
      repo=$(cd "$slot_dir" 2>/dev/null && safe_git remote get-url origin 2>/dev/null | grep -oP '[^/]+(?=\.git$)' || echo "?")
      if [ -n "$(cd "$slot_dir" && safe_git status --porcelain 2>/dev/null)" ]; then
        dirty=" (uncommitted changes)"
      fi
      printf "  slot %d: \033[90midle    [%s]%s\033[0m\n" "$i" "$repo" "$dirty"
    else
      printf "  slot %d: \033[90mempty\033[0m\n" "$i"
    fi
  done

  echo ""
  echo "  $active / 6 active"
  echo ""
}

# ── Commands ──
case "${1:-}" in
  status|-s)
    show_status
    exit 0
    ;;
  kill)
    KILL_SLOT="${2:?"Usage: code kill <slot-number>"}"
    if ! [[ "$KILL_SLOT" =~ ^[1-6]$ ]]; then
      echo "  Invalid slot number. Must be 1-6." >&2
      exit 1
    fi
    # Use flock as the source of truth — not PID existence, which is
    # vulnerable to PID reuse if the original session died and the PID
    # was recycled by an unrelated process.
    if ! is_slot_active "$KILL_SLOT"; then
      echo "  Slot $KILL_SLOT is not active." >&2
      exit 1
    fi
    pid=$(cat "$LOCKS_DIR/slot-$KILL_SLOT.lock" 2>/dev/null)
    if [ -z "$pid" ]; then
      echo "  Slot $KILL_SLOT lock file has no PID." >&2
      exit 1
    fi
    kill "$pid"
    echo "  Sent SIGTERM to slot $KILL_SLOT (PID $pid)."
    echo "  The session's trap handler will clean up bridges and release the slot."
    exit 0
    ;;
  sync)
    # Start a temporary git socat bridge for fetching
    SYNC_GIT_PORT=$(python3 -c 'import socket; s=socket.socket(); s.bind(("",0)); print(s.getsockname()[1]); s.close()')
    socat TCP-LISTEN:${SYNC_GIT_PORT},bind=127.0.0.1,reuseaddr,fork \
      UNIX-CONNECT:"$ENVOY_GITHUB_SOCK" &
    SYNC_SOCAT_PID=$!
    sleep 0.3
    if ! kill -0 "$SYNC_SOCAT_PID" 2>/dev/null; then
      echo "  ERROR: sync bridge failed to start. Is Envoy running?" >&2
      echo "  Check: ls -la $ENVOY_GITHUB_SOCK" >&2
      exit 1
    fi
    trap "kill $SYNC_SOCAT_PID 2>/dev/null; wait $SYNC_SOCAT_PID 2>/dev/null" EXIT

    for repo_dir in "$REPOS_DIR"/*/; do
      [ -d "$repo_dir" ] || continue
      cd "$repo_dir"
      repo_name=$(basename "$repo_dir")
      # Check for config poisoning before fetching
      if ! check_git_config_poison "$repo_dir" "backing repo ($repo_name)"; then
        echo "SKIPPED $repo_name (config poisoning detected)"
        echo "{\"event\":\"config_poison_detected\",\"repo\":\"$repo_name\",\"dir\":\"$repo_dir\",\"source\":\"code-sync\",\"time\":\"$(date -Iseconds)\"}" \
          >> "$SESSION_LOG"
        continue
      fi
      GITHUB_PATH=$(cat "$META_DIR/$repo_name/github-path" 2>/dev/null || echo "")
      if [ -n "$GITHUB_PATH" ]; then
        # Cross-validate against frozen .git/config to detect meta/ tampering
        if ! validate_github_path "$repo_dir" "$GITHUB_PATH"; then
          echo "SKIPPED $repo_name (meta/github-path mismatch)"
          echo "{\"event\":\"config_poison_meta_mismatch\",\"repo\":\"$repo_name\",\"dir\":\"$repo_dir\",\"source\":\"code-sync\",\"time\":\"$(date -Iseconds)\"}" \
            >> "$SESSION_LOG"
          continue
        fi
        # Pass URL via -c flag instead of remote set-url, because
        # .git/config is read-only (mode 0444) to prevent config poisoning.
        safe_git -c "remote.origin.url=http://127.0.0.1:${SYNC_GIT_PORT}/${GITHUB_PATH}.git" fetch origin
      else
        safe_git fetch origin
      fi
      echo "Fetched $repo_name"
    done
    for i in $(seq 1 6); do
      slot_dir="$SLOTS_DIR/slot-$i"
      [ -d "$slot_dir" ] || continue
      if is_slot_active "$i"; then
        echo "Slot $i: SKIPPED (active)"
        continue
      fi
      # Clean up stale status file from crashed sessions
      rm -f "$HOME/shared/status/slot-$i.json"
      cd "$slot_dir"
      if [ -n "$(safe_git status --porcelain 2>/dev/null)" ]; then
        echo "Slot $i: SKIPPED (uncommitted changes)"
        continue
      fi
      # Check for config poisoning before checkout/reset/clean (these can
      # trigger custom filter commands outside the sandbox)
      if ! check_git_config_poison "$slot_dir" "slot-$i"; then
        echo "Slot $i: SKIPPED (config poisoning detected)"
        echo "{\"event\":\"config_poison_detected\",\"slot\":$i,\"dir\":\"$slot_dir\",\"source\":\"code-sync\",\"time\":\"$(date -Iseconds)\"}" \
          >> "$SESSION_LOG"
        continue
      fi
      # Determine the default branch for this slot's repo
      slot_repo=$(safe_git remote get-url origin 2>/dev/null | grep -oP '[^/]+(?=\.git$)' || echo "")
      slot_default=$(cat "$META_DIR/$slot_repo/default-branch" 2>/dev/null || echo "main")
      if ! [[ "$slot_default" =~ ^[a-zA-Z0-9/._-]+$ ]]; then
        echo "Slot $i: SKIPPED (invalid default-branch in meta)"
        continue
      fi
      safe_git checkout "$slot_default" 2>/dev/null
      safe_git reset --hard "origin/$slot_default" 2>/dev/null
      safe_git clean -fdx 2>/dev/null
      echo "Slot $i: synced"
    done
    exit 0
    ;;
  help|-h|--help)
    echo "Usage:"
    echo "  code              launch claude code on a repo"
    echo "  code status       show all slots"
    echo "  code kill <slot>  stop a running session"
    echo "  code sync         fetch and reset idle slots"
    echo "  add-repo org/repo add a repo to the pool"
    exit 0
    ;;
esac

# ── Interactive launch ──
show_status

SLOT=$(claim_slot)
if [ -z "$SLOT" ]; then
  echo "  All 6 slots are in use."
  exit 1
fi

REPO_NAME=$(pick_repo) || exit 1
REPO_DIR="$REPOS_DIR/$REPO_NAME"

# ── Detect the repo's default branch (main, master, or other) ──
REPO_DEFAULT_BRANCH=$(cat "$META_DIR/$REPO_NAME/default-branch" 2>/dev/null || echo "main")
if ! [[ "$REPO_DEFAULT_BRANCH" =~ ^[a-zA-Z0-9/._-]+$ ]]; then
  echo "  ERROR: invalid default-branch value in meta: $REPO_DEFAULT_BRANCH" >&2
  echo "  Inspect: cat $META_DIR/$REPO_NAME/default-branch" >&2
  exit 1
fi

SUGGESTED_BRANCH="agent/$REPO_NAME-$(date +%m%d-%H%M%S)"
read -rp "  Branch [$SUGGESTED_BRANCH]: " BRANCH
BRANCH=${BRANCH:-$SUGGESTED_BRANCH}

if ! [[ "$BRANCH" =~ ^[a-zA-Z0-9/._-]+$ ]]; then
  echo "  Invalid branch name." >&2
  exit 1
fi

# ── Warn if another active slot is already on this repo+branch ──
for status_file in "$HOME/shared/status"/slot-*.json; do
  [ -f "$status_file" ] || continue
  slot_num=$(basename "$status_file" | grep -oP '\d+')
  # Skip stale status files from crashed sessions (flock released but trap didn't fire)
  if ! is_slot_active "$slot_num"; then
    rm -f "$status_file"
    continue
  fi
  existing_repo=$(jq -r .repo "$status_file" 2>/dev/null)
  existing_branch=$(jq -r .branch "$status_file" 2>/dev/null)
  if [ "$existing_repo" = "$REPO_NAME" ] && [ "$existing_branch" = "$BRANCH" ]; then
    echo "" >&2
    echo "  WARNING: Slot $slot_num is already working on $REPO_NAME @ $BRANCH." >&2
    echo "  Concurrent work on the same branch will cause push conflicts." >&2
    read -rp "  Continue anyway? [y/N] " CONFIRM
    [[ "$CONFIRM" =~ ^[Yy]$ ]] || exit 1
    break
  fi
done

SLOT_DIR="$SLOTS_DIR/slot-$SLOT"

# ── Handle worktree lifecycle ──
if [ -d "$SLOT_DIR" ]; then
  CURRENT_REPO=$(cd "$SLOT_DIR" && safe_git remote get-url origin 2>/dev/null | grep -oP '[^/]+(?=\.git$)' || echo "")

  if [ "$CURRENT_REPO" != "$REPO_NAME" ]; then
    if [ -n "$(cd "$SLOT_DIR" && safe_git status --porcelain 2>/dev/null)" ]; then
      echo "  Slot $SLOT has uncommitted changes from $CURRENT_REPO." >&2
      echo "  Commit or discard first: cd ~/workspaces/slots/slot-$SLOT" >&2
      exit 1
    fi
    echo "  Switching slot $SLOT from $CURRENT_REPO to $REPO_NAME..."
    CURRENT_MAIN="$REPOS_DIR/$CURRENT_REPO"
    if [ -d "$CURRENT_MAIN" ]; then
      cd "$CURRENT_MAIN" && safe_git worktree remove "$SLOT_DIR" --force 2>/dev/null
    else
      rm -rf "$SLOT_DIR"
    fi
  else
    # Same repo — check for uncommitted changes
    if [ -n "$(cd "$SLOT_DIR" && safe_git status --porcelain 2>/dev/null)" ]; then
      echo "  Slot $SLOT has uncommitted changes." >&2
      echo "  Commit or discard first: cd ~/workspaces/slots/slot-$SLOT" >&2
      exit 1
    fi
  fi
fi

# ── Start per-slot socat bridges (Envoy unix socket → localhost TCP) ──
API_PORT=$(start_bridges "$SLOT") || exit 1
GIT_PORT=$((8090 + SLOT))

# ── Override remote URL via environment (no mutation of shared .git/config) ──
# Worktrees share the backing repo's .git/config. If two slots work on the
# same repo, `git remote set-url` would race — the last write wins, and when
# one session ends and kills its bridge, the other's git operations break.
# GIT_CONFIG env vars have the highest precedence and are scoped to this
# process tree, avoiding the race entirely.
GITHUB_PATH=$(cat "$META_DIR/$REPO_NAME/github-path" 2>/dev/null || echo "")
if [ -n "$GITHUB_PATH" ]; then
  # Cross-validate against the frozen remote URL in .git/config to detect
  # meta/ directory tampering (e.g., redirecting to attacker-controlled repo).
  validate_github_path "$REPO_DIR" "$GITHUB_PATH" || exit 1
  export GIT_CONFIG_COUNT=2
  export GIT_CONFIG_KEY_0="remote.origin.url"
  export GIT_CONFIG_VALUE_0="http://127.0.0.1:${GIT_PORT}/${GITHUB_PATH}.git"
  # Override pushurl too — git uses it preferentially for push operations.
  # Without this, a worktree-level remote.origin.pushurl could redirect
  # pushes to an attacker-controlled endpoint even though url is overridden.
  export GIT_CONFIG_KEY_1="remote.origin.pushurl"
  export GIT_CONFIG_VALUE_1="http://127.0.0.1:${GIT_PORT}/${GITHUB_PATH}.git"
fi

# ── Check backing repo config for poisoning BEFORE any fetch ──
# A previous sandboxed session could have written insteadOf, pushInsteadOf,
# or custom filter commands to the backing repo's .git/config. The fetch
# must not run until these are ruled out — insteadOf rewrites take effect
# during fetch, and filter commands execute during checkout/reset/clean.
check_git_config_poison "$REPO_DIR" "backing repo" || exit 1

# ── Create worktree if needed ──
if [ ! -d "$SLOT_DIR" ]; then
  cd "$REPO_DIR"
  safe_git fetch origin
  safe_git worktree add "$SLOT_DIR" "origin/$REPO_DEFAULT_BRANCH" 2>/dev/null
fi

cd "$SLOT_DIR"

# ── Proactively clean worktree config from previous sessions ──
# No legitimate worktree-specific config is needed in this setup (remote URL
# is overridden via env vars, identity is in the immutable global gitconfig).
# Deleting prevents crash-recovery scenarios where a previous session's
# poisoned config.worktree blocks the next session on this slot (e.g., after
# SIGKILL or OOM kill where the trap handler never fired).
_wt_gitdir=$(safe_git rev-parse --git-dir 2>/dev/null || true)
[ -n "$_wt_gitdir" ] && rm -f "$_wt_gitdir/config.worktree"

# ── Second-pass check on worktree-level config (safety net) ──
# The proactive delete above handles the normal crash-recovery case. This
# check catches anything the delete somehow missed (e.g., a config.worktree
# recreated between the delete and this point).
check_git_config_poison "$SLOT_DIR" "worktree" || exit 1

safe_git fetch origin

# ── Remove untracked files from previous sessions ──
# A previous sandboxed Claude session could have planted convention files
# (CLAUDE.md, .env, .npmrc, Makefile, etc.) that auto-load and influence
# the next session's behavior. git checkout -B resets tracked files but
# leaves untracked files intact. This is the same contamination vector
# that `code sync` addresses — applied here so every session starts from
# a pristine worktree regardless of whether `code sync` was run manually.
# For fresh worktrees (just created via worktree add), this is a no-op.
# Cold-start penalty for reinstalling dependencies is the accepted tradeoff.
safe_git clean -fdx 2>/dev/null

# ── Check out the requested branch ──
# If the branch already exists locally with unpushed commits, offer to resume
# rather than silently resetting to the default branch (which would orphan
# committed but unpushed work).
if safe_git rev-parse --verify "$BRANCH" &>/dev/null; then
  UNPUSHED=$(safe_git log --oneline "$BRANCH" --not "origin/$REPO_DEFAULT_BRANCH" 2>/dev/null | head -5)
  if [ -n "$UNPUSHED" ]; then
    echo ""
    echo "  Branch $BRANCH exists with unpushed commits:"
    echo "$UNPUSHED" | sed 's/^/    /'
    echo ""
    read -rp "  Resume this branch? [Y/n] " RESUME
    if [[ "${RESUME:-Y}" =~ ^[Yy]$ ]]; then
      safe_git checkout "$BRANCH"
    else
      echo "  Pick a different branch name." >&2
      exit 1
    fi
  else
    # Branch exists but has no unpushed commits — safe to reset
    safe_git checkout -B "$BRANCH" "origin/$REPO_DEFAULT_BRANCH"
  fi
else
  # New branch — create from default
  safe_git checkout -B "$BRANCH" "origin/$REPO_DEFAULT_BRANCH"
fi

# ── Point Claude Code at the per-slot Envoy bridge (no raw API key needed) ──
export ANTHROPIC_BASE_URL="http://127.0.0.1:${API_PORT}"
# Format-passing dummy key — Envoy strips this and injects the real key.
# Uses sk-ant- prefix so Claude Code's key format validation (if any) passes.
export ANTHROPIC_API_KEY="sk-ant-proxy00-000000000000000000000000000000000000000000000000"

# ── Ensure git doesn't hang on credential prompts inside the sandbox ──
# Already exported by safe-git.sh, but made explicit here so it's clearly
# inherited by Claude Code and all sandboxed child processes.
export GIT_TERMINAL_PROMPT=0

# ── Write PID to lock file (for status display and kill command) ──
echo $$ > "$LOCKS_DIR/slot-$SLOT.lock"

# ── Write status for other agents ──
mkdir -p "$HOME/shared/status"
echo "{\"repo\":\"$REPO_NAME\",\"branch\":\"$BRANCH\",\"started\":\"$(date -Iseconds)\"}" \
  > "$HOME/shared/status/slot-$SLOT.json"

# ── Structured session log (for correlating with Envoy access logs) ──
echo "{\"event\":\"start\",\"slot\":$SLOT,\"api_port\":$API_PORT,\"git_port\":$GIT_PORT,\"repo\":\"$REPO_NAME\",\"branch\":\"$BRANCH\",\"pid\":$$,\"time\":\"$(date -Iseconds)\"}" \
  >> "$SESSION_LOG"

CLAUDE_PID=""
SUBSHELL_PID=""

# Send SIGTERM, wait up to $grace seconds, then SIGKILL if still alive.
# Prevents cleanup from hanging if claude or the subshell ignore SIGTERM.
kill_with_timeout() {
  local pid=$1 grace=${2:-5}
  kill "$pid" 2>/dev/null || return 0
  for i in $(seq 1 "$grace"); do
    kill -0 "$pid" 2>/dev/null || return 0
    sleep 1
  done
  kill -9 "$pid" 2>/dev/null
  wait "$pid" 2>/dev/null
}

cleanup() {
  echo "{\"event\":\"stop\",\"slot\":$SLOT,\"api_port\":$API_PORT,\"git_port\":$GIT_PORT,\"repo\":\"$REPO_NAME\",\"branch\":\"$BRANCH\",\"pid\":$$,\"time\":\"$(date -Iseconds)\"}" \
    >> "$SESSION_LOG"
  rm -f "$HOME/shared/status/slot-$SLOT.json"
  # Kill claude and subshell first — they depend on the bridges.
  # Killing bridges first would leave claude hanging on pending requests.
  # Use kill_with_timeout to escalate to SIGKILL if they ignore SIGTERM,
  # preventing permanently stuck slots.
  [ -n "$CLAUDE_PID" ] && kill_with_timeout "$CLAUDE_PID" 5
  [ -n "$SUBSHELL_PID" ] && kill_with_timeout "$SUBSHELL_PID" 3
  kill "$SOCAT_API_PID" 2>/dev/null
  kill "$SOCAT_GIT_PID" 2>/dev/null
  [ -n "$SOCAT_NPM_PID" ] && kill "$SOCAT_NPM_PID" 2>/dev/null
  wait "$SOCAT_API_PID" 2>/dev/null
  wait "$SOCAT_GIT_PID" 2>/dev/null
  [ -n "$SOCAT_NPM_PID" ] && wait "$SOCAT_NPM_PID" 2>/dev/null
  # fd 200 (flock) is released automatically when the shell exits
}
trap cleanup EXIT

NPM_PORT=$((8100 + SLOT))

echo ""
echo "  Slot $SLOT | $REPO_NAME | $BRANCH | sandbox: on"
echo "  API bridge: localhost:$API_PORT | Git bridge: localhost:$GIT_PORT"
if $HAS_NPM_PRIVATE; then
  # Export the bridge URL so .npmrc can reference it for private scopes.
  # npm supports env var interpolation in .npmrc via ${VAR} syntax.
  #
  # Add the following to your repo's .npmrc (committed or local):
  #
  #   @yourorg:registry=${NPM_PRIVATE_REGISTRY_URL}
  #   //127.0.0.1:8101/:_authToken=dummy
  #   //127.0.0.1:8102/:_authToken=dummy
  #   //127.0.0.1:8103/:_authToken=dummy
  #   //127.0.0.1:8104/:_authToken=dummy
  #   //127.0.0.1:8105/:_authToken=dummy
  #   //127.0.0.1:8106/:_authToken=dummy
  #
  # The _authToken lines are needed because npm requires auth config per
  # registry URL. The value doesn't matter ("dummy" is fine) — Envoy strips
  # the inbound Authorization header and injects the real token from Secrets
  # Manager. All six ports are listed so the same .npmrc works regardless of
  # which slot the session lands in.
  export NPM_PRIVATE_REGISTRY_URL="http://127.0.0.1:${NPM_PORT}"
  echo "  Private registry bridge: localhost:$NPM_PORT"
fi
echo "  Ctrl-C or /exit to stop"
echo ""

# ── Run Claude Code (sandbox enforced via ~/.claude/settings.json) ──
# Backgrounded with `wait` so SIGTERM from `code kill` interrupts the wait
# immediately and runs the EXIT trap, rather than being deferred until
# claude exits on its own. The trap kills claude by PID.
claude &
CLAUDE_PID=$!
wait $CLAUDE_PID 2>/dev/null
CLAUDE_PID=""

echo ""
echo "  Claude exited. Bridges still active."
echo "  Review, commit, and push. Ctrl-D to release slot."
echo ""

# ── Re-check for config poisoning planted during the sandboxed session ──
# The sandboxed Claude session can write to .git/worktrees/slot-N/config.worktree.
# Inside the sandbox this is inert (only 127.0.0.1 is reachable), but the
# subshell below runs outside the sandbox with outbound HTTPS via the NAT
# gateway. A planted pushInsteadOf could redirect `git push` to an attacker-
# controlled endpoint, exfiltrating code.
if ! check_git_config_poison "$SLOT_DIR" "worktree"; then
  echo "  WARNING: Config poisoning detected in worktree after Claude session." >&2
  echo "  Do NOT run git commands. Inspect: git config --list --show-origin" >&2
  echo "  Releasing slot without subshell." >&2
  exit 1
fi

# Drop into a subshell with live bridges so the user can push.
# Runs in background with `wait` so SIGTERM from `code kill` interrupts
# immediately rather than being deferred until the subshell exits.
# The init-file sources safe-git.sh and aliases `git` to `safe_git`, giving
# the user the same -c flag overrides that trusted scripts have. This
# neutralizes credential.helper, core.askPass, core.hooksPath, core.sshCommand,
# and core.fsmonitor even if a new poisoning vector bypasses the re-check above.
cd "$SLOT_DIR"
PS1="(slot-$SLOT) \w → " bash --init-file /usr/local/lib/subshell-rc.sh --noprofile &
SUBSHELL_PID=$!
wait $SUBSHELL_PID 2>/dev/null
SUBSHELL_PID=""

echo ""
echo "  Slot $SLOT released."
CODE_SCRIPT

chmod +x /usr/local/bin/code

# ── add-repo command (uses temporary socat bridge to Envoy unix socket) ──
cat > /usr/local/bin/add-repo << 'ADDREPO'
#!/bin/bash
set -euo pipefail

source /usr/local/lib/safe-git.sh

REPO=${1:?"Usage: add-repo <org/repo>"}

if ! [[ "$REPO" =~ ^[a-zA-Z0-9._-]+/[a-zA-Z0-9._-]+$ ]]; then
  echo "  Invalid repo format. Expected: org/repo (e.g., yourorg/frontend)" >&2
  exit 1
fi

REPO_NAME=$(echo "$REPO" | grep -oP '[^/]+$')
REPOS_DIR="$HOME/workspaces/repos"
META_DIR="$HOME/workspaces/meta"
ENVOY_GITHUB_SOCK="/var/run/envoy/github.sock"

if [ -d "$REPOS_DIR/$REPO_NAME" ]; then
  echo "Repo $REPO_NAME already exists. Fetching latest..."
  # Start temporary bridge for fetch
  GIT_PORT=$(python3 -c 'import socket; s=socket.socket(); s.bind(("",0)); print(s.getsockname()[1]); s.close()')
  socat TCP-LISTEN:${GIT_PORT},bind=127.0.0.1,reuseaddr,fork \
    UNIX-CONNECT:"$ENVOY_GITHUB_SOCK" &
  SOCAT_PID=$!
  sleep 0.3
  trap "kill $SOCAT_PID 2>/dev/null; wait $SOCAT_PID 2>/dev/null" EXIT
  cd "$REPOS_DIR/$REPO_NAME"
  # Check for config poisoning before fetching
  check_git_config_poison "$REPOS_DIR/$REPO_NAME" "backing repo" || exit 1
  # Pass URL via -c flag instead of remote set-url, because
  # .git/config is read-only (mode 0444) to prevent config poisoning.
  safe_git -c "remote.origin.url=http://127.0.0.1:${GIT_PORT}/$REPO.git" fetch origin
  exit 0
fi

mkdir -p "$REPOS_DIR"

# Start temporary socat bridge for clone
GIT_PORT=$(python3 -c 'import socket; s=socket.socket(); s.bind(("",0)); print(s.getsockname()[1]); s.close()')
socat TCP-LISTEN:${GIT_PORT},bind=127.0.0.1,reuseaddr,fork \
  UNIX-CONNECT:"$ENVOY_GITHUB_SOCK" &
SOCAT_PID=$!
sleep 0.3
trap "kill $SOCAT_PID 2>/dev/null; wait $SOCAT_PID 2>/dev/null" EXIT

# Clone via temporary bridge — auth header injected by Envoy
safe_git clone "http://127.0.0.1:${GIT_PORT}/$REPO.git" "$REPOS_DIR/$REPO_NAME"

cd "$REPOS_DIR/$REPO_NAME"

# ── Enable per-worktree config to protect the shared .git/config ──
# With worktreeConfig enabled, `git config --worktree` writes to a
# per-worktree config.worktree file instead of the shared .git/config.
# This lets us make .git/config read-only to sandboxed processes while
# still allowing legitimate per-session config writes in the worktree.
# Requires repositoryformatversion = 1 (one-way migration; older git
# clients will refuse to operate on this repo — acceptable in a
# controlled server environment).
safe_git config extensions.worktreeConfig true

# ── Lock down backing .git/ directory ──
# Make .git/config read-only to prevent config poisoning from sandboxed
# sessions (insteadOf/pushInsteadOf URL redirection, custom filter
# commands, or any future git config directives that could execute code).
# Worktree-specific config goes to .git/worktrees/<name>/config.worktree
# instead, which is scoped to a single slot. Uses a privileged helper
# because chattr +i requires root and add-repo runs as agent. The helper
# validates the path before applying the flag.
sudo /usr/local/bin/lock-git-config "$REPOS_DIR/$REPO_NAME/.git/config"

# Store repo metadata outside .git/ in a location not writable from inside
# the sandbox. This prevents a sandboxed session from tampering with the
# org/repo path or default branch used by trusted scripts.
mkdir -p "$META_DIR/$REPO_NAME"
echo "$REPO" > "$META_DIR/$REPO_NAME/github-path"

# Detect and store the default branch (main, master, or other)
DEFAULT_BRANCH=$(safe_git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||')
if [ -z "$DEFAULT_BRANCH" ]; then
  DEFAULT_BRANCH="main"
  echo "  Warning: could not detect default branch, assuming 'main'" >&2
fi
echo "$DEFAULT_BRANCH" > "$META_DIR/$REPO_NAME/default-branch"

echo "Added $REPO_NAME"
ADDREPO

chmod +x /usr/local/bin/add-repo

# ── Secrets rotation command (admin use) ──
cat > /usr/local/bin/rotate-secrets << 'ROTATE'
#!/bin/bash
set -euo pipefail

echo "Fetching fresh secrets from Secrets Manager..."
/usr/local/bin/refresh-proxy-secrets

echo "Restarting Envoy to apply new config..."
systemctl restart envoy

# Wait for Envoy to be ready (check unix socket since listeners are sockets now)
for i in $(seq 1 10); do
  if [ -S /var/run/envoy/anthropic.sock ] && \
     socat -u OPEN:/dev/null UNIX-CONNECT:/var/run/envoy/anthropic.sock 2>/dev/null; then
    echo "Envoy restarted successfully. New credentials active."
    exit 0
  fi
  sleep 1
done

echo "WARNING: Envoy did not become ready within 10 seconds." >&2
echo "Check: systemctl status envoy" >&2
exit 1
ROTATE

chmod 755 /usr/local/bin/rotate-secrets
chown root:root /usr/local/bin/rotate-secrets

# ── Privileged helper to make .git/config immutable (called by add-repo) ──
# add-repo runs as the agent user, but chattr +i requires CAP_LINUX_IMMUTABLE
# (root). This helper validates the path and applies the immutable flag.
cat > /usr/local/bin/lock-git-config << 'LOCKCONFIG'
#!/bin/bash
set -euo pipefail
TARGET="$1"
# Validate it's under the expected directory and ends in .git/config
if ! [[ "$TARGET" =~ ^/home/agent/workspaces/repos/[a-zA-Z0-9._-]+/\.git/config$ ]]; then
  echo "Refused: path not in expected location" >&2
  exit 1
fi
[ -f "$TARGET" ] || { echo "File not found" >&2; exit 1; }
chmod 444 "$TARGET"
chattr +i "$TARGET"
LOCKCONFIG
chmod 755 /usr/local/bin/lock-git-config
chown root:root /usr/local/bin/lock-git-config

# Sudoers rule — agent can only run this specific helper as root
echo 'agent ALL=(root) NOPASSWD: /usr/local/bin/lock-git-config' \
  > /etc/sudoers.d/agent-lock-git-config
chmod 440 /etc/sudoers.d/agent-lock-git-config

# ── Make pre-sandbox scripts, shared library, and Envoy binary immutable ──
# These execute outside the sandbox and form part of the trusted computing base.
# The Envoy binary holds all credentials in memory — a tampered binary could
# exfiltrate them. Immutability prevents modification even if an attacker gains
# write access as root — they would need to explicitly `chattr -i` first,
# which is logged by auditd.
chattr +i /usr/local/bin/envoy
chattr +i /usr/local/bin/code
chattr +i /usr/local/bin/add-repo
chattr +i /usr/local/bin/refresh-proxy-secrets
chattr +i /usr/local/bin/rotate-secrets
chattr +i /usr/local/bin/lock-git-config
chattr +i /usr/local/lib/safe-git.sh
chattr +i /usr/local/lib/subshell-rc.sh

# ── Install and configure auditd ──
dnf install -y audit
AGENT_UID=$(id -u agent)
cat > /etc/audit/rules.d/agent.rules << AUDITRULES
-a always,exit -F arch=b64 -F uid=$AGENT_UID -S execve -k agent_cmd
-w /home/agent/.ssh/ -p rwa -k ssh_keys
-w /home/admin/ -p rwa -k cross_user
-w /etc/envoy/ -p rwa -k envoy_config
-w /home/agent/.claude/ -p wa -k claude_config
-w /usr/local/bin/code -p wa -k trusted_scripts
-w /usr/local/bin/add-repo -p wa -k trusted_scripts
-w /usr/local/bin/refresh-proxy-secrets -p wa -k trusted_scripts
-w /usr/local/bin/rotate-secrets -p wa -k trusted_scripts
-w /usr/local/bin/lock-git-config -p wa -k trusted_scripts
-w /usr/local/lib/safe-git.sh -p wa -k trusted_scripts
-w /usr/local/lib/subshell-rc.sh -p wa -k trusted_scripts
-w /usr/local/bin/envoy -p wa -k trusted_scripts
AUDITRULES
systemctl enable auditd
augenrules --load

# ── SSH hardening ──
cat > /etc/ssh/sshd_config.d/hardened.conf << 'SSHCONF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
AuthenticationMethods publickey
PermitRootLogin no
AllowUsers agent admin
MaxSessions 8
SSHCONF
systemctl restart sshd

# ── Prepare SSH directories ──
for USER in agent admin; do
  mkdir -p /home/$USER/.ssh
  chmod 700 /home/$USER/.ssh
  touch /home/$USER/.ssh/authorized_keys
  chmod 600 /home/$USER/.ssh/authorized_keys
  chown -R $USER:$USER /home/$USER/.ssh
done
chown root:root /home/agent/.ssh/authorized_keys /home/admin/.ssh/authorized_keys

# ── Shell prompts ──
echo "PS1='\[\e[32m\]{dev}[\w]\n→ \[\e[36m\]'" >> /home/agent/.bashrc
cat > /home/agent/.bash_profile << 'BASHPROFILE'
source ~/.bashrc
# ── Auto-wrap interactive SSH sessions in tmux for disconnect resilience ──
# Each SSH connection gets its own tmux window in a grouped session.
# Multiple `ssh dev` connections each land in a separate window automatically.
# If disconnected, just `ssh dev` again — your old windows are still there.
if [ -n "$SSH_CONNECTION" ] && [ -z "$TMUX" ] && [ -t 0 ]; then
  if tmux has-session -t work 2>/dev/null; then
    exec tmux new-session -t work \; new-window
  else
    exec tmux new-session -s work
  fi
fi
BASHPROFILE
chown agent:agent /home/agent/.bashrc /home/agent/.bash_profile

echo "PS1='\[\e[31m\]{dev-admin}[\w]\n→ \[\e[36m\]'" >> /home/admin/.bashrc
echo "source ~/.bashrc" >> /home/admin/.bash_profile
chown admin:admin /home/admin/.bashrc /home/admin/.bash_profile

# ── File watchers and core dump suppression ──
cat >> /etc/sysctl.conf << 'SYSCTL'
fs.inotify.max_user_watches=524288
fs.inotify.max_user_instances=512
kernel.core_pattern=|/bin/false
SYSCTL
sysctl -p

# ── Block IMDS access for the agent user ──
# Prevents the agent from using instance metadata to obtain temporary AWS
# credentials and calling Secrets Manager directly, which would bypass
# the Envoy proxy and expose raw API keys.
AGENT_UID_IPTABLES=$(id -u agent)
iptables -A OUTPUT -m owner --uid-owner "$AGENT_UID_IPTABLES" -d 169.254.169.254 -j DROP
# Persist across reboots
iptables-save > /etc/sysconfig/iptables
cat > /etc/systemd/system/iptables-restore.service << 'IPTABLES_SVC'
[Unit]
Description=Restore iptables rules
Before=network-pre.target
Wants=network-pre.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/iptables-restore /etc/sysconfig/iptables
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
IPTABLES_SVC
systemctl daemon-reload
systemctl enable iptables-restore

# ── Swap ──
dd if=/dev/zero of=/swapfile bs=1M count=4096
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# ── Directories ──
mkdir -p /home/agent/logs /home/agent/shared/status
mkdir -p /home/agent/workspaces/repos /home/agent/workspaces/slots /home/agent/workspaces/meta /home/agent/.locks
chown -R agent:agent /home/agent/logs /home/agent/shared /home/agent/workspaces /home/agent/.locks

# ── Git identity and safety settings for agent user ──
# All settings are baked into this file and made immutable below.
# To change git identity, an admin must chattr -i, edit, and chattr +i.
mkdir -p /home/agent/.config/git
cat > /home/agent/.gitconfig << 'GITCONF'
[user]
	name = dev
	email = agent@yourorg.com
[core]
	hooksPath = /dev/null
	fsmonitor = false
[push]
	autoSetupRemote = true
[branch "main"]
	pushRemote = no_push
[branch "master"]
	pushRemote = no_push
[transfer]
	fsckObjects = true
GITCONF
chown agent:agent /home/agent/.gitconfig
# Make the gitconfig file immutable so agents cannot re-enable hooks
# or change identity/push safety settings
chattr +i /home/agent/.gitconfig

# ── Claude Code sandbox configuration ──
# Enforces OS-level isolation for all bash commands via bubblewrap.
# - enabled: sandbox is active for every session
# - autoAllowBashIfSandboxed: sandboxed commands run without prompting
# - failIfUnavailable: Claude Code refuses to start if sandbox can't initialize
# - allowUnsandboxedCommands: false — disables the escape hatch entirely
# - allowedDomains: localhost (socat bridges to Envoy) plus public package registries
#   Public registries need no auth and go directly through the NAT gateway.
#   Private registries route through Envoy for credential injection (see below).
mkdir -p /home/agent/.claude
cat > /home/agent/.claude/settings.json << 'CLAUDECONF'
{
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "failIfUnavailable": true,
    "allowUnsandboxedCommands": false,
    "network": {
      "allowedDomains": [
        "127.0.0.1",
        "registry.npmjs.org",
        "pypi.org",
        "files.pythonhosted.org",
        "index.crates.io",
        "static.crates.io",
        "crates.io",
        "soldeer.xyz"
      ]
    }
  }
}
CLAUDECONF
chown -R agent:agent /home/agent/.claude

# ── Protect sandbox config from agent tampering ──
# The agent user owns the directory for Claude Code to read it,
# but the settings file itself is owned by root and immutable.
chown root:agent /home/agent/.claude/settings.json
chmod 440 /home/agent/.claude/settings.json
chattr +i /home/agent/.claude/settings.json

# ── Resource limits for agent sessions ──
# nproc prevents fork bombs. nofile is raised above the default 1024 because
# Claude Code with file watching on large repos (especially with node_modules)
# can easily exhaust it, producing mysterious EMFILE errors. Address space (as)
# limits are intentionally omitted: Node.js maps large virtual address ranges
# that don't correspond to physical memory usage, and a tight `as` limit causes
# mysterious ENOMEM crashes. Memory overconsumption is caught by the CloudWatch
# memory alarm instead, which alerts without killing sessions.
cat >> /etc/security/limits.conf << 'LIMITS'
agent soft nofile 8192
agent hard nofile 16384
agent soft nproc 256
agent hard nproc 384
LIMITS

# ── CloudWatch agent ──
dnf install -y amazon-cloudwatch-agent
cat > /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json << 'CWCONFIG'
{
  "metrics": {
    "metrics_collected": {
      "mem": {
        "measurement": ["mem_used_percent"],
        "metrics_collection_interval": 300
      },
      "disk": {
        "measurement": ["disk_used_percent"],
        "resources": ["/"],
        "metrics_collection_interval": 300
      }
    },
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}"
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/home/agent/logs/agent-*.log",
            "log_group_name": "/agent/session-logs",
            "log_stream_name": "{instance_id}/{file_name}",
            "retention_in_days": 30
          },
          {
            "file_path": "/home/agent/logs/sessions.jsonl",
            "log_group_name": "/agent/session-events",
            "log_stream_name": "{instance_id}/sessions",
            "retention_in_days": 30
          },
          {
            "file_path": "/var/log/envoy/envoy.log",
            "log_group_name": "/agent/envoy-logs",
            "log_stream_name": "{instance_id}/envoy",
            "retention_in_days": 30
          },
          {
            "file_path": "/var/log/envoy/anthropic_access.log",
            "log_group_name": "/agent/envoy-access-logs",
            "log_stream_name": "{instance_id}/anthropic-access",
            "retention_in_days": 30
          },
          {
            "file_path": "/var/log/envoy/github_access.log",
            "log_group_name": "/agent/envoy-access-logs",
            "log_stream_name": "{instance_id}/github-access",
            "retention_in_days": 30
          },
          {
            "file_path": "/var/log/envoy/npm_private_access.log",
            "log_group_name": "/agent/envoy-access-logs",
            "log_stream_name": "{instance_id}/npm-private-access",
            "retention_in_days": 30
          },
          {
            "file_path": "/var/log/audit/audit.log",
            "log_group_name": "/agent/audit-logs",
            "log_stream_name": "{instance_id}/auditd",
            "retention_in_days": 90
          }
        ]
      }
    }
  }
}
CWCONFIG
systemctl enable amazon-cloudwatch-agent
/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s

# ── Start Envoy (after CloudWatch is configured to capture its logs) ──
systemctl start envoy

# ── Automatic security updates ──
dnf install -y dnf-automatic
sed -i 's/apply_updates = no/apply_updates = yes/' /etc/dnf/automatic.conf
systemctl enable --now dnf-automatic-install.timer

# ── Weekly cleanup cron (root-owned, agent cannot modify) ──
cat > /usr/local/bin/agent-cleanup << 'CLEANUP'
#!/bin/bash
# Runs as agent via cron, but the script itself is root-owned and immutable.

source /usr/local/lib/safe-git.sh

LOCKS_DIR="/home/agent/.locks"
SLOTS_DIR="/home/agent/workspaces/slots"
REPOS_DIR="/home/agent/workspaces/repos"
META_DIR="/home/agent/workspaces/meta"
SESSION_LOG="/home/agent/logs/sessions.jsonl"

# ── Kill orphaned socat bridges for inactive slots ──
# If a session was killed by SIGKILL or OOM, the trap handler never fired
# and socat bridge processes persist until the slot is next claimed (when
# start_bridges does fuser -k). This cleans them up proactively so they
# don't leak file descriptors and idle connections to Envoy's unix sockets.
for i in $(seq 1 6); do
  lockfile="$LOCKS_DIR/slot-$i.lock"
  if [ -f "$lockfile" ] && ! (flock -n 200 || exit 1) 200<>"$lockfile" 2>/dev/null; then
    continue  # Slot is active, skip
  fi
  for port in $((8080+i)) $((8090+i)) $((8100+i)); do
    fuser -k ${port}/tcp 2>/dev/null || true
  done
done

for i in $(seq 1 6); do
  lockfile="$LOCKS_DIR/slot-$i.lock"
  slot_dir="$SLOTS_DIR/slot-$i"
  [ -d "$slot_dir" ] || continue

  # Check if slot is locked (active session holds flock)
  if [ -f "$lockfile" ] && ! (flock -n 200 || exit 1) 200<>"$lockfile" 2>/dev/null; then
    continue  # Slot is active, skip
  fi

  # Clean up stale status file from crashed sessions
  rm -f "/home/agent/shared/status/slot-$i.json"

  cd "$slot_dir" || continue
  if [ -n "$(safe_git status --porcelain 2>/dev/null)" ]; then
    continue
  fi
  # Check for config poisoning before checkout/clean (these can trigger
  # custom filter commands outside the sandbox)
  if ! check_git_config_poison "$slot_dir" "slot-$i"; then
    echo "  WARNING: slot-$i has poisoned worktree config, quarantining" >&2
    # Log confirmed poisoning as a structured security event — ships to
    # CloudWatch via the sessions.jsonl pipeline for near-real-time alerting.
    echo "{\"event\":\"config_poison_quarantined\",\"slot\":$i,\"dir\":\"$slot_dir\",\"source\":\"agent-cleanup\",\"time\":\"$(date -Iseconds)\"}" \
      >> "$SESSION_LOG"
    # Resolve the real .git dir — in a worktree, .git is a file containing
    # a gitdir: pointer, not a directory.
    wt_config="$slot_dir/.git/config.worktree"
    if [ -f "$slot_dir/.git" ]; then
      real_git=$(sed 's/^gitdir: //' "$slot_dir/.git")
      wt_config="$real_git/config.worktree"
    fi
    [ -f "$wt_config" ] && mv "$wt_config" "$wt_config.quarantined.$(date +%s)"
    # Fall through to normal cleanup — slot is now safe to reset
  fi
  # Determine the default branch for this slot's repo
  slot_repo=$(safe_git remote get-url origin 2>/dev/null | grep -oP '[^/]+(?=\.git$)' || echo "")
  slot_default=$(cat "$META_DIR/$slot_repo/default-branch" 2>/dev/null || echo "main")
  if ! [[ "$slot_default" =~ ^[a-zA-Z0-9/._-]+$ ]]; then
    echo "  WARNING: slot-$i has invalid default-branch in meta, skipping" >&2
    continue
  fi
  safe_git checkout "$slot_default" 2>/dev/null
  safe_git branch --format='%(refname:short)' | while IFS= read -r branch; do
    case "$branch" in
      "$slot_default"|main|master) continue ;;
      *) safe_git branch -D "$branch" 2>/dev/null ;;
    esac
  done
done

for repo_dir in "$REPOS_DIR"/*/; do
  [ -d "$repo_dir" ] || continue
  cd "$repo_dir"
  safe_git worktree prune
  safe_git gc --auto
done

find /home/agent/logs -name "*.log" -mtime +7 -delete
# Truncate sessions.jsonl — CloudWatch has the canonical copy at /agent/session-events.
# Truncation is safe: active sessions append on next event; no data loss beyond the local copy.
find /home/agent/logs -name "sessions.jsonl" -size +10M -exec truncate -s 0 {} \;
npm cache clean --force 2>/dev/null
echo "Cleanup done $(date)"
CLEANUP

chmod 755 /usr/local/bin/agent-cleanup
chown root:root /usr/local/bin/agent-cleanup
chattr +i /usr/local/bin/agent-cleanup
su - agent -c '(crontab -l 2>/dev/null; echo "0 3 * * 0 /usr/local/bin/agent-cleanup >> /home/agent/logs/cleanup.log 2>&1") | crontab -'

echo "Setup complete $(date)" >> /var/log/agent-setup.log
```

Click **Launch instance**. Note the instance ID.

> **Remember:** Go back to step 2 and replace `REPLACE_WITH_INSTANCE_ID_AFTER_LAUNCH` in the IAM policy with this instance ID.

### Verify the User-Data Script Completed

The entire security model depends on the user-data script finishing successfully. If it fails partway through — for example, Envoy download errors, npm install times out, or an `chattr` command fails — you could end up with an instance that has the agent user but no sandbox enforcement, no IMDS blocking, or no immutable flags. **Verify before proceeding.**

1. Go to **EC2 → Instances** → select `dev` → **Connect → Session Manager → Connect**
2. Run:

```bash
# Check for the final success marker
tail -1 /var/log/agent-setup.log
# Should show: "Setup complete" with a timestamp

# If missing, check what went wrong:
sudo tail -100 /var/log/cloud-init-output.log

# Quick smoke test of critical security controls:
sudo systemctl status envoy            # Should be active
lsattr /usr/local/bin/code             # Should show ----i---
lsattr /home/agent/.claude/settings.json  # Should show ----i---
sudo iptables -L OUTPUT -n | grep 169.254.169.254  # Should show DROP rule
```

> **If the script failed:** Fix the issue, then either re-run the failed portion manually or terminate the instance and launch a fresh one with a corrected user-data script. Do not proceed with a partially provisioned instance.


---

## 7. Set Up SSM Session Logging

Configure SSM to log all terminal sessions to S3, providing an immutable, complete record of every SSH session.

### Create the S3 Bucket

1. Go to **S3 → Create bucket**
2. Bucket name: `YOUR_BUCKET_NAME-ssm-session-logs` (must be globally unique)
3. Region: same as your instance
4. **Block all public access**: leave enabled (default)
5. Bucket Versioning: **Enable** (prevents log deletion/overwriting)
6. Default encryption: **SSE-S3** (or SSE-KMS for stricter control)
7. **Create bucket**

### Set a Lifecycle Rule

1. Open the bucket → **Management → Create lifecycle rule**
2. Rule name: `expire-old-logs`
3. Apply to all objects
4. Transition to **S3 Glacier Flexible Retrieval** after **90 days**
5. **Expire** objects after **365 days** (adjust to your retention requirements)
6. **Create rule**

### Configure SSM Session Preferences

1. Go to **Systems Manager → Session Manager → Preferences → Edit**
2. Under **S3 logging**:
   - Check **Enable**
   - S3 bucket name: `YOUR_BUCKET_NAME-ssm-session-logs`
   - S3 key prefix: `sessions/`
   - Check **Encrypt log data** (uses the bucket's default encryption)
3. Under **CloudWatch logging** (optional, for real-time access):
   - Check **Enable**
   - Log group: `/ssm/session-logs`
   - Set retention to **30 days**
4. **Save**

> **What this captures:** Every keystroke and terminal output from every SSM session — including SSH sessions tunneled through SSM. Combined with Envoy access logs, you can reconstruct exactly what happened: what commands ran (auditd), what the terminal showed (SSM session logs), what API calls were made (Envoy access logs), and what was pushed (git history).


---

## 8. Set Up EBS Snapshots

1. Go to **EC2 → Elastic Block Store → Lifecycle Manager → Create lifecycle policy**
2. Policy type: **EBS snapshot policy**
3. Target resource type: **Instance**
4. Target resource tags: Key = `Name`, Value = `dev`
5. Policy description: `Nightly dev server backup`
6. Schedule:
   - Frequency: **Daily**
   - Starting at: **04:00 UTC**
   - Retention: **Count → 7**
7. **Create policy**


---

## 9. Set Up CloudWatch Alarms

### Create an SNS Topic

1. Go to **SNS → Topics → Create topic**
2. Type: **Standard**
3. Name: `dev-alerts`
4. **Create topic**
5. Click **Create subscription** → Protocol: **Email** → Endpoint: your email address → **Create subscription**
6. Confirm via email

### Create CPU Alarm

1. Go to **CloudWatch → Alarms → Create alarm → Select metric**
2. **EC2 → Per-Instance Metrics** → find your instance ID → select **CPUUtilization** → **Select metric**
3. Statistic: **Average**, Period: **5 minutes**
4. Threshold: **Greater than 85**
5. Datapoints to alarm: **3 out of 3**
6. Notification: select `dev-alerts`
7. Name: `dev-high-cpu`
8. **Create alarm**

### Create Memory Alarm

1. **Create alarm → Select metric**
2. **CWAgent → InstanceId** → select **mem_used_percent**
3. Statistic: **Average**, Period: **5 minutes**
4. Threshold: **Greater than 85**, Datapoints: **3 out of 3**
5. Notification: select `dev-alerts`
6. Name: `dev-high-memory`
7. **Create alarm**

### Create Disk Alarm

1. **Create alarm → Select metric**
2. **CWAgent → InstanceId, path** → path `/` → select **disk_used_percent**
3. Statistic: **Average**, Period: **5 minutes**
4. Threshold: **Greater than 85**, Datapoints: **1 out of 1**
5. Notification: select `dev-alerts`
6. Name: `dev-high-disk`
7. **Create alarm**

### Create Envoy Proxy Abuse Alarms

These metric filters turn Envoy access logs into near-real-time detection of anomalous proxy usage, such as a prompt-injected agent hammering the API or probing blocked paths.

**1. Create a metric filter for high Anthropic API request rate:**

1. Go to **CloudWatch → Log groups** → open `/agent/envoy-access-logs`
2. Click **Metric filters → Create metric filter**
3. Filter pattern: `{ $.response_code = * }` (matches all requests in the anthropic-access log stream)
4. Select the log group, filter only the `*/anthropic-access` log streams
5. Metric namespace: `DevServer/Envoy`
6. Metric name: `AnthropicRequestCount`
7. Metric value: `1`
8. Default value: `0`
9. **Create**

Then create an alarm on this metric:

1. Go to **CloudWatch → Alarms → Create alarm → Select metric**
2. **DevServer/Envoy → AnthropicRequestCount**
3. Statistic: **Sum**, Period: **5 minutes**
4. Threshold: **Greater than 200** (adjust based on your normal usage — 6 agents at moderate pace)
5. Datapoints to alarm: **2 out of 2**
6. Notification: select `dev-alerts`
7. Name: `dev-envoy-high-api-rate`
8. **Create alarm**

**2. Create a metric filter for 403 responses (blocked path probing):**

1. Back in the metric filters for `/agent/envoy-access-logs`
2. **Create metric filter**
3. Filter pattern: `{ $.response_code = 403 }`
4. Metric namespace: `DevServer/Envoy`
5. Metric name: `EnvoyForbiddenCount`
6. Metric value: `1`
7. Default value: `0`
8. **Create**

Then create an alarm:

1. **Create alarm → Select metric**
2. **DevServer/Envoy → EnvoyForbiddenCount**
3. Statistic: **Sum**, Period: **5 minutes**
4. Threshold: **Greater than 10**
5. Datapoints to alarm: **1 out of 1**
6. Notification: select `dev-alerts`
7. Name: `dev-envoy-forbidden-spike`
8. **Create alarm**

**3. Create a metric filter for 5xx responses (upstream/infrastructure failures):**

1. Back in the metric filters for `/agent/envoy-access-logs`
2. **Create metric filter**
3. Filter pattern: `{ $.response_code >= 500 }`
4. Metric namespace: `DevServer/Envoy`
5. Metric name: `EnvoyServerErrorCount`
6. Metric value: `1`
7. Default value: `0`
8. **Create**

Then create an alarm:

1. **Create alarm → Select metric**
2. **DevServer/Envoy → EnvoyServerErrorCount**
3. Statistic: **Sum**, Period: **5 minutes**
4. Threshold: **Greater than 5**
5. Datapoints to alarm: **1 out of 1**
6. Notification: select `dev-alerts`
7. Name: `dev-envoy-server-errors`
8. **Create alarm**

> **What these detect:** The API rate alarm catches runaway agents or automated abuse. The 403 alarm catches attempts to access blocked proxy paths — for example, a prompt injection trying to reach the GitHub REST API or Anthropic endpoints beyond `/v1/messages`. A burst of 403s is a strong signal that something is probing the proxy's route restrictions. The 5xx alarm catches infrastructure failures upstream of Envoy — NAT gateway outages, DNS resolution failures, upstream TLS errors, or Envoy misconfiguration. This is the one failure mode where all six agents go down simultaneously while the management plane (SSM) stays up, so proactive alerting is essential.

### Create Auditd Security Alarms

The audit rules detect the highest-severity events: attempts to modify the trusted computing base (immutable scripts, Envoy binary, Envoy config, Claude config). These deserve near-real-time alerting, not just manual `ausearch`.

**1. Create a metric filter for trusted script modification attempts:**

1. Go to **CloudWatch → Log groups** → open `/agent/audit-logs`
2. Click **Metric filters → Create metric filter**
3. Filter pattern: `trusted_scripts`
4. Metric namespace: `DevServer/Security`
5. Metric name: `TrustedScriptTamperAttempt`
6. Metric value: `1`
7. Default value: `0`
8. **Create**

Then create an alarm:

1. **Create alarm → Select metric**
2. **DevServer/Security → TrustedScriptTamperAttempt**
3. Statistic: **Sum**, Period: **1 minute**
4. Threshold: **Greater than 0**
5. Datapoints to alarm: **1 out of 1**
6. Notification: select `dev-alerts`
7. Name: `dev-trusted-script-tamper`
8. **Create alarm**

**2. Create a metric filter for Envoy config access:**

1. Back in the metric filters for `/agent/audit-logs`
2. **Create metric filter**
3. Filter pattern: `envoy_config`
4. Metric namespace: `DevServer/Security`
5. Metric name: `EnvoyConfigAccessAttempt`
6. Metric value: `1`
7. Default value: `0`
8. **Create**

Then create an alarm:

1. **Create alarm → Select metric**
2. **DevServer/Security → EnvoyConfigAccessAttempt**
3. Statistic: **Sum**, Period: **1 minute**
4. Threshold: **Greater than 0**
5. Datapoints to alarm: **1 out of 1**
6. Notification: select `dev-alerts`
7. Name: `dev-envoy-config-tamper`
8. **Create alarm**

**3. Create a metric filter for config poisoning events:**

Session events (`sessions.jsonl`) include structured entries when `agent-cleanup` or `code sync` detect and quarantine config poisoning. These are confirmed security events.

1. Go to **CloudWatch → Log groups** → open `/agent/session-events`
2. **Create metric filter**
3. Filter pattern: `{ $.event = "config_poison_*" }`
4. Metric namespace: `DevServer/Security`
5. Metric name: `ConfigPoisonDetected`
6. Metric value: `1`
7. Default value: `0`
8. **Create**

Then create an alarm:

1. **Create alarm → Select metric**
2. **DevServer/Security → ConfigPoisonDetected**
3. Statistic: **Sum**, Period: **1 minute**
4. Threshold: **Greater than 0**
5. Datapoints to alarm: **1 out of 1**
6. Notification: select `dev-alerts`
7. Name: `dev-config-poison-detected`
8. **Create alarm**

> **What these detect:** Any `trusted_scripts` or `envoy_config` audit match means someone is actively attempting to modify immutable files in the trusted computing base. Any `config_poison_*` session event means a sandboxed Claude session planted config poison that was caught and quarantined. All three warrant immediate investigation.

### Set Anthropic API Spend Limits

1. In the **Anthropic Console**: **Settings → Limits** → set a monthly spend cap
2. Set billing alerts at 50% and 80% of the cap


---

## 10. Generate Your FIDO2 SSH Key and Configure Access

### On Your MacBook

```bash
brew install --cask session-manager-plugin

ssh-keygen -t ed25519-sk -C "macbook-dev" -f ~/.ssh/dev_sk
# Touch your YubiKey when prompted
```

> If you get "invalid format", use `ecdsa-sk` instead.

> **YubiKey Bio:** Add `-O verify-required` to require fingerprint.

```bash
cat ~/.ssh/dev_sk.pub | pbcopy
```

### Bootstrap Into the Server

1. Go to **EC2 → Instances** → select `dev`
2. Click **Connect → Session Manager → Connect**

```bash
sudo su -
echo "PASTE_YOUR_PUBLIC_KEY_HERE" >> /home/agent/.ssh/authorized_keys
echo "PASTE_YOUR_PUBLIC_KEY_HERE" >> /home/admin/.ssh/authorized_keys
chattr +i /home/agent/.ssh/authorized_keys
chattr +i /home/admin/.ssh/authorized_keys
```

### Configure SSH on Your MacBook

Add to `~/.ssh/config`:

```
Host dev
  HostName i-0abc123def456
  User agent
  IdentityFile ~/.ssh/dev_sk
  ServerAliveInterval 15
  ServerAliveCountMax 4
  ProxyCommand aws --profile dev ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'

Host dev-admin
  HostName i-0abc123def456
  User admin
  IdentityFile ~/.ssh/dev_sk
  ServerAliveInterval 15
  ServerAliveCountMax 4
  ProxyCommand aws --profile dev ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'
```

Replace `i-0abc123def456` with your actual instance ID.

### Verify SSH Access Before Closing the Console Session

**Do not close the SSM console session until you confirm SSH works from your MacBook.** Open a terminal on your MacBook and run:

```bash
ssh dev echo "agent key works"
ssh dev-admin echo "admin key works"
```

Both commands should print their message after a YubiKey touch. If either fails, you still have the SSM console session open to fix the authorized_keys.

> **Troubleshooting:** If you get "Permission denied (publickey)", verify the key was pasted correctly, the file permissions are right (`600` on authorized_keys, `700` on `.ssh`), and you're using the correct IdentityFile path. If needed, remove the immutable flag (`sudo chattr -i`), fix the key, and re-apply.


---

## 11. Configure Git and Add Repos

### Git Identity

Git identity and safety settings are configured during instance provisioning (in the user-data script) and baked into the immutable `~/.gitconfig`. The provisioned defaults are:

- **`user.name = dev`** and **`user.email = agent@yourorg.com`** — commit identity (edit these in the user-data script before launch, or see below to change post-launch)
- **`push.autoSetupRemote = true`** — auto-tracks remote branches on first push
- **`branch.main.pushRemote = no_push`** / **`branch.master.pushRemote = no_push`** — prevents accidental direct pushes to protected branches
- **`core.hooksPath = /dev/null`** — first layer of defense against malicious repository hooks
- **`core.fsmonitor = false`** — disables the fsmonitor hook, another git config option that can execute arbitrary commands
- **`transfer.fsckObjects = true`** — validates incoming git objects during fetch, defending against crafted malicious objects that exploit git's object parsing

> **Defense in depth for git config:** Git local config (per-repo `.git/config`) overrides global config for all settings, including `core.hooksPath`, `core.fsmonitor`, `credential.helper`, `core.askPass`, `core.pager`, `core.editor`, `core.sshCommand`, and `diff.external`. Since the sandbox allows writes to the backing repo's `.git/config`, a sandboxed session could re-enable hooks, set a malicious credential helper, askPass, pager, editor, SSH command, or diff program, re-enable fsmonitor, add `insteadOf` / `pushInsteadOf` URL redirection rules, or define custom filter commands that execute during git operations in trusted scripts outside the sandbox. Four layers mitigate this: (1) `add-repo` enables `extensions.worktreeConfig` and makes `.git/config` immutable (`chmod 444` + `chattr +i`), so sandboxed config writes go to a per-worktree file and the shared config cannot be modified, deleted, or replaced; (2) all trusted scripts source a shared `check_git_config_poison()` function from `/usr/local/lib/safe-git.sh` that detects `insteadOf`, `pushInsteadOf`, custom filter commands (with an allowlist for git LFS), and executable config overrides (`core.hooksPath`, `core.fsmonitor`, `core.askPass`, `core.pager`, `core.editor`, `core.sshCommand`, `diff.external`, `diff.tool`, `merge.tool`, `sequence.editor`) before any fetch or checkout; (3) all trusted scripts source a shared `safe_git()` wrapper that passes `-c` flags to override all command-executing config keys on every invocation (command-line `-c` flags take precedence over all config levels — `credential.helper` is handled by the runtime override only, not detection, to avoid false positives from legitimate system-level helpers); and (4) repo metadata (`github-path`, `default-branch`) is stored in `~/workspaces/meta/` outside `.git/` to prevent sandbox tampering.

> **To change git identity after launch:** An admin must `ssh dev-admin`, remove the immutable flag (`sudo chattr -i /home/agent/.gitconfig`), edit the file, and re-apply (`sudo chattr +i /home/agent/.gitconfig`).

### Git Credential Routing via Envoy

All git operations route through the Envoy proxy via unix sockets. The `code` and `add-repo` commands create per-session socat bridges from localhost TCP ports to the Envoy unix socket, and Envoy injects the GitHub PAT automatically. No credential helper needed, no token in the environment.

The `add-repo` command clones via a temporary socat bridge. The `code` command starts per-slot bridges on deterministic ports (8091–8096 for git) and overrides the remote URL via `GIT_CONFIG` environment variables, scoped to the session's process tree. This avoids mutating the shared `.git/config` and eliminates remote URL races when two slots work on the same repo.

> **Note:** The GitHub proxy only forwards git smart-HTTP paths (`*.git/info/refs`, `*.git/git-upload-pack`, `*.git/git-receive-pack`). All other paths return `403 Forbidden`. This prevents a prompt injection from using the proxy to call the GitHub REST API (e.g., to create webhooks, modify settings, or read data beyond what git operations expose).

### Verify Envoy Is Running

```bash
ssh dev
# Check that the Anthropic proxy socket exists and is connectable
ls -la /var/run/envoy/anthropic.sock
# Should show: srw-rw---- envoy envoy ... /var/run/envoy/anthropic.sock

# Verify agent can connect (will get 403 since / is blocked — confirms Envoy is running)
curl -s --unix-socket /var/run/envoy/anthropic.sock http://localhost/ -o /dev/null -w '%{http_code}'
# Should print: 403
```

> **Note:** The Envoy admin interface is bound to a separate unix socket (`/var/run/envoy/admin.sock`) with mode `0600`, owned by the `envoy` user. The agent user cannot access the admin endpoint, which prevents credential extraction via `/config_dump`.

### Verify Sandbox Runtime Is Installed

```bash
ssh dev
which srt
# Should print: /usr/lib/node_modules/@anthropic-ai/sandbox-runtime/bin/srt (or similar)

cat ~/.claude/settings.json
# Should show sandbox enabled with failIfUnavailable: true
```

### Verify the Dummy API Key Is Accepted

The `code` script sets `ANTHROPIC_API_KEY` to a format-passing dummy value (`sk-ant-proxy00-...`) that Envoy strips and replaces with the real key. This ensures Claude Code's key format validation (if any) passes at startup. Verify the proxy round-trip works:

```bash
ssh dev
# Start a socat bridge to Envoy
socat TCP-LISTEN:9999,bind=127.0.0.1,reuseaddr,fork \
  UNIX-CONNECT:/var/run/envoy/anthropic.sock &
SOCAT_PID=$!
sleep 0.3

export ANTHROPIC_BASE_URL="http://127.0.0.1:9999"
export ANTHROPIC_API_KEY="sk-ant-proxy00-000000000000000000000000000000000000000000000000"
claude --version  # Basic startup check

# Then start an interactive session briefly to confirm it connects:
# claude
# Type a short message, verify it gets a response, then /exit

kill $SOCAT_PID 2>/dev/null
```

> **Important:** Test this before relying on the setup. If the dummy value is rejected, every `code` session will fail at launch with a confusing error that doesn't mention the key format.

### Verify Backing Repo Config Is Protected

The sandbox allows writes to the backing repo's `.git/` directory, which means a sandboxed session could poison `.git/config` with `insteadOf` / `pushInsteadOf` URL redirection rules, custom filter commands, or other directives that execute code in trusted scripts running outside the sandbox. Three layers of protection mitigate this:

1. **`extensions.worktreeConfig` + immutable `.git/config`** (primary control): `add-repo` enables per-worktree config and makes `.git/config` immutable (`chmod 444` + `chattr +i`). Sandboxed `git config` writes go to the per-worktree `config.worktree` file instead. This blocks config poisoning at the write boundary — including `unlink()` + recreate attacks that would bypass mode `0444` alone, since `chattr +i` prevents deletion regardless of parent directory permissions.
2. **`check_git_config_poison()`** (detection): The `code` and `add-repo` scripts check for `insteadOf`, `pushInsteadOf`, and custom filter commands before any fetch, checkout, or reset. Git LFS entries (`filter.lfs.*`) are allowlisted. This catches poison that predates the protection or survives a permissions misconfiguration.
3. **`safe_git()` wrapper** (per-invocation override): Command-line `-c` flags override `core.hooksPath`, `core.fsmonitor`, `credential.helper`, `core.askPass`, `core.pager`, `core.editor`, `core.sshCommand`, and `diff.external` on every git invocation, regardless of what any config file says.

Verify all three layers after adding a repo:

```bash
ssh dev

# 1. Verify .git/config is immutable:
add-repo yourorg/yourrepo
ls -la ~/workspaces/repos/yourrepo/.git/config
# Should show: -r--r--r-- (mode 444)
lsattr ~/workspaces/repos/yourrepo/.git/config
# Should show: ----i--- (immutable flag set)

# 2. Verify worktreeConfig is enabled:
cd ~/workspaces/repos/yourrepo
git config extensions.worktreeConfig
# Should print: true

# 3. Verify that a sandboxed write to .git/config fails:
code
# In a Claude Code session, try:
git config --local core.hooksPath /tmp/test-hooks
# Should fail with: error: could not lock config file .git/config: Permission denied

# 4. Verify that worktree-scoped config writes still succeed:
git config --worktree user.name "test"
git config --worktree --unset user.name
# Both should succeed (writes to .git/worktrees/slot-N/config.worktree)

# 5. Test if the meta directory is writable from inside the sandbox:
echo "test" > ~/workspaces/meta/yourrepo/test-write
# This SHOULD fail with Permission denied. If it succeeds, the sandbox
# is not confining writes to the worktree — see remediation below.
# 6. If step 5 failed (expected), no cleanup needed. If it succeeded:
rm -f ~/workspaces/meta/yourrepo/test-write
```

If the `.git/config` write in step 3 succeeds despite the immutable flag (e.g., filesystem doesn't support extended attributes), escalate to making the backing `.git/config` owned by root and re-applying the immutable flag:

```bash
ssh dev-admin
sudo chown root:root /home/agent/workspaces/repos/REPO_NAME/.git/config
sudo chmod 444 /home/agent/workspaces/repos/REPO_NAME/.git/config
sudo chattr +i /home/agent/workspaces/repos/REPO_NAME/.git/config
```

If `meta/` is writable from inside the sandbox (step 5 succeeded), the `validate_github_path()` cross-validation catches the high-severity case (`github-path` tampering that redirects authenticated git operations). The residual risk is `default-branch` tampering, which can cause a session to start from the wrong branch of the correct repo — see Accepted Tradeoffs. To close this gap fully, make `meta/` root-owned and use a privileged helper for writes:

```bash
ssh dev-admin

# Lock down meta/ directory
sudo chown -R root:agent /home/agent/workspaces/meta
sudo chmod 750 /home/agent/workspaces/meta
# Existing repo subdirs need the same treatment:
sudo find /home/agent/workspaces/meta -type d -exec chmod 750 {} \;
sudo find /home/agent/workspaces/meta -type f -exec chmod 640 {} \;
```

Then create `/usr/local/bin/write-repo-meta` — a root-owned helper that validates inputs and writes the two meta files:

```bash
cat > /usr/local/bin/write-repo-meta << 'WRITEMETA'
#!/bin/bash
set -euo pipefail
REPO_NAME="$1"
GITHUB_PATH="$2"
DEFAULT_BRANCH="$3"

# Validate inputs
if ! [[ "$REPO_NAME" =~ ^[a-zA-Z0-9._-]+$ ]]; then
  echo "Invalid repo name" >&2; exit 1
fi
if ! [[ "$GITHUB_PATH" =~ ^[a-zA-Z0-9._-]+/[a-zA-Z0-9._-]+$ ]]; then
  echo "Invalid github path" >&2; exit 1
fi
if ! [[ "$DEFAULT_BRANCH" =~ ^[a-zA-Z0-9/_.-]+$ ]]; then
  echo "Invalid branch name" >&2; exit 1
fi

META_DIR="/home/agent/workspaces/meta/$REPO_NAME"
mkdir -p "$META_DIR"
echo "$GITHUB_PATH" > "$META_DIR/github-path"
echo "$DEFAULT_BRANCH" > "$META_DIR/default-branch"
chown -R root:agent "$META_DIR"
chmod 750 "$META_DIR"
chmod 640 "$META_DIR"/*
WRITEMETA
chmod 755 /usr/local/bin/write-repo-meta
chown root:root /usr/local/bin/write-repo-meta
chattr +i /usr/local/bin/write-repo-meta

# Sudoers rule — agent can only run this specific helper as root
echo 'agent ALL=(root) NOPASSWD: /usr/local/bin/write-repo-meta' \
  > /etc/sudoers.d/agent-meta
chmod 440 /etc/sudoers.d/agent-meta
```

Then update `add-repo` to call `sudo write-repo-meta "$REPO_NAME" "$REPO" "$DEFAULT_BRANCH"` instead of writing directly to `meta/`. This is optional — only needed if the sandbox allows meta/ writes and your risk tolerance requires closing the `default-branch` tampering gap.

### Verify IMDS Is Blocked for Agent

```bash
ssh dev
# This should hang and then time out (DROP rule):
curl -sf -m 3 http://169.254.169.254/latest/meta-data/
# Expected: curl exits with error (timeout), no response

# Verify from admin that the iptables rule is in place:
ssh dev-admin
sudo iptables -L OUTPUT -n -v | grep 169.254.169.254
# Should show a DROP rule for the agent UID
```

### Verify Secrets Access (admin only)

```bash
ssh dev-admin
sudo aws secretsmanager get-secret-value \
  --secret-id "api-keys" \
  --query 'SecretString' \
  --output text | jq .
```

### Verify Admin Socket Is Restricted (admin only)

```bash
ssh dev-admin
# Admin can access Envoy stats via the unix socket:
sudo curl --unix-socket /var/run/envoy/admin.sock http://localhost/stats | head

# Verify the agent user CANNOT access it:
sudo -u agent curl --unix-socket /var/run/envoy/admin.sock http://localhost/stats
# Should fail: Permission denied
```

### Verify Envoy Access Logging

```bash
ssh dev-admin
# After running at least one Claude Code session:
sudo tail /var/log/envoy/anthropic_access.log
# Should show JSON-formatted request logs with timestamps, paths, response codes

sudo tail /var/log/envoy/github_access.log
# Should show git operation logs (fetch, push)

# Verify logs are shipping to CloudWatch:
# CloudWatch → Log groups → /agent/envoy-access-logs
```

### Verify GitHub Access Log ACL

```bash
ssh dev
# Agent should be able to read the GitHub access log (for event-driven sync):
tail -1 /var/log/envoy/github_access.log
# Should show a JSON log entry (or empty if no git operations yet)

# Agent should NOT be able to read other Envoy logs:
cat /var/log/envoy/anthropic_access.log
# Should fail: Permission denied

cat /var/log/envoy/envoy.log
# Should fail: Permission denied
```

### Verify Trusted Scripts Are Immutable

```bash
ssh dev-admin
lsattr /usr/local/bin/envoy /usr/local/bin/code /usr/local/bin/add-repo /usr/local/bin/refresh-proxy-secrets /usr/local/bin/rotate-secrets /usr/local/bin/lock-git-config /usr/local/lib/safe-git.sh /usr/local/lib/subshell-rc.sh
# All should show: ----i--- (immutable flag set)
```

### Add Your Repos

```bash
ssh dev
add-repo yourorg/frontend
add-repo yourorg/backend
add-repo yourorg/infra
```

Add more at any time with the same command. When you add new repos, update your GitHub PAT's repository access to include them.

### GitHub Branch Protection (Required)

For each repo:

1. On GitHub: **repo → Settings → Branches → Add branch protection rule**
2. Branch name pattern: `main`
3. Enable:
   - **Require a pull request before merging**
   - **Require approvals** (at least 1)
   - **Do not allow bypassing the above settings**
4. **Save changes**

### CI/CD Pipeline Hardening (Required)

Agents push to feature branches by design. If your CI system triggers on any branch push (the default for GitHub Actions, Jenkins, GitLab CI, etc.), a prompt-injected agent could commit a malicious workflow file (`.github/workflows/`, `Jenkinsfile`, `.gitlab-ci.yml`) to its branch. That workflow would then execute in your CI environment with whatever credentials the CI runner has (deployment keys, cloud credentials, npm tokens, etc.).

This risk exists outside the EC2 instance — it's a property of your CI configuration — but it must be addressed since the entire workflow assumes agents push branches freely.

**GitHub Actions mitigations:**

1. **Require workflow approval for PRs from new contributors.** Go to **repo → Settings → Actions → General → Fork pull request workflows** and select "Require approval for all outside collaborators."
2. **Use a `paths` filter or `workflow_dispatch` trigger** on sensitive workflows so they don't fire on every branch push.
3. **Restrict the `GITHUB_TOKEN` permissions** in your workflow files to the minimum needed: add `permissions:` at the top of each workflow with explicit read/write scopes.
4. **Pin third-party actions to SHA hashes**, not tags (tags can be moved).

**General CI mitigations (any platform):**

1. **Restrict which branches trigger pipelines.** Only allow CI to run on `main` and branches matching a pattern like `release/*`. Feature branches should only trigger CI when a PR is opened (where review gates apply).
2. **Separate CI credentials from deployment credentials.** Build-and-test pipelines should not have access to production secrets.
3. **Review workflow file changes in PRs.** Any PR that modifies CI configuration files should require explicit approval from a maintainer.


---

## 12. Daily Workflow

Open as many terminals as you need. Each one is an independent Claude Code session running inside the sandbox, with its own dedicated proxy bridge ports.

**Your sessions survive disconnects.** Every SSH connection is automatically wrapped in tmux. If your MacBook loses internet, sleeps, or the WiFi drops, just `ssh dev` again — you'll reconnect to your existing session with Claude Code still running. SSH keepalives handle brief blips (under ~60 seconds) transparently; tmux handles everything longer.

```
# Terminal 1                    # Terminal 2                    # Terminal 3
ssh dev                         ssh dev                         ssh dev
code                            code                            code

  Repos: frontend backend infra

  slot 1: empty                   slot 1: active [backend]        slot 1: active [backend]
  slot 2: empty                     (ports 8081/8091)               (ports 8081/8091)
  ...                             slot 2: empty                   slot 2: active [frontend]
                                  ...                               (ports 8082/8092)
                                                                  ...

  Pick a repo:                    Pick a repo:                    Pick a repo:
    1) frontend                     1) frontend                     1) frontend
    2) backend                      2) backend                      2) backend
    3) infra                        3) infra                        3) infra

  > 2                             > 1                             > 3
  Branch [agent/backend-0401-1422]:                                 Branch [agent/infra-0401-1425]:

  Slot 1 | backend | ... |        Slot 2 | frontend | ... |      Slot 3 | infra | ... |
  sandbox: on                     sandbox: on                     sandbox: on
  API bridge: localhost:8081      API bridge: localhost:8082      API bridge: localhost:8083
  Git bridge: localhost:8091      Git bridge: localhost:8092      Git bridge: localhost:8093

  > claude starts                 > claude starts                 > claude starts
```

When Claude finishes or you exit:

```bash
# After Claude exits, you're dropped into a subshell with live bridges:
  Claude exited. Bridges still active.
  Review, commit, and push. Ctrl-D to release slot.

(slot-1) ~/workspaces/slots/slot-1 → git diff
(slot-1) ~/workspaces/slots/slot-1 → git add -A && git commit -m "agent: refactored auth"
(slot-1) ~/workspaces/slots/slot-1 → git push origin HEAD
(slot-1) ~/workspaces/slots/slot-1 → ^D   # Ctrl-D to release the slot

  Slot 1 released.

# Start another session on a different repo
code
  > 3   # pick infra this time
```

Other commands:

```bash
code status       # see all slots (with port assignments for active sessions)
code kill 3       # stop a running session in slot 3
code sync         # fetch and reset idle slots
add-repo yourorg/new-service   # add a repo any time
```

### Local repo sync (on your MacBook)

Agent pushes show up as `git-receive-pack` entries in the Envoy GitHub access log. Tailing this log over SSH gives you event-driven local fetching — zero GitHub API calls, no polling, instant updates. Save this as `~/bin/watch-pushes` and `chmod +x` it:

```bash
#!/bin/bash
# Event-driven local fetch: tails the Envoy GitHub access log over SSH
# and fetches locally whenever an agent pushes. Zero GitHub API calls.
#
# Only catches pushes that go through the EC2 server's Envoy proxy.
# Pushes from your MacBook, teammates, or GitHub PR merges won't appear.
# For those, use watch-repos (polling) instead or in addition.

REPO_DIR=~/code
DEBOUNCE=5  # seconds — a single push generates multiple log entries

declare -A LAST_FETCH

fetch_repo() {
  local repo=$1
  local now
  now=$(date +%s)
  local last=${LAST_FETCH[$repo]:-0}
  if (( now - last < DEBOUNCE )); then
    return
  fi
  LAST_FETCH[$repo]=$now
  if [ -d "$REPO_DIR/$repo/.git" ]; then
    git -C "$REPO_DIR/$repo" fetch --all --quiet 2>/dev/null
    echo "$(date +%H:%M:%S) fetched $repo"
  fi
}

while true; do
  echo "$(date +%H:%M:%S) connected — watching for agent pushes..."
  # Catch-up fetch on (re)connect to cover any pushes missed while disconnected
  for dir in "$REPO_DIR"/*/; do
    [ -d "$dir/.git" ] || continue
    git -C "$dir" fetch --all --quiet 2>/dev/null
  done
  ssh dev "tail -f /var/log/envoy/github_access.log" 2>/dev/null \
    | grep --line-buffered "git-receive-pack" \
    | while read -r line; do
        repo=$(echo "$line" | jq -r .path | grep -oP '[^/]+(?=\.git)')
        [ -n "$repo" ] && fetch_repo "$repo"
      done
  echo "$(date +%H:%M:%S) connection lost, reconnecting in 5s..."
  sleep 5
done
```

Run it in a terminal tab: `watch-pushes &`. Your editor's git integration picks up agent pushes within seconds. Update `REPO_DIR` to match your local clone path.

> **How it works:** The GitHub access log is readable by the `agent` user via a POSIX ACL (set during provisioning). The script tails it over your normal `ssh dev` connection — no admin access needed. When the SSH connection drops, it reconnects automatically and does a catch-up fetch to cover any pushes missed during the gap. A 5-second debounce prevents redundant fetches from the multiple log entries a single push generates.

> **Alternative: polling.** If you also need to track pushes from outside the EC2 server (your MacBook, teammates, GitHub PR merges), use a polling approach instead. Replace the `ssh dev "tail -f ..."` pipeline with periodic `git -C "$repo" ls-remote` calls at a 15-second interval. This uses GitHub API calls but catches all pushes regardless of origin.

### Secret rotation

```bash
# After updating secrets in AWS Secrets Manager:
ssh dev-admin
sudo rotate-secrets
# Active Claude Code sessions are unaffected — Envoy picks up new keys on restart
# socat bridges automatically reconnect to the new Envoy sockets
```

### Updating Claude Code and sandbox-runtime

Claude Code and sandbox-runtime are npm global packages that receive updates — potentially including security fixes. Check for updates periodically (e.g., when rotating the GitHub PAT):

```bash
ssh dev-admin
sudo npm outdated -g @anthropic-ai/claude-code @anthropic-ai/sandbox-runtime

# If updates are available:
sudo npm update -g @anthropic-ai/claude-code @anthropic-ai/sandbox-runtime

# Verify:
claude --version
srt --version
```

> **Note:** `dnf-automatic` handles OS package updates, but npm global packages are not covered. If stability matters more than latest features, pin to a known-good version: `sudo npm install -g @anthropic-ai/claude-code@X.Y.Z`.

### Updating Envoy

Envoy holds all credentials in memory and handles all external traffic. A CVE in Envoy is the highest-severity vulnerability in this setup. Unlike OS packages (covered by `dnf-automatic`) and npm globals (checked manually), Envoy is a standalone binary with no auto-update mechanism. Check for security releases periodically:

```bash
ssh dev-admin

# Check current version:
/usr/local/bin/envoy --version

# Check for new releases:
# https://github.com/envoyproxy/envoy/releases

# To update (on your MacBook, compute the new hash first):
#   curl -fsSL -O "https://github.com/envoyproxy/envoy/releases/download/vX.Y.Z/BINARY_NAME"
#   shasum -a 256 BINARY_NAME
# Then on the server:
sudo curl -fsSL -o /tmp/envoy-new "https://github.com/envoyproxy/envoy/releases/download/vX.Y.Z/BINARY_NAME"
echo "NEW_SHA256  /tmp/envoy-new" | sudo sha256sum -c -
sudo chmod +x /tmp/envoy-new

# Validate the existing config against the NEW binary before replacing.
# If this fails, the production binary is still intact — investigate without downtime.
sudo /tmp/envoy-new --mode validate -c /etc/envoy/envoy.yaml

# Only now replace the production binary:
sudo chattr -i /usr/local/bin/envoy
sudo mv /tmp/envoy-new /usr/local/bin/envoy
sudo chattr +i /usr/local/bin/envoy

# Restart (brief <1s interruption; socat bridges reconnect automatically):
sudo systemctl restart envoy
sudo systemctl status envoy
```

> **Important:** Always validate the config against the new binary before replacing. Envoy config format can change between major versions. The temp-file-then-move pattern ensures that if the download is interrupted, corrupted, or the new binary rejects the config, the production binary remains intact and the proxy continues operating.

### System maintenance (admin only)

```bash
ssh dev-admin
sudo dnf install ...

# Review audit logs:
sudo ausearch -k agent_cmd --start today
sudo ausearch -k envoy_config --start today
sudo ausearch -k claude_config --start today
sudo ausearch -k trusted_scripts --start today

# Check Envoy status:
sudo systemctl status envoy
sudo curl --unix-socket /var/run/envoy/admin.sock http://localhost/stats \
  | grep -E 'upstream_cx_total|upstream_rq_[245]'

# Review Envoy access logs (API call audit trail):
sudo tail -20 /var/log/envoy/anthropic_access.log | jq .
sudo tail -20 /var/log/envoy/github_access.log | jq .

# Query access logs for a specific time window:
sudo cat /var/log/envoy/anthropic_access.log \
  | jq 'select(.timestamp > "2025-04-01T10:00:00")' \
  | jq '{timestamp, method, path, response_code, bytes_sent}'

# Review session events (slot start/stop with port mappings):
cat /home/agent/logs/sessions.jsonl | jq .
# Correlate Envoy access logs with sessions by matching timestamps
# to the start/stop events for each slot

# Verify sandbox config is intact:
cat /home/agent/.claude/settings.json
lsattr /home/agent/.claude/settings.json
# Should show: ----i--- (immutable flag set)

# Verify git hooks are disabled:
lsattr /home/agent/.gitconfig
# Should show: ----i--- (immutable flag set)

# Verify trusted scripts are intact:
lsattr /usr/local/bin/envoy /usr/local/bin/code /usr/local/bin/add-repo /usr/local/bin/refresh-proxy-secrets /usr/local/bin/rotate-secrets /usr/local/bin/agent-cleanup /usr/local/bin/lock-git-config /usr/local/lib/safe-git.sh /usr/local/lib/subshell-rc.sh
# All should show: ----i--- (immutable flag set)

# Verify Envoy sockets exist with correct permissions:
ls -la /var/run/envoy/
# Should show: anthropic.sock and github.sock with mode srw-rw---- owned by envoy:envoy
# Agent access is via group membership (agent is in envoy group)

# Verify IMDS is blocked for agent:
sudo iptables -L OUTPUT -n -v | grep 169.254.169.254
# Should show DROP rule for agent UID

# Check agent logs in CloudWatch (immutable copy):
# CloudWatch → Log groups → /agent/session-logs
# CloudWatch → Log groups → /agent/session-events  (structured slot start/stop events)
# CloudWatch → Log groups → /agent/envoy-logs
# CloudWatch → Log groups → /agent/envoy-access-logs
# CloudWatch → Log groups → /ssm/session-logs (full terminal recordings)
```


---

## 13. Backup YubiKey

1. Generate a backup key:

```bash
ssh-keygen -t ed25519-sk -C "macbook-dev-backup" -f ~/.ssh/dev_sk_backup
```

2. Add it to both users:

```bash
ssh dev-admin
sudo chattr -i /home/agent/.ssh/authorized_keys /home/admin/.ssh/authorized_keys
echo "BACKUP_PUBLIC_KEY_HERE" | sudo tee -a /home/agent/.ssh/authorized_keys
echo "BACKUP_PUBLIC_KEY_HERE" | sudo tee -a /home/admin/.ssh/authorized_keys
sudo chattr +i /home/agent/.ssh/authorized_keys /home/admin/.ssh/authorized_keys
```

3. Store the backup YubiKey in a different location from your primary.

4. SSM Session Manager through the AWS console is your emergency backdoor.


---

## 14. Security Notes

### What's Protected

- **No public IP.** Instance unreachable from the internet. SSM traffic stays inside VPC via endpoints.
- **YubiKey per connection.** Physical key required for every SSH session.
- **No sudo for agents.** Claude Code cannot escalate to root. The agent user has no sudo permissions. Envoy restarts and secret rotation are admin-only operations.
- **OS-level sandbox on every session.** The sandbox-runtime uses bubblewrap to enforce filesystem and network isolation at the kernel level. Bash commands and all their child processes are confined to the worktree directory (read-write) with read-only access elsewhere. Network access is restricted to `127.0.0.1` (socat bridges to Envoy) and a set of public package registries (npm, PyPI, crates.io, Soldeer). Authenticated API traffic (Anthropic, GitHub, private registries) must go through the per-slot socat bridges to Envoy's credential-injecting unix sockets. The sandbox is mandatory: `failIfUnavailable: true` prevents Claude Code from starting without it, and `allowUnsandboxedCommands: false` disables the escape hatch that would let commands bypass the sandbox.
- **Sandbox config is immutable.** The `~/.claude/settings.json` file is owned by `root:agent` with mode `440` and the immutable flag (`chattr +i`). The agent user cannot modify, delete, or replace it. Changes to the sandbox config are audited via auditd (`claude_config` key).
- **Defense in depth: sandbox + Envoy + security groups.** Three independent layers enforce network restrictions. The sandbox blocks direct outbound at the OS level. Envoy controls which upstream services receive credentials. The VPC security group blocks all non-443 egress at the network level. Compromising one layer doesn't defeat the others.
- **API keys never in agent process memory.** The Envoy proxy injects credentials into outbound requests at the network layer. Claude Code sends requests to a per-slot socat bridge on localhost, which forwards to Envoy's unix socket; Envoy strips the dummy key and injects the real key before forwarding to `api.anthropic.com`. A prompt injection running `printenv`, reading `/proc/self/environ`, or inspecting process memory finds only a format-passing dummy value (`sk-ant-proxy00-...`) — not the real key. The real key exists only in Envoy's config file, owned by `root:envoy` with mode `600`.
- **Envoy listeners on unix sockets, not TCP ports.** Both the Anthropic and GitHub proxy listeners bind to unix sockets (`/var/run/envoy/anthropic.sock`, `/var/run/envoy/github.sock`) with mode `0660`, owned by `envoy:envoy`. Only the `envoy` user and members of the `envoy` group (which includes `agent`) can connect. This eliminates TCP port squatting as an attack vector: a rogue process cannot bind to a well-known port to intercept proxy traffic, because there are no well-known ports. The per-slot socat bridges that translate between TCP and unix sockets are session-scoped and die with the session.
- **Core dumps disabled system-wide.** `kernel.core_pattern=|/bin/false` in sysctl and `LimitCORE=0` in the Envoy systemd unit prevent core dumps that could contain credentials from being written to disk. This provides two independent layers: the systemd setting covers Envoy specifically, while the sysctl setting covers every process on the host — including any child processes or crashes outside systemd's supervision.
- **IMDS blocked for agent user.** An iptables rule drops all traffic from the `agent` UID to `169.254.169.254` (the instance metadata service). Without this, the agent could obtain temporary AWS credentials via IMDS and call Secrets Manager directly, bypassing Envoy entirely and extracting raw API keys. The rule is persisted across reboots via `iptables-restore` at boot.
- **Envoy admin interface inaccessible to agents.** The admin endpoint is bound to a unix socket (`/var/run/envoy/admin.sock`) with mode `0600`, owned by the `envoy` user. The agent user cannot connect to it. This prevents credential extraction via `/config_dump`, which would otherwise expose the raw API keys embedded in Envoy's route configuration. Admins can access it via `sudo curl --unix-socket /var/run/envoy/admin.sock http://localhost/stats`.
- **Envoy log access scoped via POSIX ACL.** The agent user has read access to only `/var/log/envoy/github_access.log` (via `setfacl`), which contains request metadata (timestamps, paths, response codes) and no secrets. The Anthropic access log, Envoy operational log, and admin access log remain agent-inaccessible. This enables event-driven local repo syncing (see section 12) without granting broader log access.
- **GitHub token never in agent environment.** Git operations route through per-slot socat bridges to the Envoy GitHub unix socket, which injects the `Authorization` header. No credential helper, no token in env vars or process memory.
- **GitHub proxy path-restricted.** Envoy only forwards the three git smart-HTTP paths (`*.git/info/refs`, `*.git/git-upload-pack`, `*.git/git-receive-pack`). All other paths — including the GitHub REST API and any other `.git/` subpaths — return `403 Forbidden`. This prevents prompt injections from using the proxy to call the GitHub REST API — for example, to create webhooks, modify repository settings, or read data beyond what git operations expose. Even if the PAT's permissions are later expanded, the proxy enforces a git-only surface area.
- **Envoy path normalization enabled.** Both proxy listeners set `normalize_path: true` and `merge_slashes: true`, preventing encoded path traversal (`%2F..%2F`) or double-slash tricks from bypassing the regex route matching. This removes a dependency on Envoy's default normalization behavior across versions.
- **Envoy runs as a dedicated system user.** The `envoy` user has no login shell, no home directory, and no sudo access. Its only capability is reading its own config and binding to unix sockets.
- **Envoy config is root-generated, validated, and envoy-readable.** The `refresh-proxy-secrets` script runs as root, fetches from Secrets Manager, validates secret formats (prefix checks for `sk-ant-` and `github_pat_`) and character sets (rejects values containing characters outside `[a-zA-Z0-9_-]` to prevent YAML injection via the heredoc interpolation), writes to a temp file, validates the config with `envoy --mode validate`, and only then atomically moves it into place. If validation fails, the previous working config is restored from backup. The agent user cannot read `/etc/envoy/envoy.yaml`.
- **Envoy hardened via systemd.** `NoNewPrivileges`, `ProtectSystem=strict`, `ProtectHome=true`, `PrivateTmp`, empty `CapabilityBoundingSet`. Envoy can only write to its log directory and socket directory.
- **Envoy binary integrity verified at install and update, immutable at runtime.** The Envoy binary is verified against a pinned SHA256 hash after download during instance provisioning. If the hash does not match, the boot script aborts, preventing a supply-chain-compromised binary from running with access to all credentials. The boot script also fails fast if the SHA256 placeholder has not been replaced, preventing accidental deployment without integrity verification. Binary updates use a temp-file-then-move pattern: the new binary is downloaded to `/tmp`, hash-verified, and config-validated against the existing config before replacing the production binary. If any step fails, the production binary remains intact. After installation, the binary is protected with `chattr +i` and monitored by auditd, preventing runtime tampering.
- **Envoy log rotation prevents disk exhaustion.** Local Envoy access logs are rotated daily with 7-day retention via logrotate (`copytruncate` avoids needing to signal Envoy for log reopening). CloudWatch retains the canonical immutable copy for 30 days.
- **Anthropic API proxy path-scoped.** Envoy only forwards requests matching `/v1/messages*` to the Anthropic API. All other paths return `403 Forbidden`. This prevents an agent or prompt injection from using the proxy to access Anthropic account management, key creation, or billing endpoints even if the API key has those permissions.
- **Anthropic API timeouts accommodate extended thinking.** The Anthropic proxy listener uses 600-second timeouts for both `stream_idle_timeout` and `request_timeout`, matching the GitHub listener. This provides headroom for Claude Code sessions using extended thinking, where streaming responses may have long pauses between chunks during complex reasoning.
- **Pre-sandbox scripts and Envoy binary are immutable and audited.** All scripts that execute outside the sandbox (`code`, `add-repo`, `refresh-proxy-secrets`, `rotate-secrets`, `agent-cleanup`), privileged helpers (`lock-git-config`), the shared libraries (`/usr/local/lib/safe-git.sh`, `/usr/local/lib/subshell-rc.sh`), and the Envoy binary (`/usr/local/bin/envoy`) are root-owned and protected with `chattr +i`. This prevents modification even if an attacker gains write access as root — they would need to explicitly `chattr -i` first. All modification attempts are logged by auditd via the `trusted_scripts` key.
- **Per-slot session logging for incident correlation.** Each `code` session writes structured JSON events (start/stop with slot number, bridge ports, repo, branch, PID, and timestamp) to `~/logs/sessions.jsonl`. This log ships to CloudWatch (`/agent/session-events`) and enables correlating Envoy access log entries with specific Claude Code sessions by matching timestamps to each slot's active time window and bridge port assignments.
- **Request-level access logging with anomaly detection.** Both the Anthropic and GitHub proxy listeners log every request in structured JSON format (timestamp, method, path, response code, bytes sent/received, duration, downstream remote address). Logs ship to CloudWatch for immutable retention. CloudWatch metric filters and alarms detect anomalous patterns in near-real-time: sustained high API request rates trigger `dev-envoy-high-api-rate`, and bursts of 403 responses (blocked path probing) trigger `dev-envoy-forbidden-spike`. Both alert via SNS email.
- **Full terminal session recording.** SSM session logging captures the complete terminal stream (all input and output) for every SSH session and writes it to S3 with bucket versioning enabled (immutable). Combined with Envoy access logs and auditd, this provides four independent audit layers: commands executed (auditd), terminal I/O (SSM session logs), API calls made (Envoy access logs), and code changes (git history).
- **Git hooks, fsmonitor, credential helper, pager, editor, and other executable config keys neutralized via two independent layers.** The agent user's `~/.gitconfig` sets `core.hooksPath = /dev/null` and `core.fsmonitor = false`, and is protected with `chattr +i`. However, git local config (per-repo `.git/config`) overrides global config, so a malicious repo or sandboxed session could potentially re-enable hooks, set a malicious credential helper, pager, editor, or diff program via per-repo config. Three layers prevent this: (1) `add-repo` makes `.git/config` immutable (`chmod 444` + `chattr +i`) and enables `extensions.worktreeConfig` so sandboxed config writes go to a per-worktree file; (2) `check_git_config_poison()` detects known poisoning patterns (including executable config overrides like `core.hooksPath`, `core.fsmonitor`, `core.askPass`, `core.pager`, `core.editor`, `core.sshCommand`, `diff.external`, `diff.tool`, `merge.tool`, `sequence.editor` — `credential.helper` is intentionally omitted from detection to avoid false positives from legitimate system-level helpers, but is cleared by the runtime override) before any fetch or checkout; and (3) all trusted scripts source a shared `safe_git()` wrapper from `/usr/local/lib/safe-git.sh` (root-owned, immutable) that passes `-c` flags to override all command-executing config keys on every invocation. Command-line `-c` flags take precedence over all config levels (local, global, system), making each call immune to per-repo config overrides.
- **Git `url.*.insteadOf`, `pushInsteadOf`, and custom filter commands detected before fetch.** Per-repo `.git/config` can set `url.<base>.insteadOf` or `url.<base>.pushInsteadOf` rules that silently redirect git fetch or push URLs to an attacker-controlled endpoint, bypassing Envoy. Custom filter commands (`filter.*.clean`, `filter.*.smudge`, `filter.*.process`) execute during `checkout`, `reset --hard`, and `clean` — operations that trusted scripts run outside the sandbox with outbound HTTPS available. Unlike `hooksPath` and `fsmonitor`, these can't be blanket-overridden with a single `-c` flag because the keys include variable names. The shared `check_git_config_poison()` function in `safe-git.sh` detects all three families before any fetch, checkout, or reset. Git LFS entries are allowlisted only if both the key prefix (`filter.lfs.*`) and the value prefix (`git-lfs `) match — preventing an attacker from keeping the LFS key name while substituting a malicious command. The check runs against the backing repo config before the first fetch (preventing `insteadOf` from taking effect during the fetch itself) and again against the worktree config as a second pass.
- **Backing repo `.git/config` is immutable.** The `add-repo` script enables `extensions.worktreeConfig` and makes `.git/config` immutable (`chmod 444` + `chattr +i`) after cloning. This redirects `git config --worktree` writes to a per-worktree `config.worktree` file (under `.git/worktrees/<slot>/`) while preventing any modification, deletion, or replacement of the shared config. Mode `0444` alone would be insufficient — `unlink()` checks the parent directory's write permission, and the sandbox allows writes to `.git/`. The `chattr +i` flag prevents deletion regardless of parent directory permissions. This is the primary control against config poisoning — the detection checks in `check_git_config_poison()` serve as defense-in-depth.
- **Repo metadata stored outside `.git/` with cross-validation against frozen config.** The `github-path` and `default-branch` files that trusted scripts use to construct remote URLs and determine the default branch are stored in `~/workspaces/meta/REPO_NAME/`, not in the repo's `.git/` directory. This prevents a sandboxed session from modifying these files to redirect subsequent sessions to a different repository or branch. As an additional safeguard, `validate_github_path()` in `safe-git.sh` cross-validates the `meta/github-path` value against the org/repo path embedded in the frozen `.git/config` remote URL (which is read-only) before any authenticated git operation. If the paths diverge, the script aborts — preventing meta-directory tampering from redirecting fetches and pushes to an attacker-controlled repo even if the sandbox allows writes to `meta/`.
- **Git push safety baked into immutable config.** The agent user's `~/.gitconfig` includes `branch.main.pushRemote = no_push` and `branch.master.pushRemote = no_push`, preventing accidental direct pushes to protected branches. These settings are protected by the same `chattr +i` flag that protects `core.hooksPath`, so they cannot be overridden by a prompt injection.
- **Git object integrity verified on fetch.** `transfer.fsckObjects = true` in the immutable `~/.gitconfig` validates incoming git objects during fetch, defending against crafted malicious objects that exploit git's object parsing. This adds negligible overhead and catches corrupted or adversarial objects before they enter the local repository.
- **Git protocol restricted to HTTP only.** `GIT_ALLOW_PROTOCOL=http` is exported by the shared `safe-git.sh` library, blocking git's `ext::` transport (which allows arbitrary command execution via remote URLs), as well as `ssh://`, `file://`, and `git://` protocols. All legitimate git operations in this setup use plain HTTP to localhost socat bridges, so no other protocol is needed. This eliminates the entire class of protocol-based command execution attacks — including via `.gitmodules` submodule URLs and crafted refs — regardless of config or sandbox state.
- **Git hangs prevented on credential failure.** `GIT_TERMINAL_PROMPT=0` is exported by the shared `safe-git.sh` library, which is sourced by every script that runs git operations (`code`, `add-repo`, `agent-cleanup`). If a socat bridge dies and git falls back to interactive credential prompting, it fails fast with a clear error instead of hanging the session silently.
- **Post-Claude subshell hardened against config poisoning.** The subshell that runs after Claude exits (for manual review and `git push`) operates outside the sandbox with outbound HTTPS available via the NAT gateway. Two protections prevent a sandboxed Claude session from planting config poison (e.g., `pushInsteadOf` in `config.worktree`) that would take effect in the subshell: (1) `check_git_config_poison()` re-runs against the worktree after Claude exits, aborting without a subshell if poison is detected; (2) the subshell loads `/usr/local/lib/subshell-rc.sh` (root-owned, immutable) which aliases `git` to `safe_git`, applying the same `-c` flag overrides that trusted scripts use. This neutralizes `credential.helper`, `core.askPass`, `core.hooksPath`, `core.sshCommand`, and `core.fsmonitor` even in interactive use.
- **PAT scoped to specific repos.** The GitHub fine-grained PAT only has access to the repos you explicitly selected. A compromised token cannot access other repos in your org.
- **Server-side branch protection.** Cannot be bypassed by anything on the dev server.
- **CI/CD hardening guidance included.** See section 11 for required CI pipeline mitigations to prevent prompt-injected agents from committing malicious workflow files to feature branches.
- **Audit trail with immutable log shipping and alerting.** auditd logs agent commands, Envoy config access, Claude Code config access, and trusted script modification attempts. Agent session logs, Envoy logs, Envoy access logs, and auditd logs all ship to CloudWatch Logs in near-real-time (immutable copy). SSM session recordings ship to S3 with versioning (immutable copy). Structured session events (slot start/stop with port mappings, config poisoning detections) ship to CloudWatch for per-slot correlation. CloudWatch alarms fire on security-relevant audit keys and config poisoning events for near-real-time alerting via SNS.
- **Egress restricted to HTTPS.** Security group blocks all outbound except port 443 and VPC-internal DNS.
- **Proactive alerts.** CloudWatch alarms for CPU, memory, disk, Envoy API request rate, Envoy 403 spikes, auditd security events (trusted script modification attempts, Envoy config access), and config poisoning detections. Anthropic API spend cap prevents runaway costs.
- **Immutable SSH authorized_keys.** Agent cannot add rogue SSH keys.
- **Resource limits via PAM.** Per-session process count limits prevent fork bombs. File descriptor limits are raised above defaults (8192 soft / 16384 hard) to accommodate Claude Code's file watching on large repos while still providing an upper bound.
- **IAM resource ARNs scoped to region and account.**
- **Atomic slot locking via flock.** Slots are locked using kernel-level `flock` on file descriptors. The lock is automatically released when the shell process exits, regardless of how it exits (normal, signal, crash). No stale-lock detection needed — impossible to have orphaned locks.
- **Inter-agent visibility without coordination risk.** Agents can see what other slots are working on via status files, but cannot send instructions to each other.
- **Zero-downtime secret rotation.** `rotate-secrets` fetches fresh credentials from Secrets Manager, validates the generated config, and restarts Envoy. Active Claude Code sessions continue working — their socat bridges automatically reconnect to the new Envoy sockets, and subsequent requests use the new credentials. If config validation fails, the previous working config is restored from backup.
- **Cleanup script is tamper-proof and self-healing.** The weekly cleanup cron runs as the agent user but executes a root-owned, immutable script (`/usr/local/bin/agent-cleanup`). A prompt injection cannot modify the script to execute arbitrary code outside the sandbox on the next cron run. If the cleanup detects a poisoned worktree `config.worktree`, it quarantines the file (renamed with a `.quarantined.<timestamp>` suffix for forensic review) and proceeds with normal slot reset, preventing poisoned slots from becoming permanently stuck.

### Accepted Tradeoffs

- **Outbound HTTPS unrestricted at the security group level.** The VPC security group allows port 443 to any destination. However, the sandbox network policy restricts agent processes to `127.0.0.1` only — all outbound HTTPS must go through per-slot socat bridges to Envoy's unix sockets, and Envoy currently proxies only Anthropic and GitHub. A next hardening step is to add domain allowlisting in Envoy to restrict which upstream hosts it will forward to.
- **Public package registries are reachable but are potential exfiltration channels.** The sandbox allows direct HTTPS access to `registry.npmjs.org`, `pypi.org`, `files.pythonhosted.org`, `index.crates.io`, `static.crates.io`, `crates.io`, and `soldeer.xyz`. This lets `npm install`, `pip install`, `cargo build`, and `forge soldeer install` work inside the sandbox. However, a prompt injection could encode data in package name lookups or request metadata to these domains. VPC flow logs capture the connections but not the content. This is an accepted tradeoff: blocking registries would break Claude Code's ability to install dependencies, which is a core part of its workflow.
- **Private registries route through Envoy for credential injection.** If `NPM_PRIVATE_TOKEN` is set in Secrets Manager, Envoy proxies and authenticates requests to the private registry (default: `npm.pkg.github.com`, configurable via `NPM_PRIVATE_REGISTRY`). The same architecture applies to other private registries — add additional tokens, listeners, and clusters following the same pattern. Private registry traffic is logged in Envoy access logs alongside API and git traffic. Configure your repo's `.npmrc` to point scoped packages at the per-slot bridge URL using the `NPM_PRIVATE_REGISTRY_URL` environment variable exported by the `code` script (npm supports `${VAR}` interpolation in `.npmrc`). You must also add `_authToken` entries for all six slot ports (8101–8106) — npm requires auth config per registry URL, but the token value is irrelevant ("dummy" is fine) because Envoy strips the inbound `Authorization` header and injects the real token. Do not put real tokens in `.npmrc`. See the full `.npmrc` example in the `code` script comments.
- **No request-rate cap on the Anthropic proxy.** The CloudWatch alarm detects sustained high API request rates but is reactive — it fires after two consecutive 5-minute periods above the threshold, during which a runaway agent could accumulate significant spend. The Anthropic Console spend cap provides an outer bound but operates on a monthly timescale. For a preventive control, Envoy's `envoy.filters.http.local_ratelimit` can enforce a per-listener requests-per-second cap (e.g., 3 req/s would accommodate 6 agents comfortably while capping worst-case burst spend). This is a straightforward follow-up hardening step.
- **Admin user is root-equivalent.** Intentional for maintenance. Mitigated by separate SSH key, separate user, YubiKey required, audit logging.
- **Agent can discover the proxy architecture.** An agent running `curl localhost:8081` (if a socat bridge for their slot happens to use that port) can confirm a proxy exists. With the sandbox active, these localhost requests are permitted since `127.0.0.1` is in the allowed domains list. The agent cannot extract credentials: the admin interface is on a unix socket inaccessible to the agent user, the Anthropic proxy only forwards `/v1/messages*` paths, and the GitHub proxy only forwards git smart-HTTP paths (all others return 403). However, the agent could attempt to use the proxy to make authenticated API calls to the allowed paths. Mitigate by using an API key scoped to only the permissions Claude Code needs.
- **Brief interruption during secret rotation.** Restarting Envoy causes a brief (<1s) window where the proxy is unavailable. Active Claude Code sessions' socat bridges will fail to connect during this window; retries will succeed once the new Envoy sockets are created. For true zero-downtime rotation, Envoy's hot restart mechanism can be configured.
- **socat bridges add a process per listener per slot.** Each active slot runs two socat bridge processes (one for Anthropic, one for GitHub). At full capacity (6 slots), this is 12 socat processes — negligible resource overhead but additional processes to manage. Bridges are cleaned up automatically when the session exits via the trap handler. If the trap doesn't fire (SIGKILL, OOM), orphaned bridges persist until the slot is next claimed (`start_bridges` does `fuser -k`) or until the weekly `agent-cleanup` cron kills them.
- **No container isolation.** Agents run as the same user (`agent`) in separate SSH sessions. They share the same kernel, network namespace, and can see each other's processes via `/proc`. The sandbox mitigates this significantly: each session's bash commands are confined to their own worktree directory. `/proc/PID/environ` reveals only a format-passing dummy key (`sk-ant-proxy00-...`) — not real credentials.
- **PAT is longer-lived than GitHub App tokens.** The PAT expires every 90 days instead of every hour. If compromised (via the proxy, since it's not in the environment), it provides access until manually revoked or until expiry. Mitigated by: scoping to specific repos, storing only in Secrets Manager and Envoy config, never in agent-accessible files.
- **`code sync` and session start remove all untracked files.** Both `code sync` (for idle slots) and every new `code` session (before branch checkout) run `git clean -fdx`, which removes `node_modules`, build caches, and other expensive-to-recreate artifacts. The next session pays a cold-start penalty reinstalling dependencies. This is the correct tradeoff: a prompt injection can leave behind untracked files (e.g., `CLAUDE.md`, `.env`, `.npmrc`, `Makefile`) designed to influence future sessions, and a clean slot should mean clean.
- **Shared git object database.** All worktrees for a given repo share the same `.git` directory. A corrupt git operation in one worktree could theoretically affect others. The shared `.git/config` is immutable (`chattr +i`) with `extensions.worktreeConfig` enabled, so per-slot config goes to worktree-specific files, but refs, objects, and index are still shared.
- **Agent logs are mutable locally.** The agent can delete files in `~/logs/`. The CloudWatch copy is immutable. SSM session recordings in S3 are versioned and immutable.
- **Agents share a GitHub PAT.** All 6 agents use the same token with the same permissions. There is no per-agent credential scoping. This is simpler than per-slot tokens but means you cannot revoke access for a single slot without affecting all of them.
- **Envoy is a single point of failure.** If Envoy crashes, all 6 agent sessions lose API and GitHub access (their socat bridges will fail to connect to the unix sockets). Mitigated by `Restart=always` in the systemd unit (auto-recovery in ~5 seconds) and the Envoy readiness check in the `code` launcher. Socat bridges will reconnect to the new sockets automatically on the next request.
- **Sandbox adds minor overhead.** bubblewrap introduces minimal latency on filesystem operations. In practice this is not measurable for typical development workflows.
- **Sandbox config requires admin to modify.** Because `~/.claude/settings.json` is root-owned and immutable, the agent user cannot adjust sandbox settings (e.g., to allow a new domain or path). This is intentional — sandbox configuration changes require an admin SSH session.
- **Git hooks globally disabled for agent, with per-invocation enforcement in trusted scripts.** The agent user cannot use legitimate git hooks (pre-commit linters, etc.) due to `core.hooksPath = /dev/null` in the immutable global gitconfig. Additionally, all trusted scripts enforce `-c` overrides for all command-executing config keys (`core.hooksPath`, `core.fsmonitor`, `credential.helper`, `core.askPass`, `core.pager`, `core.editor`, `core.sshCommand`, `diff.external`) on every git invocation via a shared `safe_git()` wrapper sourced from `/usr/local/lib/safe-git.sh`, which overrides any per-repo config. Custom filter commands and executable config overrides are detected by `check_git_config_poison()` before any fetch, checkout, or reset, with an allowlist for git LFS (`filter.lfs.*`). If you need hooks, replace `/dev/null` with a directory containing only vetted hook scripts (owned by root and immutable), and update the shared `safe_git()` wrapper to point to the same directory. If you need custom filters beyond LFS, add them to the allowlist in `check_git_config_poison()`.
- **Agent gitconfig is immutable.** The agent cannot change any git configuration set in `~/.gitconfig` (including `core.hooksPath`, `core.fsmonitor`, `user.name`, `user.email`, and push safety settings). The backing repo's `.git/config` is also immutable (`chattr +i`). Per-worktree `config.worktree` settings work for non-security-sensitive options, but trusted scripts override security-sensitive settings via command-line `-c` flags which take precedence over all config levels.
- **SSM session logs add storage costs.** Terminal recordings can be verbose, especially for long Claude Code sessions. Mitigated by the S3 lifecycle rule (Glacier after 90 days, expiry after 365 days). Expect roughly 1–10 MB per session depending on output volume.
- **Updating immutable files requires admin access.** To modify any of the trusted scripts (`code`, `add-repo`, `refresh-proxy-secrets`, `rotate-secrets`, `agent-cleanup`), privileged helpers (`lock-git-config`), the shared libraries (`/usr/local/lib/safe-git.sh`, `/usr/local/lib/subshell-rc.sh`), or the Envoy binary (`/usr/local/bin/envoy`), an admin must first `sudo chattr -i /path/to/file`, make the change, and re-apply `sudo chattr +i`. This is intentional friction — changes to the trusted computing base should be deliberate and auditable. See section 12 for the Envoy update procedure. To remove the immutable flag on a backing repo's `.git/config` (e.g., for one-time config changes), an admin must `sudo chattr -i` the file and re-apply after the change.
- **Per-slot session correlation is timestamp-based.** Envoy access logs don't contain a slot identifier directly. Correlation requires matching Envoy log timestamps against the start/stop events in `sessions.jsonl` to determine which slot was active on which bridge port at that time. This is straightforward for post-hoc analysis but not instant — for real-time per-slot monitoring, consider adding per-slot Envoy listeners in a future hardening step.
- **Git remote URLs overridden via environment or `-c` flag, not config.** Each `code` session sets the remote URL via `GIT_CONFIG` environment variables rather than mutating `.git/config`. `code sync` and `add-repo` pass the URL via `-c remote.origin.url=...` on the fetch command. This avoids writes to the immutable `.git/config`, eliminates races when two slots work on the same repo (each session's env vars are independent), and eliminates stale remote URLs in idle slots (the persisted config is never touched).
- **Event-driven local sync only catches agent pushes.** The `watch-pushes` script tails the Envoy GitHub access log, so it only sees pushes that go through the EC2 server's proxy. Pushes from your MacBook, teammate pushes, and GitHub PR merges are invisible to it. The script does a catch-up `fetch --all` on each reconnect, which partially compensates. For complete coverage of all push sources, use polling (`ls-remote` at 15-second intervals) instead of or in addition to the event-driven approach.
- **Sandbox allows writes to backing `.git/` directories.** Git worktrees need write access to the backing repo's `.git/` directory (at `~/workspaces/repos/REPO/.git/`) for normal operations (updating refs, index, objects). The sandbox does not block these writes, which means a sandboxed session could attempt to poison the shared `.git/config`. Three layers mitigate this: (1) `add-repo` enables `extensions.worktreeConfig` and makes `.git/config` immutable (`chmod 444` + `chattr +i`), preventing modification, deletion, or replacement of the shared config; (2) the shared `check_git_config_poison()` function detects `insteadOf`, `pushInsteadOf`, and custom filter commands (with a git LFS allowlist) before any fetch, checkout, or reset in trusted scripts; (3) the `safe_git()` wrapper overrides `core.hooksPath`, `core.fsmonitor`, `credential.helper`, `core.askPass`, `core.sshCommand`, and `diff.external` on every invocation. If a future git config directive introduces a new code execution vector not covered by the detection, the immutable `.git/config` is the primary control that blocks it generically.
- **`meta/default-branch` is writable if sandbox allows `meta/` writes.** The `validate_github_path()` cross-validation protects `github-path` (the high-severity field that controls where authenticated git operations are directed) by comparing it against the frozen `.git/config` remote URL. However, `default-branch` is not cross-validated — there is no equivalent frozen reference. If a sandboxed session can write to `~/workspaces/meta/`, tampering with `default-branch` could cause the next session to check out the wrong starting branch of the correct repo. This does not cross a security boundary (no credential redirection, no exfiltration path), but could cause an agent to work from an unexpected base. To close this gap, make `meta/` root-owned and use the `write-repo-meta` privileged helper (see section 11).
- **Envoy log rotation has a tiny data loss window.** `copytruncate` copies the log file and then truncates the original. Any log lines written between the copy and truncate operations are lost locally. This is acceptable because CloudWatch has the canonical immutable copy, and the window is typically microseconds.
- **npm global packages are not auto-updated.** Claude Code and sandbox-runtime require manual updates (see section 12). OS packages are covered by `dnf-automatic`, but npm globals are not. Check for updates periodically.
- **Envoy has no auto-update mechanism.** The Envoy binary is installed from a GitHub release and must be updated manually (see section 12). Since Envoy holds all credentials and handles all external traffic, it is the highest-severity component to leave unpatched. Subscribe to the [Envoy security advisories](https://github.com/envoyproxy/envoy/security/advisories) and prioritize Envoy CVEs over other updates.


---

## 15. Cost Summary

| Component | Monthly Cost |
|-----------|-------------|
| m6g.2xlarge (1yr RI, no upfront) | ~$140 |
| 200 GB gp3 (6000 IOPS, 400 MB/s) | ~$22 |
| EBS snapshots (~200 GB × 7) | ~$2 |
| NAT gateway (fixed + data) | ~$35 |
| VPC endpoints (3 × ~$7.50) | ~$23 |
| VPC flow logs | ~$2 |
| CloudWatch (alarms + metrics + logs) | ~$5 |
| S3 session logs | ~$1 |
| Secrets Manager (1 secret) | ~$0.40 |
| Data transfer | ~$5 |
| **Total** | **~$235/mo** |


---

## 16. Reserve the Instance

Run on-demand for a week to validate, then:

1. Go to **EC2 → Reserved Instances → Purchase Reserved Instances**
2. Platform: **Linux/UNIX**
3. Instance type: `m6g.2xlarge`
4. Tenancy: **Default**
5. Term: **1 year**
6. Payment option: **No Upfront** (flexible) or **All Upfront** (cheapest)
7. Purchase
