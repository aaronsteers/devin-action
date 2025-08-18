# Devin Action

A reusable GitHub action which calls out to Devin.ai, creating a new Devin session with a given prompt or playbook. Designed to be used directly, or in slash commands. When invoked via slash commands, Devin can:
- post a response back as a comment
- update the current PR
- open an new PR if needed

## Features

- ✅ Creates Devin.ai sessions with custom prompts
- ✅ Supports triggering via slash commands in PR and issue comments
- ✅ Automatically gathers context from GitHub issues and comments
- ✅ Supports playbook macro integration
- ✅ Posts status updates and session links as comments
- ✅ Flexible input handling - use any combination of inputs

## Inputs

| Name           | Description                                                                 | Required | Default  |
|----------------|-----------------------------------------------------------------------------|----------|----------|
| `comment-id`   | Comment ID for context and reply chaining                                  | false    |          |
| `issue-number` | Issue number for context gathering                                          | false    |          |
| `playbook-macro` | Playbook macro for structured workflows - should start with '!' (e.g., !my_playbook) | false    |          |
| `prompt-text`  | Additional custom prompt text                                               | false    |          |
| `devin-token`  | Devin API Token (required for authentication)                              | true     |          |
| `github-token` | GitHub Token (required for posting comments and accessing repo context)    | false    |          |
| `start-message`| Custom message for the start comment                                       | false    | 🤖 **Starting Devin AI session...** |
| `tags`         | Comma-separated list of additional tags to apply to the Devin session (automatic tags are always added: `gh-actions-trigger` and `playbook-{macro-name}` if playbook-macro is provided) | false    |          |

## Usage

### Basic Example

```yaml
- name: Create Devin Session
  uses: aaronsteers/devin-action@v1
  with:
    prompt-text: "Please review this code and suggest improvements"
    devin-token: ${{ secrets.DEVIN_TOKEN }}
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

### Slash Command Example

This action is designed to work with slash commands in issue and PR comments. The action will automatically gather context from the comment and/or issue.

```yaml
- name: Run Devin from Comment
  uses: aaronsteers/devin-action@v1
  with:
    comment-id: ${{ github.event.comment.id }}
    issue-number: ${{ github.event.issue.number }}
    devin-token: ${{ secrets.DEVIN_TOKEN }}
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

### With Playbook Macro Integration

```yaml
- name: Run Devin with Playbook Macro
  uses: aaronsteers/devin-action@v1
  with:
    playbook-macro: "!fix-ci-failures"
    issue-number: ${{ github.event.issue.number }}
    prompt-text: "Additional context about the specific failure"
    devin-token: ${{ secrets.DEVIN_TOKEN }}
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

## Context Gathering

The action intelligently builds prompts by combining available context:

1. **Comment Context**: If `comment-id` is provided, includes the comment body and author
2. **Issue Context**: If `issue-number` is provided, includes issue title, description, and author  
3. **Playbook Macro Reference**: If `playbook-macro` is provided, instructs Devin to use the specified playbook macro
4. **Custom Prompt**: If `prompt-text` is provided, includes additional custom instructions

All provided inputs are concatenated to create a comprehensive prompt for Devin.

## Available Slash Commands

When using the slash command dispatch workflow here in this repo, the following commands are available:

- `/devin [prompt]` - Creates a general Devin session with optional custom prompt
- `/ai-fix` - Creates a Devin session focused on analyzing and fixing issues
- `/ai-ask` - Creates a Devin session focused on providing help and guidance

## Example Executions

For execution examples, check the [pinned issues](https://github.com/aaronsteers/devin-action/issues) here in this repo.

## Example Workflow [Slash Commands]

`.github/workflows/ai-help-command.yml`

```yml
name: AI Help Command

on:
  repository_dispatch:
    types: [ai-help-command]
  workflow_dispatch:
    inputs:
      issue-number:
        description: 'Issue number for context'
        required: true
        type: string
      comment-id:
        description: 'Comment ID that triggered the command (optional)'
        required: false
        type: string

permissions:
  contents: read
  issues: write
  pull-requests: read

jobs:
  ai-repro:
    runs-on: ubuntu-latest
    steps:
      - name: Run AI Help
        uses: aaronsteers/devin-action@v1
        with:
          comment-id: ${{ github.event.client_payload.slash_command.args.named.comment-id || inputs.comment-id }}
          issue-number: ${{ github.event.client_payload.slash_command.args.named.issue || inputs.issue-number }}
          playbook-macro: '!issue_help'
          devin-token: ${{ secrets.DEVIN_AI_API_KEY }}
          github-token: ${{ secrets.MY_COMMENTS_PAT }}
          start-message: '🤖 **AI Help session starting...**'
```

## License

This project is licensed under the terms of the [MIT License](LICENSE).
