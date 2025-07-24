# Bitrise Agent

A Bitrise CLI plugin for agentic workflows

## Overview

This plugin helps to automate code reviews by using AI to analyze pull requests and provide feedback, suggestions, and potential issue detection. It uses the OpenAI and Anthropic API to generate insights about your code changes, with structured responses focusing on code quality, potential bugs, and performance issues.

## Use-cases

| Tool | Scope | Description |
| ---- | ---- | ---- |
| pr-summary | Reviewer | Summarize PR changes using AI |
| ci-summary | Reviewer |  Summarize CI failures using AI |
| coming soon | Fixer | |
| coming soon | Developer | |

## Installation

### Install plugin

```bash
bitrise plugin install --source https://github.com/bitrise-io/bitrise-plugins-agent.git
```

### Configure plugin

You can add a review.bitrise.yml file to the root directory to configure the plugin

```yml
language: "en-US"               # language to use
tone_instructions: ""           # any additional instruction for the LLM on how to respond
reviews:
  profile: "chill"              # can be chill or assertive
  summary: true                 # should it generate summary
  walkthrough: true             # should it generate walkthrough
  collapse_walkthrough: true    # should the summary and walkthrough collapsed
  haiku: true                   # should it generate a haiku
  path_filters: ""              # todo
  path_instructions: ""         # todo
```

## CI Error Summarizer

The agent can summarize failed CI Builds and surface errors and suggestions to fix the error. The agent will post the findings as an annotation to the build.

You should always run it on the VM that you would like to analyze. This way we can make sure that the agent can pick up all the needed context to summarize correctly the error. Always add it as the last step in the workflow.

**Currently Bitrise is supported** as a CI Environment. The agent will pick up the environment variables during the CI execution time.

You should set the following environments:
- LLM_API_KEY
- GITHUB_TOKEN

```bash
$ bitrise :agent ci-summary -p openai -m gpt-4.1
```

<details>

<summary>🤖 Example Workflow</summary>

```yaml
workflows:
  primary:
    envs:
    - GITHUB_TOKEN: "$AI_PR_REVIEWER_GITHUB_TOKEN"
    - LLM_API_KEY: "$AI_PR_REVIEWER_OPENAI_API_KEY"
    steps:
    - activate-ssh-key@4:
    - (...)
    - script@1.2.1:
        title: Review Build Error
        is_always_run: true
        run_if: enveq "BITRISE_BUILD_STATUS" "1"
        inputs:
        - content: |-
            #!/bin/bash
            set -e

            bitrise plugin install --source https://github.com/bitrise-io/bitrise-plugins-agent.git

            bitrise :agent ci-summary \
              -m=gpt-4.1 \
              --log-level=debug
```

</details>

## PR Summary

The agent can summarize Pull Request changes and surface errors and line feedback. The agent will post the findings as a comment to GitHub.

You should always run it in an environment where the agent can reach the git repository. This way we can make sure that the agent can pick up all the needed context to summarize correctly the Pull Request and find any errors in it. Ideally you create a parallel workflow on a small linux machine to execute the agent once you have pull the git repository.

You should set the following environments:
- LLM_API_KEY
- GITHUB_TOKEN

```bash
$ bitrise :agent pr-summary -p openai -m gpt-4.1 -b master -r github --repo bitrise-io/bitrise-plugins-agent --pr 36
```

<details>

<summary>🤖 Example Workflow</summary>

**Make sure to set this report "AI Review" as non-blocking on GitHub for merges**

```yaml
workflows:
  ai_pr_summary:
    triggers:
      pull_request:
      - target_branch: '*'
        source_branch: '*'
        changed_files: '*'
    status_report_name: 'AI Review'
    meta:
      bitrise.io:
        machine_type_id: g2.linux.medium
        stack: linux-docker-android-22.04
    envs:
    - GITHUB_TOKEN: $ADD_GITHUB_ACCESS_TOKEN
    - LLM_API_KEY: $ADD_OPENAI_ACCESS_TOKEN
    steps:
    - activate-ssh-key@4:
        run_if: '{{getenv "SSH_RSA_PRIVATE_KEY" | ne ""}}'
    - git-clone@8.4.0: {}
    - script@1.2.1:
        title: Generate AI Review for PR
        inputs:
        - content: |-            
            #!/bin/bash
            set -e

            # Parse repository name from repo URL (works for SSH & HTTPS)
            REPO_URL="${GIT_REPOSITORY_URL}"
            REPO=$(echo "$REPO_URL" | sed -E 's#(git@|https://)([^/:]+)[/:]([^/]+)/([^.]+)(\.git)?#\3/\4#')

            # 1. Unshallow the repo if it's a shallow clone (safe to run even if already full)
            git fetch --unshallow || true

            # 2. Fetch all branch refs (this ensures both the PR and the target/destination branch are present)
            git fetch origin

            # 3. Fetch both relevant branches explicitly for safety (redundant but safe)
            git fetch origin "$BITRISEIO_GIT_BRANCH_DEST"
            git fetch origin "$BITRISE_GIT_BRANCH"

            # 4. Create/reset local branches to match the remote
            git checkout -B "$BITRISEIO_GIT_BRANCH_DEST" "origin/$BITRISEIO_GIT_BRANCH_DEST"
            git checkout -B "$BITRISE_GIT_BRANCH" "origin/$BITRISE_GIT_BRANCH"

            # (Optionally: check out the PR branch if that is the branch you want to analyze)
            git checkout "$BITRISE_GIT_BRANCH"

            bitrise plugin install --source https://github.com/bitrise-io/bitrise-plugins-agent.git

            bitrise :agent pr-summary \
              -m=gpt-4.1 \
              -c "${GIT_CLONE_COMMIT_HASH}" \
              -b "${BITRISEIO_GIT_BRANCH_DEST}" \
              -r github \
              --pr "${BITRISE_PULL_REQUEST}" \
              --repo "${REPO}" \
              --log-level debug"
```