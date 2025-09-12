# Triage Bot for Gitlab
```
Usage: triage-bot [options]

Node.js implementation of GitLab Triage for automated issue and MR management

Options:
  -V, --version               output the version number
  -n, --dry-run               Don't actually update anything, just print what would be done (default: false)
  -f, --policies-file <file>  Path to policies YAML file (default: "./.triage-policies.yml")
  --all-projects              Process all projects the token has access to (default: false)
  -s, --source <type>         The source type: projects or groups (default: "projects")
  -i, --source-id <id>        Source ID or path
  --resource-reference <ref>  Resource short-reference, e.g. #42, !33
  -t, --token <token>         GitLab API token
  -H, --host-url <url>        GitLab host URL (default: "https://gitlab.com")
  -r, --require <file>        Require a file before performing operations
  -d, --debug                 Print debug information (default: false)
  --init                      Initialize project with example policy file (default: false)
  --init-ci                   Initialize project with example .gitlab-ci.yml (default: false)
  -h, --help                  display help for command
```

## Configuration: .triage-policies.yml

The bot is fully driven by a YAML configuration file (default: ./.triage-policies.yml). This section documents the structure, fields and supported options.

Top-level fields:
- host_url (optional)
    - GitLab instance base URL. Overrides the CLI --host-url for the current run if present.
    - Example: https://gitlab.example.com
- bot_username (optional but recommended when using command framework)
    - The username of your bot account (without @). Used for comment-based commands like @yourbot ~label.
    - Example: platuserbot
- resource_rules (required)
    - A mapping of resource types to their rule sets and optional summary policies.
    - Supported resource types:
        - issue
        - merge_request
        - hooks (special set for webhook-driven commands)
    - Structure is described below per type.

Notes and defaults:
- The policy file is validated at startup. Invalid fields cause the run to fail with detailed errors.
- If host_url in the config differs from the CLI value, the config’s value is used for that run.
- Use --dry-run to preview all changes without modifying GitLab.

### Resource rules for issues and merge requests

resource_rules.issue and resource_rules.merge_request have the same structure:

- allow_on_hooks (optional, boolean)
    - When true, allows these rules to be evaluated during webhook-triggered runs.
- rules (required, array)
    - List of individual rule objects. Each rule has:
        - name (required, string)
            - Human-readable name of the rule.
        - conditions (optional, object)
            - Filters resources before actions are applied. If omitted, the rule applies to all loaded resources.
            - Supported condition fields:
                - state
                    - String. Common values: opened, closed, merged (MRs).
                - labels
                    - Array of labels that must be present. Special values:
                        - "None" means the resource must have zero labels.
                        - Brace list expansion is supported to match any in a set:
                            - help-needed::{triage,investigation,in progress}
                - forbidden_labels
                    - Array of labels that must NOT be present. Supports the same brace expansion as labels.
                - js
                    - A JavaScript boolean expression evaluated in the rule context.
                    - Commonly available variables:
                        - resource: the current issue/MR object
                        - hook_rest_body: the raw webhook payload (when run via hooks)
                    - Example:
                        - resource["assignees"].length === 0
                - date
                    - Time-based filter on resource dates.
                    - Fields:
                        - attribute: created_at or updated_at (string)
                        - condition: older_than or newer_than (string)
                        - interval_type: hours, days, weeks, months (string)
                        - interval: positive integer
                    - Example:
                        - attribute: updated_at
                          condition: older_than
                          interval_type: months
                          interval: 4
                - author_member (only meaningful in webhook context)
                    - Checks whether the webhook author is a member of a given source.
                    - Fields:
                        - source: group (currently supported)
                        - condition: member_of
                        - source_id: numeric group ID
        - actions (optional, object)
            - Executed when a resource matches the conditions. If omitted, the rule is skipped after filtering.
            - Supported actions:
                - labels
                    - Array of labels to add.
                    - Supports templated inputs and brace expansion. Example: {{...labels}}
                - remove_labels
                    - Array of labels to remove.
                - comment
                    - String (supports multiline). Supports simple template variables such as:
                        - {{author}}: mentions the resource author.
                    - Example:
                        - "{{author}}, this issue has no labels. Please add relevant labels."
                - merge (merge requests only)
                    - Merge orchestration:
                        - mergeWhenPipelineSucceeds: boolean
                            - Set true to queue merge when all checks pass.
                        - cancel: boolean
                            - Set true to cancel a previously queued merge.
                - extension
                    - Path to a JavaScript file (relative to the process working directory) to extend the behavior for the action. Useful for custom logic.

Examples:

- Require at least one label on issues and comment otherwise:
    - conditions:
      state: opened
      labels: [None]
      actions:
      comment: |
      {{author}}, this issue has no labels. Please add relevant labels.

- Enforce a single workflow label:
    - conditions:
      state: opened
      labels: [workflow::triage, workflow::investigation]
      forbidden_labels: [workflow::in progress]
      actions:
      remove_labels: [workflow::triage]

