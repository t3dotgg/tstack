---
name: polish-docs
description: Use when the user says "polish docs", "polish the docs", or "polish-docs", or when you decide a README, guide, or other documentation file should be written for people to read. Neon writes or rewrites the text, then sol-loop checks it for accuracy. Not for PR descriptions, commit messages, or PR comments.
---

# polish-docs

Use this workflow to write documentation for people to read. Neon, the
writing model on Antigravity (`agy`), writes the text. GPT Sol then checks
the facts in a `sol-loop`.

Neon owns the wording, structure, and style. Do not write or edit the prose
yourself, even to fix a typo. Your job is to collect the facts, brief Neon,
run the accuracy check, and carry findings between Sol and Neon.

## Modes

- **Rewrite.** A document already exists. Neon reads it and rewrites it from
  scratch in its own words. The old text is a source of facts, not a draft to
  tidy up. Keep every true fact unless the user says to cut it.
- **Write.** No document exists, or the user or you ask for new text. Neon
  writes it from the facts you collect.

If the request does not make the mode clear, use Rewrite when the target file
exists and Write when it does not.

## 1. Gather facts

Read the code, configs, existing docs, PRs, and threads the document depends
on. Write down what Neon needs as plain facts, not sentences you hope it will
keep:

- Target path and mode.
- Audience and what they already know. If you don't know, assume a reader who
  is new to the project.
- Purpose: what the reader should be able to do after reading it.
- Facts, numbers, commands, and names, each with its source (file:line, URL,
  or command output).
- The meaning of every internal reference. "Decision 4.2", "anchor b", ticket
  numbers, and team shorthand mean nothing to Neon or to the reader. Spell out
  what each one decided or describes.
- Anything the user said must stay, go, or change.

In Rewrite mode, save a copy of the original to your scratch directory before
Neon starts. It may be untracked or have uncommitted edits, and Sol needs it
to check for dropped facts.

Do not hand Neon a draft. Your phrasing will leak into its text.

## 2. Start Neon

In T3 Code, call `orchestrator_capabilities` and pick the latest Neon model
unless the user named a version. Do not hardcode an ID. Start it with
`delegate_task` (`role: "implementation"`, `mode: "async"`, a new
`clientRequestId` per call). Save the `taskId`.

Outside T3, use a native Neon subagent if your harness has one. If no route
runs Neon, for example because Antigravity is not signed in, report the
blocker. Do not substitute another model, and do not write the document
yourself.

Before each Neon task, save a copy of the target, or note that it does not
exist yet, and record `git status --short -uall`. If Neon fails or times
out, cancel it and confirm it stopped. Restore the target from your copy, or
delete it if it did not exist, then restart Neon or report the blocker. Do
not revert other paths yourself. If `git status` shows Neon touched any,
list them for the user.

Draft brief (Neon gets no history):

```
You are writing documentation for people to read. You own the wording,
structure, and style.
Repo: <abs path> Target: <path>
Mode: <one of>
  Rewrite: read the existing file as source material only. Rebuild it from
  scratch in your own words and structure. Keep every true fact unless told
  to cut it. Do not copyedit the old text.
  Write: write new content from the facts below.
Audience: <who, and what they already know> Purpose: <what they can do after>
Facts and sources: <list>
Internal references explained: <list>
Must keep / cut / change: <list>
Edit only the target path. Do not commit, push, comment on PRs, or start
subagents. When you finish, request the accuracy check in your reply. The
orchestrator runs it with a separate reviewer and sends you the findings.
<Writing rules section, below, verbatim with examples>
Reply with: the path, a short summary of the structure you chose, and
"Ready for accuracy check".
```

When Neon replies, check the draft yourself: Neon edited only the target
path, acronyms follow the audience rule, and no internal label appears
unexplained. If it edited other paths, list them for the user. If something
else fails, send it back to Neon with the fix brief (step 4), listing what
your check found. Do not fix it yourself. Run these checks before the
accuracy check, so Sol reviews the final text.

## Writing rules

Include this whole section in the brief word for word, examples too.

- Write in plain, natural language a person would use. No filler, hype, or
  stock phrases.
- Assume the reader does not know the project unless the brief says
  otherwise.
- Spell out every acronym the first time it appears, with the short form in
  parentheses. After that, the short form is fine. Skip this only for
  acronyms the brief says the audience already knows.
- Explain product-specific and team-specific terms instead of only naming
  them. Never refer to a decision, ticket, or design option by its label
  alone. Say what it decided or describes.
- Keep commands, code, paths, and numbers exactly as they appear in the
  sources.

Bad:

