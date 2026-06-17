# Coveo Bug Bounty Recon Write-up (Intigriti)

**Date:** June 2026
**Platform:** Intigriti
**Program:** Coveo
**Researcher:** Janos Palfi
**Status:** Recon / Informational findings

## A bit of context

I'm 25 and self-teaching penetration testing and offensive security. I've been working through TryHackMe, HackTheBox, OverTheWire, and building my own home lab. This was one of my first real bug bounty sessions on an actual live program.

I'm not going to pretend I'm some experienced researcher because I'm not. But I think that's kind of the point of writing this up. You have to start somewhere.

I picked Coveo on Intigriti because they specifically said they wanted researchers to look at their GitHub repos and CI/CD pipelines for supply chain issues. That felt like a realistic target for where I'm at right now.

## What Coveo was asking for

Their brief was pretty clear. They wanted:

- Client data exfiltration
- RCE in their infrastructure
- Supply chain compromise in their CI/CD pipelines

They also said their GitHub repos were in scope and encouraged researchers to look there. So that's where I started.

## What I actually did

I went to github.com/coveo and browsed their public repositories. I was looking for things like exposed secrets, hardcoded tokens, and misconfigured GitHub Actions workflows.

The repo that stood out immediately was `public-actions`. The description says it contains "GitHub Actions that can be reused in many of Coveo's repositories." From a supply chain perspective that's interesting because if something is wrong there, it affects everything that uses it.

I downloaded and read through these workflow files manually:

- `scorecard.yml`
- `dependency-review.yml`
- `java-maven-openjdk-dependency-review.yml`
- `java-maven-openjdk-dependency-submission.yml`
- `test-dependency-review.yml`

## What I found

### 1. The test workflow references a file that doesn't exist

In `test-dependency-review.yml`:

```yaml
uses: ./.github/workflows/dependency-review-v3.yml
```

There is no `dependency-review-v3.yml` in the repo. Only `dependency-review.yml` exists. So this test has never actually run. The security testing for the dependency review workflow is just silently broken.

This isn't a critical vulnerability but it does mean their automated security checks aren't catching anything in this workflow.

### 2. Some actions are pinned to version tags instead of commit hashes

```yaml
# These use version tags, which can be changed by whoever owns the action
uses: actions/checkout@v6.0.3
uses: actions/github-script@v9.0.0
uses: actions/cache@v5.0.5
uses: actions/upload-artifact@v7.0.1
```

Compare that to how they pin their security-critical actions:

```yaml
# These are pinned to a specific commit hash, much safer
uses: step-security/harden-runner@9af89fc71515a100421586dfdb3dc9c984fbf411
uses: ossf/scorecard-action@4eaacf0543bb3f2c246792bd56e8cdeffafb205a
```

The difference is that a version tag like `v6.0.3` can be moved to point to different code at any time. A commit hash is permanent. If someone compromised the `actions/checkout` repo and updated that tag, it would run in Coveo's pipelines without any change on their end.

### 3. Egress policy is set to audit, not block

In all three workflow files the Harden Runner action is configured like this:

```yaml
- name: Harden Runner
  uses: step-security/harden-runner@9af89fc71515a100421586dfdb3dc9c984fbf411
  with:
    egress-policy: audit
```

Audit mode means it logs outbound connections but doesn't block them. If a workflow got compromised, data could still leave the runner. Block mode with an allowed list of endpoints would actually prevent that.

Interestingly `scorecard.yml` already has an allowed-endpoints list defined, but the egress policy is still set to audit, so the blocking never kicks in.

### 4. Runner type comes from workflow input

In `java-maven-openjdk-dependency-review.yml`:

```yaml
runs-on: ${{ inputs.runs-on }}
```

Whatever calls this workflow decides what runner it runs on. Depending on how other repos validate this input, it could be a path toward targeting self-hosted runners.

## To be honest

Coveo already has an OpenSSF Scorecard of 8.3 which is pretty solid. They use Harden Runner, they pin their most sensitive actions to commit hashes, and they clearly think about security. The things I found are more about consistency than fundamental mistakes.

I didn't find any exposed secrets, hardcoded tokens, or the dangerous `pull_request_target` patterns I was looking for.

## What I took away from this

The big thing I realised is that knowing what a safe workflow looks like makes it a lot easier to spot when something is off. Before this session I wouldn't have known the difference between a pinned hash and a version tag or why it matters.

I also learned that doing recon properly takes time. I spent most of this session just reading and understanding the code rather than running tools. Which apparently is how it's supposed to work.

Next step is setting up a free Coveo organization and testing their actual API and search endpoints.

*This was a legitimate bug bounty session conducted within the Coveo program scope on Intigriti. All findings came from publicly available repositories.*
