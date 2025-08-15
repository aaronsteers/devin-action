# Devin Action

A reusable GitHub action which calls out to Devin.ai, creating a new Devin session with a given prompt or playbook. This action is designed for use with slash commands in PR comments and can automatically gather context from GitHub issues and comments.

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
| `start-message`| Custom message for the start comment                                       | false    |          |

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

This action is designed to work with slash commands in PR comments. The action will automatically gather context from the comment and/or issue.

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

When using the slash command dispatch workflow, the following commands are available:

- `/devin [prompt]` - Creates a general Devin session with optional custom prompt
- `/ai-fix` - Creates a Devin session focused on analyzing and fixing issues
- `/ai-ask` - Creates a Devin session focused on providing help and guidance

## License

This project is licensed under the terms of the [MIT License](LICENSE).
