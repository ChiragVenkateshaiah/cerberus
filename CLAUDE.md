# Cerberus — Working Rules for Claude Code

These rules apply to every session in this repo. Read this before proposing
or running any command.

## 1. Agentic workflow
Infrastructure work happens via Claude Code CLI + AWS CLI, issuing commands
directly (not GUI clicking). `cerberus-admin` profile for anything that
creates/modifies/destroys infra (`tofu apply/destroy`). `cerberus` profile
(cerberus-cli identity) for read-only/operational commands. Never mix the two.

## 2. Explain-after-action (mandatory — Chirag is new to cloud engineering)
After ANY command that changes AWS state (tofu apply, aws cli create/put/
delete calls, etc.), you MUST immediately follow with:
  a. A plain-English explanation of what actually happened — not just
     "success", but what resource now exists, what it's for, and how it
     connects to what already existed.
  b. A step-by-step path to verify the SAME thing in the AWS Console
     (logged in as the cerberus-console IAM user), written as: "Console ->
     [Service] -> [exact menu path] -> what you should see and what it
     means." Assume no prior AWS console familiarity — name every click.
  c. If relevant, one or two AWS CLI read-only commands (aws ... describe/
     get/list) Chirag can run himself in the WSL2 terminal to inspect the
     same resource — this is deliberate muscle-memory practice, not optional
     flourish. Flag which ones are worth him re-running from memory later
     without being told.

## 3. Console verification (cerberus-console IAM user)
Chirag checks the AWS Console using the dedicated `cerberus-console` IAM
user (ReadOnlyAccess policy + MFA — created specifically so console
verification can never accidentally create/modify infrastructure; all real
changes still go through OpenTofu). Root is NOT used for routine checks —
only the one-time creation of this IAM user required it. When giving
console steps, always give the FULL path from login:
  1. Use the cerberus-console sign-in link (account-specific URL, not the
     root login page) -> username `cerberus-console` + password -> MFA code
     from authenticator app
  2. Confirm region selector top-right matches ap-south-1 (Mumbai) —
     wrong-region is the #1 cause of "I don't see my resource"
  3. Exact service + exact left-nav path to the resource
  4. What field/tab proves the resource is configured as intended
  Note: this user has ReadOnlyAccess only. If a verification step ever
  seems to require clicking a create/edit/delete button, that's a signal
  something is wrong — real changes belong in OpenTofu, not the console.

## 4. CLI inspection (WSL2) — build the muscle
When suggesting a verification CLI command, prefer read-only calls scoped
to the resource just touched (e.g. `aws s3api get-bucket-versioning`,
`aws glue get-database`, `aws ssm get-parameter`) over broad list commands.
Explain what each flag does the first few times; stop over-explaining once
a command type has been used 2-3 times, per Chirag's own judgment.

## 5. Standing project rules (do not relitigate without Chirag raising it)
- Never auto-commit. Chirag reviews every diff before commit.
- cerberus-admin / cerberus-cli separation is non-negotiable.
- Cerberus and NovaPay identities/configs/state must never share tokens,
  MCP config, or Glue namespaces.
- ADRs document decisions in real chronological order, including wrong
  turns — this is portfolio signal, not overhead.
- infra/PLANNING_CHECKPOINT.yml must be read before proposing new infra.