- Time-based stale labeling:
    - conditions:
      state: opened
      date:
      attribute: updated_at
      condition: older_than
      interval_type: months
      interval: 4
      forbidden_labels: [stale]
      actions:
      labels: [stale]
      comment: |
      This issue has been inactive and is now marked as stale.

### Hooks: command-driven actions via comments

resource_rules.hooks defines rules that execute in webhook context (for example, on issue or MR comments). This enables a “command framework” so users can perform actions by commenting.

Structure:
- resource_rules:
  hooks:
  - name: string
  on: string
  - Webhook event type to handle. Examples: note (comment), merge_request, issue.
  use_command_framework: boolean
  - Enables command parsing for the comment body.
  command: string
  - Command pattern to match. Use templated capture syntax to accept parameters.
  - Examples:
  - "{{...labels}}" — captures one or more label tokens (e.g., ~label1 ~label2)
  - "rm {{...labels}}" — remove labels
  - "ready" — a simple flag command
  conditions: object (optional)
  - Same fields as general conditions, plus typical webhook-oriented checks such as:
  - js with hook_rest_body to check author/assignees
  - author_member to validate group membership by source_id
  actions: object
  - Same actions as general rules. Typical ones for commands:
  - labels / remove_labels with captured tokens: ["{{...labels}}"]
  - comment
  - merge operations (e.g., cancel on unapproved)

Examples:
- Allow authors/assignees to add labels by commenting:
    - use_command_framework: true
      command: "{{...labels}}"
      on: note
      conditions:
      js: (hook_rest_body.issue || hook_rest_body.merge_request) && (hook_rest_body['user']['id'] === (hook_rest_body['issue']?.['author_id'] || hook_rest_body['merge_request']?.['author_id']) || (hook_rest_body['issue']?.['assignee_ids'] || hook_rest_body['merge_request']?.['assignee_ids'] || []).includes(hook_rest_body['user']['id']))
      actions:
      labels: ["{{...labels}}"]

- Restrict label removal to group members:
    - use_command_framework: true
      command: "rm {{...labels}}"
      on: note
      conditions:
      author_member:
      source: group
      condition: member_of
      source_id: <groupId>
      actions:
      remove_labels: ["{{...labels}}"]

- “Ready” command for MR authors/assignees:
    - use_command_framework: true
      command: "ready"
      on: note
      conditions:
      forbidden_labels: ["workflow::ready for review", "workflow::blocked", "unassigned"]
      actions:
      labels: ["workflow::ready for review"]
      remove_labels: ["workflow::triage"]
      comment: |
      The merge request will be marked as ready for review.

Command framework tips:
- Set bot_username at the top level and instruct users to mention the bot:
    - Example user comment: @platuserbot ~label1 ~label2
- The {{...labels}} capture accepts space-separated label tokens, typically written as ~label in GitLab comments.

### Summary policies (optional)

You can generate aggregated summaries per resource type.

Structure:
- resource_rules:
    <resource_type>:
      summaries:
        - name: string
          rules: array of rule fragments
            - Each has:
              - conditions: same as in normal rules
              - actions:
                  summarize: object (summary-specific configuration)
          actions:
            summarize: object
              - Defines how to combine and output the summary across rules that matched.

How it works:
- For each summary block, the bot loads resources and evaluates the embedded rules.
- For any rule that matched and includes actions.summarize, it collects a chunk of data.
- If the top-level summary.actions.summarize is present and any rule matched, it generates one combined summary output (for example, a comment or report depending on your summarize configuration).

Note: The exact output format of summarize is implementation-specific. Use it to group and report on sets of resources that satisfy different conditions.

### Templating and patterns

- Variable interpolation
    - {{author}}: Mention the issue/MR author in comments.
- Command captures
    - {{...labels}}: Spread-captures one or more label tokens from comment commands.
- Label set expansion
    - Use brace lists to express “any of these” concisely:
        - help-needed::{triage,investigation,in progress}
    - The engine matches if any expanded item fits.

### Sourcing and scope

When running:
- By default, triage-bot processes a single project:
    - --source projects --source-id <projectIdOrPath>
- You can process all accessible projects:
    - --all-projects
- You can target a single resource by short reference:
    - Issues: #<iid>, MRs: !<iid> via --resource-reference
- Group sources are supported for membership conditions in hooks. Full group-wide resource processing may vary by source type.

### Operational guidance and safety

- Dry run
    - Use --dry-run to verify which actions would be performed without making changes.
- Permissions
    - The provided token must have sufficient permissions to read and update issues/MRs, apply labels, post comments, and merge MRs if you use merge actions.
- Webhooks
    - For command-based actions, configure GitLab project webhooks to deliver events (e.g., notes) to the service that invokes this bot with hooks enabled.
    - Set allow_on_hooks: true under issue/merge_request to evaluate their rules during webhook handling when needed.
- Security
    - Limit powerful commands (e.g., label removal, merge control) using conditions like author_member to restrict to specific groups or authors/assignees.
    - Validate inputs captured from comments before using them in labels or other actions where appropriate (extensions can help).

### Minimal starter configuration
