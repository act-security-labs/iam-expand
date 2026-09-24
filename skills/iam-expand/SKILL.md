---
name: iam-expand
description: Expand AWS IAM action wildcards such as s3:Get*, or every Action in a policy JSON, into the concrete actions they grant, and check whether an action name exists, by running iam-expand through npx with nothing installed. Use for "what does this wildcard or policy really allow" and for typo checks, instead of answering from memory.
license: MIT
compatibility: Node.js 22 or newer with npx; network access to registry.npmjs.org on first run.
metadata:
  author: act-security
  version: "0.1.0"
---

[iam-expand](https://github.com/act-security-labs/iam-expand) resolves IAM action patterns against the AWS action catalog in `@actsecurity/iam-data`, which publishes daily. Output is deduplicated and sorted.

## Run

```bash
npx -y @actsecurity/iam-expand@latest 's3:Get*Tagging'          # patterns as arguments
cat policy.json | npx -y @actsecurity/iam-expand@latest         # a policy: every Action and NotAction expanded in place
```

Two input modes, decided by the input shape:

| Input | Behaviour |
|---|---|
| Actions as arguments, or plain lines on stdin | One action per line |
| Valid JSON on stdin | Every `Action` and `NotAction` string or array is expanded in place and the whole document is printed back. Works on any JSON that contains policies, including a CloudFormation template or an `aws iam get-policy-version` response |

`--help` is the authoritative flag list. Only what it does not say is below.

## What `--help` does not say

- **Unknown actions vanish silently.** `s3:GetObjekt` is dropped with exit 0, so "no matches" and "typo" look identical. For "is this action real", pass `--invalid-action-behavior error` (exit 1 naming the offender) or `include` (keeps it in the list). `error` stops at the first unknown action; to find every one in a policy in one pass, diff the `include` output against the `remove` output. A locally installed version older than 0.11.85 exits 1 with `Invalid invalidActionBehavior: undefined` when the flag is omitted; passing it works everywhere.
- **A bare `*` stays `*`** unless `--expand-asterisk` is given, and on non-JSON stdin it is never expanded even with the flag. `--expand-asterisk '*'` as an argument expands to about 22,000 lines and loads the full dataset: write that to a file, never into the conversation.
- **`--invert` answers a different question.** On non-JSON input it prints every action that does *not* match. `--invert-not-actions` is the JSON cousin: it rewrites each `NotAction` into an explicit `Action` array, which is how a `NotAction` statement becomes reviewable. Both produce most of the catalog; treat them like the `*` case.
- **Piping from a slow producer** can hit the stdin first-byte timeout; raise it with `--read-wait-ms`.
- **Stale catalog.** npx reuses the `iam-data` cached with the tool's last release. If stderr warns the data package is over five days old, re-run once as `npx -y -p @actsecurity/iam-data@latest -p @actsecurity/iam-expand@latest iam-expand`. `--show-data-version` prints the date.

## Compose

Expand is the first stage of rightsizing: `cat policy.json | npx -y @actsecurity/iam-expand@latest | npx -y @actsecurity/iam-shrink@latest` removes every existing wildcard and rebuilds minimal ones. To search a policy for sensitive actions, expand first and grep the result; the expanded list is what IAM evaluates.

## Present

Tool output first, verbatim; past about 50 lines, write it to a file next to the input and show the path. Then your reading in its own section, labeled as yours. An empty result means no valid action matched, which includes the silent-drop case; confirm the pattern is spelled `service:Action*` before reporting "nothing matches".
