# 10 · Project — Typed CLI To-Do App

A small end-to-end project combining everything from Level 1: interfaces,
typed arrays/objects, classes, enums, and modules.

## What you'll build

A command-line to-do list that:

- Adds tasks
- Lists tasks (with done/pending status)
- Marks tasks done
- Deletes tasks
- Persists everything to a JSON file between runs

## Project layout

```text
todo_app/
    storage.ts
    todo.ts
    tsconfig.json
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "strict": true,
    "esModuleInterop": true,
    "outDir": "./dist"
  }
}
```

## storage.ts — persistence layer

```typescript
// storage.ts
import * as fs from "fs";

export interface Task {
  description: string;
  done: boolean;
}

const DB_PATH = "tasks.json";

export function loadTasks(): Task[] {
  if (!fs.existsSync(DB_PATH)) {
    return [];
  }

  try {
    const contents = fs.readFileSync(DB_PATH, "utf-8");
    return JSON.parse(contents) as Task[];
  } catch {
    return [];
  }
}

export function saveTasks(tasks: Task[]): void {
  fs.writeFileSync(DB_PATH, JSON.stringify(tasks, null, 2));
}
```

## todo.ts — CLI logic

```typescript
// todo.ts
import { Task, loadTasks, saveTasks } from "./storage";

function addTask(tasks: Task[], description: string): void {
  tasks.push({ description, done: false });
  console.log(`Added: ${description}`);
}

function listTasks(tasks: Task[]): void {
  if (tasks.length === 0) {
    console.log("No tasks yet.");
    return;
  }

  tasks.forEach((task, i) => {
    const status = task.done ? "x" : " ";
    console.log(`[${status}] ${i + 1}. ${task.description}`);
  });
}

function completeTask(tasks: Task[], index: number): void {
  const task = tasks[index - 1];
  if (!task) {
    console.log(`No task with number ${index}`);
    return;
  }
  task.done = true;
  console.log(`Marked task ${index} done.`);
}

function deleteTask(tasks: Task[], index: number): void {
  if (index < 1 || index > tasks.length) {
    console.log(`No task with number ${index}`);
    return;
  }
  const [removed] = tasks.splice(index - 1, 1);
  console.log(`Deleted: ${removed.description}`);
}

function main(): void {
  const tasks = loadTasks();
  const args = process.argv.slice(2);

  if (args.length === 0) {
    console.log("Usage: ts-node todo.ts [add <text> | list | done <n> | delete <n>]");
    return;
  }

  const [command, ...rest] = args;

  switch (command) {
    case "add":
      addTask(tasks, rest.join(" "));
      saveTasks(tasks);
      break;
    case "list":
      listTasks(tasks);
      break;
    case "done":
      completeTask(tasks, Number(rest[0]));
      saveTasks(tasks);
      break;
    case "delete":
      deleteTask(tasks, Number(rest[0]));
      saveTasks(tasks);
      break;
    default:
      console.log(`Unknown command: ${command}`);
  }
}

main();
```

## Running it

```bash
npx ts-node todo.ts add "Write Level 1 exercises"
npx ts-node todo.ts add "Review Level 2 outline"
npx ts-node todo.ts list
# [ ] 1. Write Level 1 exercises
# [ ] 2. Review Level 2 outline

npx ts-node todo.ts done 1
npx ts-node todo.ts list
# [x] 1. Write Level 1 exercises
# [ ] 2. Review Level 2 outline
```

## Stretch goals

- Add a `Priority` enum (`Low`/`Medium`/`High`) as a field on `Task` and sort
  the list by it.
- Add a `dueDate?: string` optional field and highlight overdue tasks.
- Add a Jest test for `storage.ts` (you'll formalize this properly with Jest
  in [Level 2](../level-2/07-testing-jest.md)).

Completing this project means you're ready for **Level 2 · Intermediate**.
