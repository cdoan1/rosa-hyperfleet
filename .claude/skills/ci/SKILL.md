---
name: ci
description: Troubleshoot CI test failures by analyzing Prow job artifacts and logs. Accepts a PR URL, Prow job URL, or job name to investigate failures.
argument-hint: "[prow-url|pr-url|job-name]"
---

You are helping the user troubleshoot CI failures for the ROSA Regional Platform using the ci-troubleshoot agent.

## Input Handling

From `$ARGUMENTS`, parse the input which can be one of:

1. **Prow job URL** — Direct link to a specific job run:
   - `https://prow.ci.openshift.org/view/gs/test-platform-results/pr-logs/pull/openshift-online_rosa-regional-platform/<PR#>/<job-name>/<run-id>`
   - `https://prow.ci.openshift.org/view/gs/test-platform-results/logs/<job-name>/<run-id>`

2. **PR URL or number** — GitHub PR to check for CI failures:
   - `https://github.com/openshift-online/rosa-regional-platform/pull/123`
   - `#123` or just `123`

3. **Job name** — Find the latest failure for a specific job:
   - `nightly-ephemeral`
   - `nightly-integration`
   - `on-demand-e2e`
   - `terraform-validate`
   - `helm-lint`
   - `check-docs`
   - `check-rendered-files`

If no arguments provided, ask the user which CI failure they want to investigate.

## Job History URLs

For job names, find the latest failure by checking these history URLs:

| Job                    | History URL                                                                                                                                                      |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `check-docs`           | `https://prow.ci.openshift.org/job-history/gs/test-platform-results/pr-logs/directory/pull-ci-openshift-online-rosa-regional-platform-main-check-docs`           |
| `terraform-validate`   | `https://prow.ci.openshift.org/job-history/gs/test-platform-results/pr-logs/directory/pull-ci-openshift-online-rosa-regional-platform-main-terraform-validate`   |
| `helm-lint`            | `https://prow.ci.openshift.org/job-history/gs/test-platform-results/pr-logs/directory/pull-ci-openshift-online-rosa-regional-platform-main-helm-lint`            |
| `check-rendered-files` | `https://prow.ci.openshift.org/job-history/gs/test-platform-results/pr-logs/directory/pull-ci-openshift-online-rosa-regional-platform-main-check-rendered-files` |
| `on-demand-e2e`        | `https://prow.ci.openshift.org/job-history/gs/test-platform-results/pr-logs/directory/pull-ci-openshift-online-rosa-regional-platform-main-on-demand-e2e`        |
| `nightly-ephemeral`    | `https://prow.ci.openshift.org/job-history/gs/test-platform-results/logs/periodic-ci-openshift-online-rosa-regional-platform-main-nightly-ephemeral`             |
| `nightly-integration`  | `https://prow.ci.openshift.org/job-history/gs/test-platform-results/logs/periodic-ci-openshift-online-rosa-regional-platform-main-nightly-integration`           |

## Execution Flow

### For Prow Job URLs

Directly invoke the ci-troubleshoot agent with the URL:

```
Agent({
  description: "Investigate CI failure",
  subagent_type: "ci-troubleshoot",
  prompt: "Investigate this CI failure: <prow-url>"
})
```

### For PR URLs/Numbers

1. Extract the PR number from the URL or argument
2. Use `gh` to check CI status for that PR:

```bash
gh pr view <PR#> --json statusCheckRollup -q '.statusCheckRollup[] | select(.conclusion == "FAILURE") | .name, .targetUrl'
```

3. Present failed jobs to the user and ask which one to investigate, OR automatically investigate the first failure
4. Invoke the ci-troubleshoot agent with the selected Prow URL

### For Job Names

1. Fetch the job history page for that job using WebFetch
2. Parse the HTML to find the most recent failed run (look for red/failed status indicators)
3. Extract the Prow job URL for that failure
4. Invoke the ci-troubleshoot agent with that URL

## Agent Invocation

Use the Agent tool with `subagent_type: "ci-troubleshoot"` and provide clear context:

```
Agent({
  description: "Diagnose <job-name> failure",
  subagent_type: "ci-troubleshoot",
  prompt: "Investigate this CI test failure and provide a diagnosis with root cause and recommended fix: <prow-url>

Context: <any additional context about what the user is working on or expecting>"
})
```

## After Agent Returns

1. **Summarize the findings** — Extract the key diagnosis points
2. **Link to relevant code** — If the agent identified specific files, show file:line references
3. **Offer next steps**:
   - "Should I create a fix for this issue?"
   - "Would you like to see the full logs for any specific step?"
   - "Should I check if this is a known issue in recent PRs?"

## Example Interactions

### Example 1: Direct Prow URL
```
User: /ci https://prow.ci.openshift.org/view/gs/test-platform-results/pr-logs/pull/openshift-online_rosa-regional-platform/191/pull-ci-openshift-online-rosa-regional-platform-main-on-demand-e2e/1848584726732865536
Assistant: Investigating the on-demand-e2e failure...
[Invokes ci-troubleshoot agent]
```

### Example 2: PR Number
```
User: /ci #191
Assistant: Checking CI status for PR #191...
Found 2 failed jobs:
1. on-demand-e2e
2. terraform-validate

Investigating the first failure (on-demand-e2e)...
[Invokes ci-troubleshoot agent]
```

### Example 3: Job Name
```
User: /ci nightly-ephemeral
Assistant: Finding the latest nightly-ephemeral failure...
[Fetches job history, finds most recent failure]
Latest failure: https://prow.ci.openshift.org/view/gs/.../1848584726732865536
Investigating...
[Invokes ci-troubleshoot agent]
```

### Example 4: No Arguments
```
User: /ci
Assistant: What CI failure would you like to investigate?

You can provide:
- A Prow job URL
- A PR number (e.g., #191 or 191)
- A job name (e.g., nightly-ephemeral, on-demand-e2e, check-docs)
```

## Tips

- For PR-based investigations, automatically investigate all failed jobs if there are only 1-2 failures
- For job name lookups, always show the specific Prow URL found before investigating
- If the user provides a GitHub PR URL instead of a Prow URL, use `gh` to find the associated CI runs
- Include any context from the user's message (e.g., "the e2e test timed out") in the agent prompt