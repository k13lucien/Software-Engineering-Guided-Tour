# Understanding AI Coding Agents

A practical guide to how AI coding agents work — and how to use them better.

---

## 1. The agent loop

An AI coding agent is not just answering questions. It runs a continuous loop until the task is done:

1. **Plan** — break the task into steps
2. **Act** — use tools to do something (read a file, run a command, edit code)
3. **Observe** — read the output, check for errors
4. **Repeat** — loop back to planning with new information

Each loop iteration is called a "step." A complex task might take 30+ steps before the agent decides it's done.

### Tools the agent can use

- Read and write files
- Run terminal commands
- Search the codebase
- Spawn sub-agents for parallel work

---

## 2. The context window

The context window is the agent's **working memory** — everything it can see at once when deciding what to do next. It has a fixed size (e.g. 200,000 tokens) and contains:

| Section | What's in it |
|---|---|
| System prompt | Instructions, persona, available tools |
| Codebase context | Files read, directory listings, search results |
| Conversation + tool history | Every prior message, tool call, and result |
| Latest tool output | Terminal output, file contents, errors |
| Free space | What's left for new output |

### The critical property: it only grows

Every tool call, every file read, every terminal output gets appended. It **never shrinks** during a session. As the context fills up, the agent has less room to think — which is why long tasks can degrade in quality near the end.

---

## 3. Context window vs. model weights

A useful analogy: the agent has two kinds of "memory."

| | What it is | Analogy |
|---|---|---|
| **Model weights** | Everything learned during training — syntax, patterns, concepts | Long-term expertise |
| **Context window** | The current task, files, and history | Working memory |

When a session ends, the context window is **completely wiped**. The weights (expertise) remain. The agent starts every new session with no memory of your project — only its general knowledge.

---

## 4. How to work better with agents

The biggest lever you have is the quality of what goes into context.

### Give rich context upfront
Tell the agent where things are before it starts exploring:
- Which files are relevant
- What the error message is
- What "done" looks like

### Scope tasks appropriately
A 50-step task burns through context fast. Break large work into focused sessions:
- "Refactor the auth module" → one session
- "Refactor the payments module" → next session

### Be precise about tool use
When an agent reads a 2,000-line file to find one function, those 2,000 lines sit in context for the rest of the task. Point the agent at the exact file and function when you can.

### Control test output
Each test run's output stacks up in context. Ask the agent to run one specific failing test rather than the whole suite.

---

## 5. The CLAUDE.md briefing file

Since the agent starts every session blank, you can write a file that gets loaded automatically at the start — like a briefing note for a contractor who forgets everything overnight.

### Recommended structure

```markdown
# Project name

Short description. Stack: Node.js + Express + PostgreSQL.

## Key locations
- API routes      → src/routes/
- Business logic  → src/services/
- DB schema       → prisma/schema.prisma
- Tests           → tests/  (Jest)

## Commands
- Run tests       → npm test
- Run one test    → npm test -- --testPathPattern=auth
- Start server    → npm run dev

## Conventions
- Use async/await, never .then() chains
- Errors go through src/utils/errors.js
- All new routes need an integration test
- Never edit generated files in src/generated/

## Current work
What's in progress right now, what's done, what's blocked.
```

### Why it works
Without this file, the agent spends its first several steps just mapping the project — reading directories, opening files, figuring out how to run tests. The briefing file collapses all of that into a few hundred tokens loaded instantly at the start.

### Tips
- Keep it short and focused — two or three sections beat a sprawling document
- The "commands" section is the highest-value part
- Add "never touch X" notes for generated or off-limits files
- Update the "current work" section at the end of each session so the next one picks up cleanly

---

## 6. Other approaches to optimize context

All of these solve the same core problem: the agent starts blank every session, so make context loading fast, cheap, and accurate.

### RAG (Retrieval-Augmented Generation)
Instead of loading everything upfront, the agent searches a knowledge base and pulls in only the relevant pieces when needed. Great for large codebases where loading everything would fill the context immediately.

### Scratchpad / working notes
The agent writes to a file during the session — discoveries, decisions, things tried that didn't work. This offloads information from context into storage, freeing up room without losing it.

### Summarization between sessions
At the end of a session, write a compact summary of what happened. The next session loads the summary instead of the full history — same information, much less context used.

### Tool-based retrieval
Give the agent tools to fetch what it needs on demand (search, docs lookup, database query). It only pulls information when it actually needs it, keeping context focused.

### Hierarchical agents
An orchestrator agent holds the high-level plan and spawns sub-agents for specific subtasks. Each sub-agent gets a small, focused context — and reports back a summary, not its full context. The orchestrator never has to hold all the detail at once.

---

## Key takeaway

The agents that work best are not the ones with the biggest context windows — they are the ones that keep context **clean, relevant, and well-structured** at every step. Understanding the context window is the foundation for using any AI coding agent effectively.
