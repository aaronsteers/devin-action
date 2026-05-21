# Live production test plan

Use this plan to validate a release candidate against the real Devin v3 API before merging or publishing it.

## Prerequisites

- Repository secret `DEVIN_TOKEN` is set to a v3-compatible Devin service user credential with `ManageOrgSessions`.
- Repository variable `DEVIN_ORG_ID` is set to the target Devin organization ID.
- The `Test Action` workflow has `workflow_dispatch` available on the default branch.
- The candidate branch contains the action changes to validate.

## Pre-merge workflow-dispatch test

1. Open GitHub Actions for the repository.
2. Select the `Test Action` workflow.
3. Choose `Run workflow`.
4. Select the candidate PR branch in the branch selector.
5. Set `run-live-tests` to `true`.
6. Keep the default prompt unless validating a specific prompt shape.
7. Keep `wait-for-stopped-status` as `false` for a fast session-creation smoke test. Set it to `true` only when validating polling behavior.
8. Start the workflow.

Expected result:

- The dry-run job passes.
- The unsupported-version fail-fast check passes by failing the `api-version: v1` step before any Devin API call.
- The live smoke-test job creates one real Devin v3 session using production secrets.
- The live job summary includes a non-empty session ID and session URL.
- The created session has the `devin-action-live-prod-test`, `workflow-<run_id>`, and `ref-<branch>` tags.

## Optional focused live checks

Run these only when the corresponding behavior changed:

1. `wait-for-stopped-status`: dispatch `Test Action` with `run-live-tests=true` and `wait-for-stopped-status=true`. Passes when polling finishes with a non-empty `status` output before `wait-minutes-max`.
2. Comment posting: dispatch one of the slash-command workflows against a throwaway issue or PR comment and confirm the start comment is updated with a Devin session link.
3. Session reuse: run the action from a temporary workflow or local branch with `reuse-session` set to the session ID from the live smoke test, and confirm the message lands in that same session.
4. Advanced analyze mode: run the action with `advanced-mode: analyze` and `session-links` pointing at a test session, then confirm the v3 request succeeds.

## Safety boundaries

- Use prompts that explicitly tell Devin not to modify repositories, create pull requests, or post external comments.
- Do not include customer data, private issue context, or secrets in test prompts.
- Use `max-acu-limit: "1"` for the live smoke test.
- Prefer testing on throwaway GitHub issues or draft PRs when comment posting is part of the validation.

## Release-candidate pass criteria

A release candidate is ready to merge when:

- Local YAML/action lint validation passes.
- Pull request CI passes.
- The workflow-dispatch live smoke test passes on the candidate branch.
- Any optional focused live checks relevant to the change pass.
