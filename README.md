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
| `reuse-session`| Existing Devin v3 session ID or URL to inject a message into. Accepts either a session ID or a full URL (e.g., `https://app.devin.ai/sessions/abc123`). When provided, sends a message to an existing session instead of creating a new one. Requires `org-id` and is mutually exclusive with `tags`. | false    |          |
| `wait-for-stopped-status` | If `true`, polls until `status_detail` is any non-`working` state. | false | `false` |
| `wait-minutes-max` | Maximum minutes to poll before timing out (only used when `wait-for-stopped-status` is enabled) | false | `20` |
| `api-version` | Devin API version. Only `v3` is supported; any other value fails fast. | false | `v3` |
| `advanced-mode` | V3 API advanced mode. Option: `analyze`. Requires `org-id`. See [Advanced Mode](#advanced-mode-v3-api) below. | false | |
| `session-links` | Session URLs or IDs to analyze (CSV or line-delimited). Required for `analyze` mode. When provided without `advanced-mode`, defaults to `analyze`. | false | |
| `org-id` | Devin organization ID. Required for v3 session creation, session reuse, and polling. | true | |
| `max-acu-limit` | Maximum ACU limit for the session. | false | |
| `playbook-id` | Playbook ID to use for the session. | false | |
| `child-playbook-id` | Playbook ID for child sessions. | false | |
| `bypass-approval` | If `true`, bypass approval for session creation. Requires `UseDevinExpert` permission. | false | `false` |
| `structured-output-schema` | JSON Schema describing the structured output Devin should produce. Accepts YAML or JSON (YAML 1.2 is a superset of JSON, so plain JSON also works). Defaults `structured_output_required` to `true`. Ignored when using `reuse-session`. See [Structured Output](#structured-output) below. | false | |
| `structured-output-required` | Flag passed as `structured_output_required` on session creation. Defaults to `true` when `structured-output-schema` is provided. Set to `false` to make structured output available but optional. Ignored when using `reuse-session`. | false | |

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
    org-id: ${{ vars.DEVIN_ORG_ID }}
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
    org-id: ${{ vars.DEVIN_ORG_ID }}
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
    org-id: ${{ vars.DEVIN_ORG_ID }}
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
    org-id: ${{ vars.DEVIN_ORG_ID }}

# Using a session URL (session ID is automatically extracted)
- name: Send Message to Existing Devin Session
  uses: aaronsteers/devin-action@v1
  with:
    reuse-session: 'https://app.devin.ai/sessions/c002a79b24b74f5b918ebc7dc6c5205b'
    prompt-text: |
      New task triggered at ${{ github.event.repository.updated_at }}
      Please process the following scope: all certified connectors
    devin-token: ${{ secrets.DEVIN_AI_API_KEY }}
    org-id: ${{ vars.DEVIN_ORG_ID }}
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
    org-id: ${{ vars.DEVIN_ORG_ID }}
    prompt-text: "Analyze commits and write a summary..."
    wait-for-stopped-status: true
```

**Wait on an existing session (multi-step):**

```yaml
- uses: aaronsteers/devin-action@v1
  id: create
  with:
    devin-token: ${{ secrets.DEVIN_AI_API_KEY }}
    org-id: ${{ vars.DEVIN_ORG_ID }}
    prompt-text: "Do something..."

# ... other steps ...

- uses: aaronsteers/devin-action@v1
  id: wait
  with:
    devin-token: ${{ secrets.DEVIN_AI_API_KEY }}
    org-id: ${{ vars.DEVIN_ORG_ID }}
    reuse-session: ${{ steps.create.outputs.session-id }}
    wait-for-stopped-status: true
```

When polling completes, the following additional outputs are available:

| Output | Description |
|--------|-------------|
| `status` | The terminal `status_detail` value when polling completes |
| `summary` | The last message from the session |

The action waits for any `status_detail` value other than `working`. See the [Devin API docs](https://docs.devin.ai/api-reference/v3/sessions/get-organizations-session) for the full list of possible status values.

## Advanced Mode (v3 API)

The action supports the Devin v3 API's advanced mode, which enables specialized session behaviors for automation workflows.

### API Version Selection

The action only supports the Devin v3 API. The `api-version` input defaults to `v3`; setting it to anything else fails before any Devin API call is made. All create, message, and polling calls require `org-id`.

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
    devin-token: ${{ secrets.DEVIN_TOKEN }}
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
    devin-token: ${{ secrets.DEVIN_TOKEN }}
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
    devin-token: ${{ secrets.DEVIN_TOKEN }}
    org-id: ${{ secrets.DEVIN_ORG_ID }}
    advanced-mode: analyze
    session-links: 'https://app.devin.ai/sessions/abc123'
    max-acu-limit: 5
```

## Structured Output

Use `structured-output-schema` to request that Devin produce structured output matching a JSON Schema. The schema is passed through as the `structured_output_schema` field on the v3 session creation request; it is ignored when using `reuse-session`.

When `structured-output-schema` is provided, the action defaults `structured_output_required` to `true`. This means Devin must call `provide_structured_output` with `is_final=true` before its turn ends. Set `structured-output-required: "false"` to make the tool available but optional; in that mode, it is not guaranteed to be called in a given turn.

The input accepts either YAML or JSON. YAML 1.2 is a strict superset of JSON, so existing JSON schemas are still valid. The input is validated up front: anything that doesn't parse to a top-level object fails the action before calling the API.

Once the session is running, the schema tells Devin what shape its "notepad" should take. Clients can then poll the session and read the latest structured output. See the [Devin structured output docs](https://docs.devin.ai/api-reference/structured-output) for more details.

### Inline Schema (YAML)

YAML is usually easier to read inside a workflow since you can skip the quotes and braces:

```yaml
- name: Run Devin with Structured Output
  uses: aaronsteers/devin-action@v1
  with:
    devin-token: ${{ secrets.DEVIN_AI_API_KEY }}
    org-id: ${{ vars.DEVIN_ORG_ID }}
    prompt-text: |
      Review this PR and keep the structured output up to date with any issues
      you find, suggestions you have, and your current approval status.
    structured-output-schema: |
      type: object
      required: [approved]
      properties:
        issues:
          type: array
          items:
            type: object
            properties:
              file: { type: string }
              line: { type: integer }
              type: { type: string }
              description: { type: string }
        suggestions:
          type: array
          items: { type: string }
        approved:
          type: boolean
```

### Inline Schema (JSON)

Plain JSON also works unchanged:

```yaml
- name: Run Devin with Structured Output
  uses: aaronsteers/devin-action@v1
  with:
    devin-token: ${{ secrets.DEVIN_AI_API_KEY }}
    prompt-text: "Review this PR and keep the structured output up to date."
    org-id: ${{ vars.DEVIN_ORG_ID }}
    structured-output-schema: |
      {
        "type": "object",
        "properties": {
          "approved": { "type": "boolean" },
          "suggestions": { "type": "array", "items": { "type": "string" } }
        },
        "required": ["approved"]
      }
```

### Schema From a File

For longer schemas, store them in a `.yaml` (or `.json`) file and pass the contents through a prior step output:

```yaml
- id: load-schema
  shell: bash
  run: |
    {
      echo "schema<<EOF"
      cat .github/schemas/pr-review.yaml
      echo "EOF"
    } >> "$GITHUB_OUTPUT"

- name: Run Devin with Structured Output
  uses: aaronsteers/devin-action@v1
  with:
    devin-token: ${{ secrets.DEVIN_AI_API_KEY }}
    org-id: ${{ vars.DEVIN_ORG_ID }}
    prompt-file: .github/prompts/pr-review.md
    structured-output-schema: ${{ steps.load-schema.outputs.schema }}
```

### Runner Requirements

Parsing tries the following in order, so any one of these on the runner is enough:

1. `yq` (mikefarah's — preinstalled on GitHub-hosted `ubuntu-latest`).
2. `python3` with PyYAML (also preinstalled on GitHub-hosted `ubuntu-latest`).
3. `jq` (JSON-only fallback — `jq` is already a hard dependency of this action).

On minimal self-hosted runners without `yq` or PyYAML, YAML input will be rejected and only JSON will parse.

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
          org-id: ${{ vars.DEVIN_ORG_ID }}
          github-token: ${{ secrets.MY_COMMENTS_PAT }}
          start-message: '🤖 **AI Help session starting...**'
```

## License

This project is licensed under the terms of the [MIT License](LICENSE).