```
The EKS workload uses IRSA to obtain STS credentials for accessing S3 via a VPC endpoint. The ALB is provisioned through the LBC using an Ingress resource, with TLS termination managed by ACM. Observability is provided through OTEL, with metrics exported to AMP and traces forwarded to X-Ray. The CI/CD pipeline uses OIDC federation to assume an IAM role, enabling IaC deployments through TF without requiring static AK/SK credentials. Ensure SCPs, SGs, NACLs, and KMS policies are configured correctly to prevent unintended access.
```

Good:

```
The application runs on Amazon Elastic Kubernetes Service (EKS) and uses IAM Roles for Service Accounts (IRSA) to obtain temporary credentials from AWS Security Token Service (STS).

These credentials allow the application to access Amazon Simple Storage Service (S3) through a private Virtual Private Cloud (VPC) endpoint.

Incoming traffic is managed by an Application Load Balancer (ALB), which is provisioned automatically by the AWS Load Balancer Controller. Transport Layer Security (TLS) certificates are managed through AWS Certificate Manager (ACM).

Application monitoring uses OpenTelemetry to collect metrics and traces, which are exported to Amazon Managed Service for Prometheus and AWS X-Ray, respectively.
```

Bad:

```
Decision 4.2 changed the scope. Pivoted to design anchor b.
```

Good:

```
The decision to use an open read-only role available to all users changed the scope for securing the application with canned user roles. Pivoted to alternative design that expanded the application to include custom roles with more granular permissions.
```

## 3. Accuracy check

When Neon's draft passes your checks, run a `sol-loop` on it. Read
[sol-loop](../sol-loop/SKILL.md) and follow its model choice, ledger, and
waiting rules. The overrides below take precedence.

- The change under review is the document, not a code diff. Approval covers
  the current document.
- Any fix to a blocker or should needs a new round, even though it is a doc
  change. A round with only nits ends the loop. Nit fixes that change only
  wording need no new round.
- Before sending fixes to Neon, save a copy of the document Sol just
  reviewed, so you can show Sol what changed. `git diff` does not show an
  untracked file.
- Budget: 3 rounds. Extend to 5 while a blocker stays open with evidence, or
  when the user asks.
- Sol reports. Neon makes every fix in its own words (step 4). You never edit
  the text, and you do not answer findings on Neon's behalf except to decline
  out-of-scope ones.

Round 1 brief (Sol gets no history):

```
Review only. Do not edit, commit, push, comment on PRs, or start subagents.
Repo: <abs path> Document: <path> Audience: <who> Purpose: <what>
Sources: <list from the Neon brief>
Rewrite mode only: Original: <saved copy path> Authorized cuts: <list>
Check every factual claim in the document against the repo and the sources:
claims, statistics, numbers, commands, names, paths, and links. Anything a
reader could act on and get wrong is in scope.
Wording, structure, tone, and style are out of scope. Do not flag sections
as unneeded. Flag wording only if it would lead a reader to the wrong action,
and grade it by that consequence.
A claim you cannot verify is not approved. Report it as should.
blocker = false claim, or wrong command, number, path, or link.
should = claim the sources do not support, missing fact the purpose needs,
wording that leads a reader to the wrong action, or (Rewrite mode) a true
fact from the original dropped without an authorized cut. nit = the rest.
Reply:
VERDICT: APPROVE or CHANGES
S1 [blocker|should|nit] <path>:<line> <claim>. Fix: <what is wrong and what the sources say> (stable ids)
```

Later rounds: send the same brief, the ledger (`fixed:` / `disagree:` /
`declined:` + evidence), and what changed since the last round. Tell Sol:
open ids and changed text only. Mark each old id `resolved` or
`still open: <reason>`.

## 4. Fixes go to Neon

Check each finding against the sources yourself, as `sol-loop` says. Send
the ones you agree with to Neon with this fix brief: a fresh `delegate_task`
in T3, or the native Neon route you used in step 2. Do not resend the draft
brief.

```
You own the wording, structure, and style of <path>.
Repo: <abs path> Audience: <who, and what they already know>
Fix only these findings from a check of the document, in your own words.
Leave the rest of the text alone. Do not start subagents, commit, or edit
any other file.
Findings: <ids, claims, and what the sources say>
<Writing rules section, verbatim with examples>
Reply with: what you changed for each id, or why you disagree, with evidence.
```

Record Neon's answers in the ledger, rerun your checks from step 2, and start
the next Sol round if one is due. A disagreement Neon backs with evidence is
a `disagree:` entry for Sol to weigh.

## 5. Report

Stop when Sol approves the current document, or one that differs from it
only by wording-only nit fixes, with no blocker or should open. Also stop
when `sol-loop`'s stuck rule applies. Budget end is not approval.

Report: path, mode, Neon model, Sol model, who ran the `sol-loop`, rounds,
findings fixed and declined, claims left unverified, and anything the user
should check. Do not commit or push unless the user asked.
