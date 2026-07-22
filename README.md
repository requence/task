# @requence/task

This package lets you start and monitor Requence tasks programmatically from a TypeScript / Bun application.

## Installation

```bash
npm install @requence/task
```

## Authentication

Every call to `createTask` requires an **access token** — either a personal access token or a scoped token created at the organization, group, or project level.

The token is resolved in this order:

1. `accessToken` option passed to `createTask()`
2. `REQUENCE_TASK_ACCESS_TOKEN` or `REQUENCE_ACCESS_TOKEN` environment variable
3. `requence.task.accessToken` or `requence.accessToken` in `package.json`

```bash
REQUENCE_ACCESS_TOKEN=your-token bun run index.ts
```

## Basic Usage

```typescript
import { createTask } from '@requence/task'

const result = await createTask({
  taskTemplate: 'my-template',
  input: { name: 'World' },
})

console.log(result.result) // The final output of the task
```

> **Note:** `await createTask(...)` waits for the **entire task** to finish. If you don't need to block, omit the `await` and use the returned task handle instead.

## Task Handle

`createTask` returns a **task handle** immediately. You can use it to access task metadata without waiting for the task to complete:

```typescript
const task = createTask({ taskTemplate: 'my-template', input: {} })

const taskId = await task.getTaskId()   // Resolves once the task is validated and started
const taskUrl = await task.getTaskUrl() // URL to view the task in the Requence UI

await task.abort('No longer needed')    // Abort the running task
await task.protect()                    // Protect the task from automatic cleanup

// Await the final result when you need it
const result = await task
```

The task handle is also an **AsyncIterable** — iterate over it to stream updates as the task executes.

### Confirming the Task Started (Cheap Monitoring)

By default, awaiting the result or iterating the handle starts full SSE monitoring. If you only need to confirm that the task passed input validation and started on the backend — without tracking every node — await `task.getTaskId()`:

```typescript
const task = createTask({ taskTemplate: 'my-template', input: { /* ... */ } })

try {
  const taskId = await task.getTaskId()
  console.log(`Task ${taskId} is running in the background`)
} catch (error) {
  console.error('Validation failed:', error.message)
}
```

`getTaskId()` resolves with the task ID once the task has passed validation and started on the backend, or rejects with a `TaskError` if the task cannot be created (e.g. schema validation failure). It never opens the SSE stream, so it stays cheap — and it never resolves with an ID for a task that failed to start.

## Options

| Option | Type | Default | Description |
|---|---|---|---|
| `taskTemplate` | `string` | — | **Required.** Name of the task template |
| `input` | `unknown` | — | Required when the template defines an input schema |
| `name` | `string` | — | Human-readable task name shown in the UI |
| `priority` | `number` | `2` | Priority `0` (lowest) to `4` (highest) |
| `accessToken` | `string` | — | Access token (falls back to env / config file) |
| `requireAck` | `boolean` | `false` | Enable delivery guarantee (see below) |
| `suppressBranchWarning` | `boolean` | `false` | Suppress the branch warning when running on a non-live branch |

## Result

When awaited, the task handle resolves to a result object:

```typescript
const result = await createTask({ taskTemplate: 'my-template', input: {} })

result.input            // The input you provided
result.result           // The final task output
result.taskId           // Unique task identifier
result.taskUrl          // URL to view the task in the Requence UI
result.getNodeData(alias)  // Get output from a specific node by alias
result.getNodeError(alias) // Get error from a specific node by alias
```

## Streaming Updates

The task handle is an `AsyncIterable` that emits updates as the task executes. Iterate over it with `for await`:

```typescript
const task = createTask({
  taskTemplate: 'my-template',
  input: { data: [1, 2, 3] },
})

for await (const update of task) {
  switch (update.type) {
    case 'taskStart':
      console.log(`Task ${update.taskId} started`)
      break
    case 'nodeStart':
      console.log(`Node ${update.node.alias ?? update.node.id} started`)
      break
    case 'nodeUpdate':
      console.log(`Node output:`, update.data)
      break
    case 'nodeError':
      console.log(`Node error:`, update.error)
      break
    case 'nodeDefer':
      console.log(`Node deferred:`, update.reason)
      break
    case 'nodeEnd':
      console.log(`Node ${update.node.alias ?? update.node.id} finished`)
      break
    case 'taskEnd':
      console.log(`Task completed:`, update.context.result)
      break
    case 'taskError':
      console.log(`Task failed:`, update.reason)
      break
    case 'taskAborted':
      console.log(`Task aborted:`, update.reason)
      break
  }
}
```

### `onUpdate` callback

Alternatively, pass a callback as the second argument to stream updates while still awaiting the final result:

```typescript
const result = await createTask(
  { taskTemplate: 'my-template', input: {} },
  (update) => {
    console.log(`[${update.type}]`, update.timestamp)
  },
)
```

### Update types

| Type | Description |
|---|---|
| `taskStart` | Task execution has begun. Contains `input` and `taskId`. |
| `nodeStart` | A node started processing. Contains `node` info (id, type, alias). |
| `nodeUpdate` | A node produced output. Contains `data` and `output` (named output, if any). |
| `nodeError` | A node encountered an error. Contains `error` message. |
| `nodeDefer` | A node has been deferred (waiting for an external callback). |
| `nodeEnd` | A node finished processing. |
| `taskEnd` | The task completed successfully. Contains final `result`. |
| `taskError` | The task failed. Contains `reason`. |
| `taskAborted` | The task was aborted. Contains `reason`. |

### Context on every update

Every update includes a `context` object with the current accumulated state:

```typescript
update.context.input                  // The task input
update.context.taskId                 // The task ID
update.context.result                 // Accumulated result (partial until taskEnd)
update.context.getNodeData(alias)     // Output from a specific node by alias
update.context.getNodeError(alias)    // Error from a specific node by alias
```

## Aborting a Task

Via the task handle:

```typescript
const task = createTask({ taskTemplate: 'my-template', input: {} })
await task.abort('No longer needed')
```

Or standalone, using a known task ID:

```typescript
import { abortTask } from '@requence/task'

await abortTask({
  taskId: 'some-task-id',
  reason: 'Cancelled by user',
})
```

## Protecting a Task

Protected tasks are excluded from automatic cleanup — useful for important results you want to keep indefinitely.

Via the task handle:

```typescript
const task = createTask({ taskTemplate: 'my-template', input: {} })
await task.protect()
```

Or standalone:

```typescript
import { protectTask } from '@requence/task'

await protectTask({ taskId: 'some-task-id' })
```

## Fetching a Task by ID

Retrieve the current state of any task:

```typescript
import { getTask } from '@requence/task'

const task = await getTask('some-task-id')
// or: getTask({ taskId: 'some-task-id', accessToken: '...' })

task.status        // 'SUCCESSFUL' | 'FAILED' | 'IDLE' | 'PENDING' | 'RUNNING' | 'STOPPED' | 'AWAITING_DELIVERY'
task.statusText    // Human-readable status description, if any
task.taskTemplate  // The task template name
task.name          // Human-readable task name
task.branch        // 'live' or a branch name

task.context.input                 // The task's input
task.context.result                // The task's result (if finished)
task.context.getNodeData(alias)    // Output from a specific node
task.context.getNodeError(alias)   // Error from a specific node
```

## Recreating a Task

Re-run an existing task with the same template and input:

```typescript
import { recreateTask } from '@requence/task'

const task = await recreateTask({
  taskId: 'some-task-id',
  name: 'My Retry',      // Optional — overrides the original name
  priority: 3,           // Optional — overrides the original priority
})

const result = await task
```

## Watching All Tasks (`watchTasks`)

