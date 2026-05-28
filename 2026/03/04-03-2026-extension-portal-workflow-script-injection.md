# 04-03-2026 - Script injection in extension-portal GitHub Actions workflow

## Severity

**Critical** - Arbitrary code execution in CI pipeline with secret exfiltration.

## Security Related

**Yes**

## Impact

| Customers Impacted | Support Cases Raised | Customer Data Loss | Exposure Window | Report to Fix | Report to Full Hardening |
| :----------------: | :------------------: | :----------------: | :-------------: | :-----------: | :----------------------: |
|        None        |          1           |        None        |      ~17d       |     ~19h      |          ~49h            |

## Issue Summary

A script injection vulnerability was identified in the `PortSwigger/extension-portal` GitHub Actions workflow via HackerOne report #3585057. The "Extension URL" field in issue submissions was interpolated directly into shell `run:` blocks using `${{ }}` expressions without sanitisation. This allowed any GitHub user to execute arbitrary commands on the Actions runner by submitting a crafted issue, exfiltrating workflow secrets including a GitHub App private key, Jira API credentials, and Zoom webhook credentials.

The reporter demonstrated the exploit by submitting issue #274 to the public repository, which executed a payload that captured the runner's environment variables and posted a comment on the issue using the stolen `GITHUB_TOKEN`.

## Timeline

- **04-03-2026 15:48 UTC** – HackerOne report #3585057 submitted by external researcher, documenting script injection in `.github/workflows/process-created-issue.yml`.
- **04-03-2026 ~14:18 UTC** – Reporter's proof-of-concept payload had already executed via issue #274, exfiltrating workflow secrets from the Actions runner environment.
- **05-03-2026 10:07 UTC** – First remediation commit: replaced direct `${{ }}` expression injection in `run:` blocks with safe `env:` variable indirection.
- **05-03-2026 10:45 UTC** – URL validation moved earlier in the pipeline with strict regex enforcement directly in the extract step. Removed separate `validate_repo_url.py` script.
- **05-03-2026 10:58 UTC** – All GitHub Actions references pinned to commit SHAs instead of mutable version tags to prevent supply chain attacks.
- **05-03-2026 11:01 UTC** – JSON payload construction replaced with `jq` to prevent injection via string concatenation.
- **05-03-2026 11:10 UTC** – Error message verbosity reduced to limit information disclosure.
- **05-03-2026 11:40 UTC** – Build manager validation added to verify Java extensions use legitimate build systems.
- **05-03-2026 11:45 UTC** – Unused issue template removed to reduce attack surface.
- **05-03-2026 13:29 UTC** – Script arguments replaced with environment variables to prevent URL exposure in process lists.
- **06-03-2026 10:20 UTC** – Additional action SHA pinning for remaining references.
- **06-03-2026 13:36 UTC** – Critical architectural change: monolithic job split into isolated jobs with separated secret scopes. External repository checkout no longer shares a runner with Jira, GitHub App, or Zoom credentials.
- **06-03-2026 13:40 UTC** – Python `set_output` helper upgraded to use heredoc delimiters with random UUID boundaries, preventing output injection.
- **06-03-2026 13:43 UTC** – Shell-based comment parsing (`/reopen` command) replaced with `actions/github-script` to eliminate shell injection in comment processing.
- **06-03-2026 13:46 UTC** – Markdown escaping added to comment templates to prevent injection via error messages.
- **06-03-2026 13:50 UTC** – Dependabot enabled for GitHub Actions to provide ongoing dependency monitoring.
- **06-03-2026 16:46 UTC** – All hardening changes consolidated and merged via PR #288.
- **06-03-2026 17:03 UTC** – Runner hardening applied: `safer-runner-action` deployed in enforce mode with sudo, Docker disabled and risky GitHub subdomains blocked for the validation job.
- **06-03-2026 19:56 UTC** – Dependabot applied first automated action version updates.
- **Post-incident** – Compromised credentials (GitHub App private key, Jira API token, Zoom webhook credentials) rotated.

## Root Cause

The `process-created-issue.yml` workflow extracted a user-supplied "Extension URL" from issue form submissions and passed it unsanitised into shell `run:` blocks via GitHub Actions `${{ }}` expression syntax. While other fields (`title`, `author`) were sanitised through a `sanitizeText()` function, the URL field was passed through raw.

GitHub Actions expands `${{ }}` expressions at YAML template time, before bash execution begins. When the URL contained bash command substitution syntax (`$(...)`), the shell executed the embedded commands. The reporter's payload used `${IFS}` to bypass whitespace restrictions in the URL regex and `exec >/dev/null 2>&1` to suppress output, allowing the crafted URL to pass validation while executing arbitrary code.

The workflow ran with access to multiple secrets (Jira credentials, a GitHub App private key, and Zoom webhook credentials) on the same runner that checked out and processed untrusted external repositories. This single-job architecture meant that any code execution in the validation phase had access to all workflow secrets.

## Resolution and Recovery

Remediation was carried out in two phases over 05-06 March 2026:

**Phase 1 - Immediate fixes (05 March):**
1. Replaced all `${{ }}` expression interpolation in `run:` blocks with safe `env:` variable indirection
2. Moved URL validation earlier in the pipeline with strict regex enforcement
3. Pinned all GitHub Actions to commit SHAs
4. Replaced manual JSON string construction with `jq`

**Phase 2 - Architectural hardening (06 March):**
1. Split the monolithic submission job into isolated jobs with separated secret scopes - external repository checkout now runs on a dedicated job with only `GITHUB_TOKEN`, with no access to Jira, GitHub App, or Zoom credentials
2. Deployed `safer-runner-action` on the validation job to disable sudo, Docker, and block exfiltration via risky GitHub subdomains
3. Replaced shell-based comment parsing with JavaScript (`actions/github-script`)
4. Added heredoc delimiters with random UUID boundaries for output handling
5. Added markdown escaping to comment templates
6. Enabled Dependabot for ongoing action version monitoring

Compromised credentials were rotated following remediation.

## Corrective and Preventative Measures

1. Rotate all secrets that were exposed: GitHub App private key, Jira API token, Zoom webhook credentials. Verify revocation of old credentials.
2. Audit all other PortSwigger GitHub Actions workflows for expression injection patterns (`${{ }}` in `run:` blocks with untrusted input).
3. Establish a policy that workflows processing untrusted input must isolate external code execution from secrets using separate jobs.
4. Add `safer-runner-action` or equivalent runner hardening to all workflows that check out external repositories.
5. Pin all GitHub Actions to commit SHAs across the organisation and enable Dependabot for automated update tracking.
6. Review GitHub App permissions to ensure least-privilege: scope installation tokens to only the repositories and permissions required.
7. Implement workflow-level secret access reviews as part of the PR process for any changes to GitHub Actions configuration.
