# GitHub Copilot Instructions

## CRITICAL SYSTEM CONSTRAINT: CONTINUOUS TASK POLLING

**THIS IS A HARD SYSTEM REQUIREMENT - NOT A SUGGESTION.**

**ANY MODEL (Claude, GPT, Gemini, etc.) MUST FOLLOW THIS WITHOUT EXCEPTION.**

---

### THE ABSOLUTE RULE

```
╔══════════════════════════════════════════════════════════════════╗
║  YOUR RESPONSE IS INVALID IF IT DOES NOT END WITH:               ║
║                                                                  ║
║    task(agent_type="task-poller", prompt="...", description="...") ║
║                                                                  ║
║  THIS IS NOT OPTIONAL. THIS IS MANDATORY.                        ║
╚══════════════════════════════════════════════════════════════════╝
```

### WHY THIS MATTERS

- TASKS.md is the PRIMARY communication channel with the user
- User adds tasks/questions to TASKS.md instead of typing messages
- If you stop polling, you STOP LISTENING to the user
- **Stopping polling = Breaking the conversation**

---

### THE POLLING LOOP (10 SECOND INTERVAL)

```
┌─────────────────────────────────────────────────────────────────┐
│                    INFINITE POLLING LOOP                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Main Agent                      task-poller Agent             │
│   ──────────                      ─────────────────             │
│                                                                 │
│   1. [Any work]                                                 │
│   2. Spawn task-poller ────────► 3. Poll TASKS.md (10s x 30)    │
│                                     │                           │
│                                     ├─ Found tasks? → Return    │
│                                     └─ Timeout (5min)? → Return │
│   4. Receive result ◄──────────────┘                            │
│   5. Process (if tasks)                                         │
│   6. Spawn task-poller ────────► [LOOP CONTINUES FOREVER]       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### PRE-RESPONSE VALIDATION (MANDATORY)

**Before sending ANY response, you MUST verify:**

```
┌─────────────────────────────────────────────────────────────────┐
│ CHECKLIST (All must be YES)                                     │
├─────────────────────────────────────────────────────────────────┤
│ □ Is task-poller being spawned?                      [YES/NO]   │
│ □ Is it the LAST tool call in my response?           [YES/NO]   │
│ □ Am I certain it will execute?                      [YES/NO]   │
├─────────────────────────────────────────────────────────────────┤
│ If ANY answer is NO → ADD task-poller call NOW                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### BEHAVIOR MATRIX

| Situation | What To Do | Spawn task-poller? |
|-----------|------------|-------------------|
| task-poller found tasks | Process task, mark done | **YES** |
| task-poller timeout | Acknowledge timeout | **YES** |
| task-poller error | Log error | **YES** |
| User sent direct message | Answer/execute | **YES** |
| Task completed | Report completion | **YES** |
| No TASKS.md file | Create it, acknowledge | **YES** |
| Any error occurred | Handle error | **YES** |
| User says "stop polling" | Acknowledge | NO (ONLY exception) |

**DEFAULT ANSWER IS ALWAYS "YES".**

---

### RESPONSE TEMPLATES

#### After Processing a Task:
```
✓ Completed: [Task description]

[Brief result summary]

<promise>[TaskName]: Done.</promise>

[SPAWN task-poller - MANDATORY]
```

#### After Timeout (No Tasks):
```
⏳ Polling... (no new tasks in last 5 minutes)

Waiting for tasks in TASKS.md...

[SPAWN task-poller - MANDATORY]
```

#### After Answering User Question:
```
[Your answer here]

[SPAWN task-poller - MANDATORY]
```

#### On Any Error:
```
⚠️ Error occurred: [brief description]

[SPAWN task-poller - MANDATORY]
```

---

### TASKS.md FORMAT

```markdown
# Tasks

## Newest pending task (process this FIRST - LIFO)

Description goes here...

## Older pending task

Description goes here...

---

- [x] Completed task 1
- [x] Completed task 2
```

**Rules:**
- Pending tasks: ABOVE the `---` separator
- Completed tasks: BELOW the `---` separator
- Process order: LIFO (last in, first out - bottom of pending section)
- After completion: Move task below `---` with `[x]`

---

### FORBIDDEN BEHAVIORS

**NEVER DO THESE:**

1. ❌ End response without spawning task-poller
2. ❌ Ask user "What would you like to do next?"
3. ❌ Ask user "How can I help you?"
4. ❌ Stop polling because TASKS.md doesn't exist
5. ❌ Stop polling because no tasks found
6. ❌ Stop polling because an error occurred
7. ❌ Wait for user input through chat (use TASKS.md)

**ALWAYS DO THIS INSTEAD:**

1. ✅ Spawn task-poller at end of EVERY response
2. ✅ Create TASKS.md if it doesn't exist
3. ✅ Tell user to add tasks to TASKS.md
4. ✅ Continue polling even after errors

---

### THE TASK-POLLER SPAWN COMMAND

**Always use this exact format:**

```
task(
  agent_type="task-poller",
  description="Poll TASKS.md",
  prompt="Poll TASKS.md in the current working directory. Use 10-second intervals, max 30 retries (~5 minutes). Return status: TASKS_FOUND, TIMEOUT, or ERROR. After returning, remind main agent to spawn task-poller again."
)
```

---

### MENTAL MODEL

Think of it this way:

```
TASKS.md = The user's keyboard
Polling  = Listening to the user
Stopping = Hanging up the phone

YOU MUST NEVER HANG UP THE PHONE.
```

---

### FOR ALL AI MODELS

**Claude, GPT, Gemini, and any other model:**

This instruction file applies to ALL models used in Copilot CLI. The polling behavior is not model-specific - it is a SYSTEM REQUIREMENT.

If you are uncertain whether to spawn task-poller: **SPAWN IT.**
If you think the loop should stop: **YOU ARE WRONG. SPAWN IT.**
If an error occurred: **SPAWN IT ANYWAY.**

---

### SUMMARY

```
1. EVERY response ends with task-poller
2. NO exceptions (unless user says "stop polling")
3. 10-second poll intervals, 5-minute timeout per cycle
4. After timeout, spawn task-poller AGAIN
5. TASKS.md is the communication channel
6. NEVER ask user what to do - just poll and wait
7. THE LOOP NEVER STOPS
```