Subscribe to real-time updates across **all tasks** — useful for dashboards, monitoring systems, or audit logs.

```typescript
import { watchTasks } from '@requence/task'

const stop = watchTasks({
  since: new Date(),
  onUpdate(update) {
    console.log(`[${update.type}] Task ${update.context.taskId}`)
  },
})

// Later — disconnect the watcher
stop()
```

`watchTasks` returns a stop function that is also an **AsyncIterable**:

```typescript
const watcher = watchTasks({ since: new Date() })

for await (const update of watcher) {
  console.log(update.type, update.context.taskId)

  if (shouldStop) {
    watcher() // stop watching
    break
  }
}
```

### `watchTasks` options

| Option | Type | Description |
|---|---|---|
| `since` | `Date` | **Required.** Only receive updates after this timestamp. |
| `accessToken` | `string` | Access token (falls back to env / config file) |
| `filter.only` | `string[]` | Filter to specific update types (e.g. `['taskEnd', 'taskError']`) |
| `onUpdate` | `function` | Callback for each update |
| `onConnect` | `function` | Called when the connection is established |
| `onReconnecting` | `function` | Called when the connection is being re-established |
| `onError` | `function` | Called on connection errors — receives `(status, reason)` |

### Incomplete updates

Each update from `watchTasks` includes an `incomplete` flag. When `true`, it means the watcher connected after the task had already started and the `taskStart` event was missed. Use this to decide whether to skip or partially process the update.

It also includes a `rootTaskId` field — the ID of the root task when the update belongs to a sub-task, or `null` for top-level tasks.

## Delivery Guarantee

By default, a task transitions immediately from `RUNNING` to `SUCCESSFUL` / `FAILED` / `STOPPED` when it finishes. If the process that started the task crashes at the exact moment the terminal event is emitted, the event can be lost.

Setting `requireAck: true` enables an explicit acknowledgement step:

1. When the task reaches its terminal state the backend holds it in **`AWAITING_DELIVERY`** instead of immediately finalising.
2. The SDK receives the terminal event over SSE and automatically sends an ACK back.
3. Only after the ACK is received does the task move to its real final status, and the exact delivery timestamp is recorded.

```typescript
const result = await createTask({
  taskTemplate: 'my-template',
  input: { /* ... */ },
  requireAck: true,
})
// Resolves only after the ACK has been confirmed
```

`requireAck` also works with `watchTasks` — when a task with `requireAck` reaches its terminal state, the watcher ACKs it automatically using the same access token.

## CLI — Generate Types

The package ships with a CLI that generates TypeScript types for your task templates, derived from the schemas defined in the Requence UI:

```bash
npx requence-task generate-types
```

This creates a `requence-env.d.ts` file in your project root. TypeScript picks it up automatically — `input`, `result`, and node data will be fully typed.

### Options

| Option | Default | Description |
|---|---|---|
| `--access-token` | — | Access token (falls back to env / config file) |
| `--outfile` | `requence-env.d.ts` | Name of the generated type file |
| `--outdir` | `.` | Directory to write the type file to |

The token resolution order for the CLI is the same as for `createTask`: option → `REQUENCE_TASK_ACCESS_TOKEN` / `REQUENCE_ACCESS_TOKEN` env var → `requence.task.accessToken` / `requence.accessToken` in `package.json`.

## Full Example

```typescript
import { createTask, watchTasks } from '@requence/task'

// Start a task and stream its progress
const task = createTask(
  {
    taskTemplate: 'invoice-processing',
    input: { invoiceUrl: 'https://...' },
    name: 'Invoice #1234',
    priority: 3,
  },
  (update) => {
    if (update.type === 'nodeUpdate') {
      console.log('Node result:', update.data)
    }
  },
)

const taskId = await task.getTaskId()
console.log('Started task:', taskId)

try {
  const result = await task
  console.log('Final result:', result.result)
} catch (error) {
  console.error('Task failed:', error.message)
}
```
