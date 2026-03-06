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
| `tags`         | Additional tags to apply to the Devin session (supports CSV or line-delimited format). Automatic tags are always added: `gh-actions-trigger` and `playbook-{macro-name}` if playbook-macro is provided. Cannot be used with `reuse-session`. | false    |          |
| `reuse-session`| Existing Devin session ID or URL to inject a message into. Accepts either a session ID or a full URL (e.g., `https://app.devin.ai/sessions/abc123`). When provided, sends a message to an existing session instead of creating a new one. Mutually exclusive with `tags`. | false    |          |
| `wait-for-stopped-status` | If `true`, polls until `status_enum` is any non-`working` state. | false | `false` |
| `wait-minutes-max` | Maximum minutes to poll before timing out (only used when `wait-for-stopped-status` is enabled) | false | `20` |
| `api-version` | Devin API version: `auto` (default), `v1`, or `v3`. When `auto`, resolves to v3 if v3-only features are used; otherwise v1. Setting `v1` with v3-only features errors. | false | `auto` |
| `advanced-mode` | V3 API advanced mode. Options: `analyze`, `create`, `improve`, `batch`, `manage`. Requires `org-id`. See [Advanced Mode](#advanced-mode-v3-api) below. | false | |
| `session-links` | Session URLs or IDs to analyze (CSV or line-delimited). Required for `analyze` mode. When provided without `advanced-mode`, defaults to `analyze`. | false | |
| `org-id` | Devin organization ID. Required when using v3 API features (`advanced-mode` or `session-links`). | false | |
| `max-acu-limit` | Maximum ACU limit for the session (v3 API only). | false | |
| `child-playbook-id` | Playbook ID for child sessions (v3 API only). Required for `batch` and `improve` modes. | false | |
| `bypass-approval` | If `true`, bypass approval for batch session creation (v3 batch mode only). Requires `UseDevinExpert` permission. | false | `false` |

## Session Tagging

This action automatically adds tags to Devin sessions for better monitoring and searching:

### Automatic Tags

- **`gh-actions-trigger`** - Always added to identify sessions triggered from GitHub Actions
- **`playbook-{macro-name}`** - Added when `playbook-macro` is provided (e.g., `playbook-issue-ask` for `!issue_ask`)

### Additional Tags

You can provide additional tags using the `tags` input parameter, which supports both formats:

**CSV format:**

```yaml
tags: 'priority-high,bug-fix,frontend'
```

**Line-delimited format:**

```yaml
tags: |
  priority-high
  bug-fix
  frontend
```

**Mixed format (CSV per line):**

```yaml
tags: |
  priority-high,urgent
  bug-fix,frontend
  team-alpha
```

## Usage

### Basic Example

```yaml
- name: Create Devin Session
  uses: aaronsteers/devin-action@v1
  with:
    prompt-text: "Please review this code and suggest improvements"
    devin-token: ${{ secrets.DEVIN_TOKEN }}
    github-token: ${{ secrets.GITHUB_TOKEN }}
    tags: 'code-review,improvement'
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
    tags: |
      ci-failure
      urgent
```

**Tags applied:** `gh-actions-trigger`, `playbook-fix-ci-failures`, `ci-failure`, `urgent`

### Injecting a Message into an Existing Session

Instead of creating a new session, you can inject a message into an existing Devin session using the `reuse-session` input. This is useful for long-running workflows that need to send multiple messages to the same session.

You can provide either a session ID or a full session URL:

```yaml
# Using a session ID
- name: Send Message to Existing Devin Session
  uses: aaronsteers/devin-action@v1
  with:
    reuse-session: 'c002a79b24b74f5b918ebc7dc6c5205b'
    prompt-text: |
      New task triggered at ${{ github.event.repository.updated_at }}
      Please process the following scope: all certified connectors
    devin-token: ${{ secrets.DEVIN_AI_API_KEY }}

# Using a session URL (session ID is automatically extracted)
- name: Send Message to Existing Devin Session
  uses: aaronsteers/devin-action@v1
  with:
    reuse-session: 'https://app.devin.ai/sessions/c002a79b24b74f5b918ebc7dc6c5205b'
    prompt-text: |
      New task triggered at ${{ github.event.repository.updated_at }}
      Please process the following scope: all certified connectors
    devin-token: ${{ secrets.DEVIN_AI_API_KEY }}
```

**Note:** The `reuse-session` input is mutually exclusive with `tags`. When injecting a message into an existing session, tags cannot be specified since they are only applicable when creating new sessions.

### Waiting for Session Completion

Use `wait-for-stopped-status` to poll the Devin session until it reaches any non-`working` state. This is useful for workflows that need the session's output before proceeding (e.g., posting a summary, gating subsequent steps).

**Wait during session creation (single step):**

```yaml
- uses: aaronsteers/devin-action@v1
  id: devin
  with:
    devin-token: ${{ secrets.DEVIN_AI_API_KEY }}
    prompt-text: "Analyze commits and write a summary..."
    wait-for-stopped-status: true
```

**Wait on an existing session (multi-step):**

```yaml
- uses: aaronsteers/devin-action@v1
  id: create
  with:
    devin-token: ${{ secrets.DEVIN_AI_API_KEY }}
    prompt-text: "Do something..."

# ... other steps ...

- uses: aaronsteers/devin-action@v1
  id: wait
  with:
    devin-token: ${{ secrets.DEVIN_AI_API_KEY }}
    reuse-session: ${{ steps.create.outputs.session-id }}
    wait-for-stopped-status: true
```

When polling completes, the following additional outputs are available:

| Output | Description |
|--------|-------------|
| `status` | The terminal `status_enum` value when polling completes |
| `summary` | The last message from the session |

The action waits for any `status_enum` value other than `working`. See the [Devin API docs](https://docs.devin.ai/api-reference/v1/sessions/retrieve-details-about-an-existing-session) for the full list of possible status values.

## Advanced Mode (v3 API)

The action supports the Devin v3 API's advanced mode, which enables specialized session behaviors for automation workflows.

### API Version Selection

The `api-version` input controls which Devin API endpoint is used:

- **`auto`** (default): Automatically selects v3 when v3-only features (`advanced-mode`, `session-links`, `max-acu-limit`) are provided; otherwise uses v1.
- **`v1`**: Forces the v1 API. Will error if any v3-only features are also provided.
- **`v3`**: Forces the v3 API. Requires `org-id`.

When `advanced-mode` or `session-links` is provided, the action automatically uses the v3 API endpoint (equivalent to `api-version: auto` behavior).

### Available Modes

| Mode | Description | Required Parameters |
|------|-------------|---------------------|
| `analyze` | Analyze existing Devin sessions to extract insights | `session-links` |
| `create` | Create a new playbook based on session analysis | None (optional: `session-links`) |
| `improve` | Improve an existing playbook based on feedback | None |
| `batch` | Start multiple Devin sessions for a list of tasks | None |
| `manage` | Manage knowledge | None |

### Prerequisites

The v3 API requires:

1. **A v3-compatible API token** — must be a service user credential (not a PAT). Requires `ManageOrgSessions` permission. Advanced mode additionally requires `UseDevinExpert`.
2. **Organization ID** via the `org-id` input
3. **Team or Enterprise plan** — the v3 API and Advanced Mode are available on Team and Enterprise plans.

### Analyze Sessions Example

This is particularly useful for automated triage workflows where one Devin session needs to inspect the conversation history of another session:

```yaml
- name: Analyze Devin Session
  uses: aaronsteers/devin-action@v1
  with:
    prompt-text: "Triage the linked session and identify the root cause of the reported issue."
    devin-token: ${{ secrets.DEVIN_V3_TOKEN }}
    github-token: ${{ secrets.GITHUB_TOKEN }}
    org-id: ${{ secrets.DEVIN_ORG_ID }}
    advanced-mode: analyze
    session-links: |
      https://app.devin.ai/sessions/abc123
      https://app.devin.ai/sessions/def456
    tags: 'session-triage,automated'
```

### Auto-detect Analyze Mode

When `session-links` is provided without `advanced-mode`, the action automatically defaults to `analyze` mode:

```yaml
- name: Analyze Session (auto-detect)
  uses: aaronsteers/devin-action@v1
  with:
    prompt-text: "Review this session and summarize what happened."
    devin-token: ${{ secrets.DEVIN_V3_TOKEN }}
    org-id: ${{ secrets.DEVIN_ORG_ID }}
    session-links: 'https://app.devin.ai/sessions/abc123'
```

### ACU Budget Control

Use `max-acu-limit` to cap the compute budget for v3 sessions:

```yaml
- name: Budget-limited Session
  uses: aaronsteers/devin-action@v1
  with:
    prompt-text: "Quick analysis of this session."
    devin-token: ${{ secrets.DEVIN_V3_TOKEN }}
    org-id: ${{ secrets.DEVIN_ORG_ID }}
    advanced-mode: analyze
    session-links: 'https://app.devin.ai/sessions/abc123'
    max-acu-limit: 5
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
