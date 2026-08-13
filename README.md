# @requence/task

Starts and monitors Requence tasks programmatically from a TypeScript / Bun application.

> **This README is a getting-started guide.** The reference — the task handle, streaming,
> watching every task, the delivery guarantee, the CLI — lives at
> **<https://docs.requence.cloud>**.

## Installation

```bash
npm install @requence/task
```

## Authentication

Every call to `createTask` requires an **access token** — either a personal access token or a
scoped token created at the organization, group, or project level.

The token is resolved in this order:

1. `accessToken` option passed to `createTask()`
2. `REQUENCE_TASK_ACCESS_TOKEN` or `REQUENCE_ACCESS_TOKEN` environment variable
3. `requence.task.accessToken` or `requence.accessToken` in `package.json`

```bash
REQUENCE_ACCESS_TOKEN=your-token bun run index.ts
```

## Starting a task

```typescript
import { createTask } from '@requence/task'

const result = await createTask({
  taskTemplate: 'my-template',
  input: { name: 'World' },
})

console.log(result.result) // The final output of the task
```

`await createTask(...)` waits for the **entire task** to finish. Omit the `await` and you get a
**task handle** back immediately — it exposes the task's id and URL, aborts and protects it, and
is an `AsyncIterable` of live updates:

```typescript
const task = createTask({ taskTemplate: 'my-template', input: {} })

const taskId = await task.getTaskId() // resolves once validated and started
const result = await task // …or await the final result
```

Generate types for `input`, `result` and node data from the template schemas you defined in the
UI:

```bash
npx requence-task generate-types
```

## Where to read next

| Topic                                                             | Page                             |
| ----------------------------------------------------------------- | -------------------------------- |
| Options, the task handle, abort, protect, `requireAck`            | [Starting Tasks][starting]       |
| Access tokens and their scopes                                    | [Authentication][auth]           |
| `for await`, `onUpdate`, the update types                         | [Streaming Updates][streaming]   |
| `watchTasks` across every task                                    | [Watching Tasks][watching]       |
| `getTask`, `recreateTask`                                         | [Fetching & Recreating][fetch]   |
| `requence-task generate-types`                                    | [CLI][cli]                       |

[starting]: https://docs.requence.cloud/execution/starting-tasks/
[auth]: https://docs.requence.cloud/execution/authentication/
[streaming]: https://docs.requence.cloud/execution/streaming/
[watching]: https://docs.requence.cloud/execution/watching-tasks/
[fetch]: https://docs.requence.cloud/execution/managing-tasks/
[cli]: https://docs.requence.cloud/execution/cli/
