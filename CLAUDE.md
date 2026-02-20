# n8n Workflow SDK — Project Instructions

## Skill: n8n-workflow-sdk

Always invoke the **`n8n-workflow-sdk`** skill (via `Skill` tool) at the start of any session
involving workflow TypeScript authoring, modification, or MCP push/pull. It provides the
authoritative SDK API reference, correct node/connection patterns, and validation guidance.

```
skill: "n8n-workflow-sdk:n8n-workflow-sdk"
```

## SDK Bug Workaround (v0.2.0 — fix pending)

**Never use `.onTrue(A).onFalse(B)` on IF nodes.** This creates an `IfElseBuilderImpl`
which is silently skipped by `parseWorkflowCodeToBuilder()`, dropping all error branch
nodes from the output.

**Instead, always use `.output(index).to()` to wire IF node branches:**

```ts
// ✗ BROKEN — error branch nodes will be dropped on round-trip
get_Globals.onError(ifNode.onTrue(stopNode).onFalse(fallback))

// ✓ CORRECT — use output() to wire branches as plain node targets
get_Globals.output(1).to(ifNode)
ifNode.output(0).to(stopNode)    // true branch
ifNode.output(1).to(fallback)    // false branch

export default wf
  .add(trigger)
  .to(get_Globals)
  .to(nextNode)
  .add(ifNode)   // addSingleNodeConnectionTargets now works correctly
```

Also applies when **modifying existing TypeScript** retrieved via `get_workflow` MCP:
rewrite any `.onTrue().onFalse()` patterns to `.output()` before pushing back.

## MCP Setup

- Tools: `search_workflows`, `get_workflow` (returns TypeScript), `create_workflow`, `update_workflow` (accepts TypeScript)
- The `workflow(id, name)` call in the TypeScript identifies which workflow to update —
  never strip or change it during modifications or the push will fail with "Bad request"
- n8n base URL: `https://your-n8n-instance.com/`

## Post-Update: Open Workflow in Browser

After every successful `update_workflow` or `create_workflow` MCP call, navigate to the workflow
in the browser using Playwright:

```
mcp__playwright-mcp__playwright_navigate → https://your-n8n-instance.com/workflow/<workflow_id>
```

The workflow ID is the first argument to the `workflow(id, name)` call in the TypeScript.

If the user asks to visually inspect, double-check, or review the workflow, take a screenshot
using `mcp__playwright-mcp__playwright_screenshot` and analyze the canvas in your response.

If the screenshot shows a login screen, notify the user:
> "n8n is asking you to log in. Please log in and let me know when you're done."
Only take a follow-up screenshot after the user confirms, to verify the workflow is now visible.

## Switch/IF/Filter Node Conditions — Importable Format

The SDK generates **incorrect condition format** for Switch/IF/Filter nodes. The output uses
`rules.rules` instead of `rules.values`, omits `options.version: 2`, and omits `operator.name`.
This causes n8n's UI importer to throw `"Could not find property option"`.

**Always write Switch nodes manually with the correct format:**

```ts
const mySwitch = node({
  type: 'n8n-nodes-base.switch',
  version: 3.2,
  config: {
    name: 'My Switch',
    parameters: {
      mode: 'rules',                         // required — must be explicit
      rules: {
        values: [                            // key is 'values', NOT 'rules'
          {
            conditions: {
              options: {
                version: 2,                  // required — missing = import fails
                caseSensitive: true,
                leftValue: '',
                typeValidation: 'strict',
              },
              combinator: 'and',
              conditions: [{
                id: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx',  // unique UUID required
                operator: {
                  name: 'filter.operator.equals',            // name field required
                  type: 'string',
                  operation: 'equals',
                },
                leftValue: expr('{{ $json.field }}'),
                rightValue: 'value',
              }],
            },
            renameOutput: true,
            outputKey: 'case-label',
          },
        ],
      },
      options: {},
    },
  },
});
```

**Operator name pattern:** `filter.operator.<operation>` — e.g.
`filter.operator.equals`, `filter.operator.notEquals`, `filter.operator.contains`, etc.

**This same format applies to IF and Filter nodes** — any node using the `filter` parameter
type needs `options.version: 2`, `combinator`, and per-condition `id` + `operator.name`.

## `.to([a, b])` Fan-out

Passing an array creates **separate outputs** (output[0]→a, output[1]→b), not parallel
targets on the same output. This is by design.

## Settings Cleanup

When pushing TypeScript via the MCP workflow's Code node, it deletes `callerPolicy` and
`binaryMode` from settings before calling the n8n API — required to avoid "Bad request".
Do not manually add these back to the `workflow()` call options.
