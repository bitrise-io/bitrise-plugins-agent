# Bitrise Agent

A Bitrise CLI plugin for agentic workflows

## Overview

This plugin helps to automate code reviews by using AI to analyze pull requests and provide feedback, suggestions, and potential issue detection. It uses the OpenAI and Anthropic API to generate insights about your code changes, with structured responses focusing on code quality, potential bugs, and performance issues.

## Use-cases

### Reviewer
- **PR Review**: Analyze GitHub and Bitbucket pull requests, writing summarries and flagging issues
- **CI Error Summarizer**: Summarizes errors in the CI Build and potential fixes

### Fixer
- Coming soon

### Developer
- Coming soon

## Installation

### Install plugin

```bash
bitrise plugin install --source https://github.com/bitrise-io/bitrise-plugins-agent.git
```

## CI Error Summarizer

The agent can summarize failed builds and surface errors and suggestions to fix the error. The agent will post the findings as an annotation to the build.

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

            # 5. Install AI Agent Plugin
            bitrise plugin install --source https://github.com/bitrise-io/bitrise-plugins-agent.git

            # 6. Run your AI reviewer (customize flags as needed)
            bitrise :agent build-summary \
              -m=gpt-4.1 \
              --log-level=debug

            echo "Done! PR reviewed."
```

</details>