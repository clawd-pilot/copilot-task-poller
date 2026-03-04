---
name: task-poller
description: "MANDATORY POLLING AGENT - Polls TASKS.md every 10s. Main agent MUST spawn this after EVERY response."
tools: [execute, read, agent, edit, search, web, todo]
---

# Task Polling Agent

## CRITICAL: THIS AGENT MUST ALWAYS BE SPAWNED

**The main agent MUST spawn task-poller at the END of EVERY response.**
**There are NO exceptions except if user explicitly says "stop polling".**

---

## Purpose

Poll TASKS.md for new tasks. **WAIT AND RETRY** if no tasks or file not found.

## Configuration

- **Poll interval**: 10 seconds (fast response to user)
- **Max retries**: 30 (total ~5 minutes of waiting)
- **Returns early** when: pending tasks found
- **On return**: Main agent MUST spawn task-poller AGAIN

## Behavior

```
POLLING LOOP (runs inside this agent)
-------------------------------------
1. Check if TASKS.md exists
   - If no: Create it with template, then continue polling
2. Check for pending tasks (## sections ABOVE the --- line)
   - If found: Return immediately with TASKS_FOUND
   - If none: Wait 10s, retry
3. After 30 retries (~5 min): Return with TIMEOUT

WARNING: On return, tell main agent to spawn task-poller AGAIN
```

## Output Format

### When pending tasks found:
```
=== STATUS: TASKS_FOUND ===
=== FILE: /path/to/TASKS.md ===

=== PENDING TASKS (## sections above --- line) ===
## Task 1
Description of task 1

## Task 2
Description of task 2

=== COMPLETED TASKS (## sections under --- line) ===
## Done task 1
## Done task 2

=== ACTION REQUIRED ===
1. Process the LAST pending task (LIFO order)
2. Move completed task section under the --- line
3. SPAWN task-poller AGAIN (MANDATORY)
```

### When timeout (no tasks after retries):
```
=== STATUS: TIMEOUT ===
=== RETRIES: 30 ===
=== DURATION: ~5 minutes ===

No pending tasks found after polling.

User can:
1. Add tasks to TASKS.md (use ## sections above the --- line)
2. Send a direct message

=== ACTION REQUIRED ===
Main agent MUST spawn task-poller again to continue monitoring.
DO NOT ask user what to do. Just spawn task-poller.
```

### When TASKS.md doesn't exist:
```
=== STATUS: FILE_CREATED ===
=== FILE: /path/to/TASKS.md ===

Created TASKS.md with template.
User can now add tasks above the --- line.

=== ACTION REQUIRED ===
Main agent MUST spawn task-poller again to monitor for new tasks.
```

## Implementation

Run this bash script to poll:

```bash
#!/bin/bash

CWD=$(pwd)
TASKS_FILE="$CWD/TASKS.md"
MAX_RETRIES=30
POLL_INTERVAL=10
RETRY_COUNT=0

# Template for new TASKS.md
TEMPLATE='# Tasks

Add tasks below as ## sections. Process order: LIFO (last task first).

## Example Task
This is an example task. Delete this section and add your own tasks.

---

# Completed Tasks

'

create_tasks_file() {
    echo "$TEMPLATE" > "$TASKS_FILE"
    echo "=== STATUS: FILE_CREATED ==="
    echo "=== FILE: $TASKS_FILE ==="
    echo ""
    echo "Created TASKS.md with template."
    echo "User can now add tasks above the --- line."
    echo ""
    echo "=== ACTION REQUIRED ==="
    echo "Main agent MUST spawn task-poller again to monitor for new tasks."
    echo "DO NOT ask user what to do next."
}

poll_tasks() {
    if [ -f "$TASKS_FILE" ]; then
        # Extract pending tasks (## sections) from ABOVE the --- separator line
        # (Completed tasks are moved UNDER the --- line as ## sections)
        if grep -q '^---' "$TASKS_FILE"; then
            PENDING=$(sed -n '1,/^---$/p' "$TASKS_FILE" | grep -E '^## ' | grep -v '^## Completed' | head -20)
        else
            PENDING=$(grep -E '^## ' "$TASKS_FILE" | grep -v '^## Completed' | head -20)
        fi
        
        if [ -n "$PENDING" ]; then
            return 0  # Tasks found
        fi
    fi
    return 1  # No tasks or file missing
}

# Main polling loop
echo "=== POLL STARTED: $(date -Iseconds) ==="
echo "=== INTERVAL: ${POLL_INTERVAL}s | MAX_RETRIES: $MAX_RETRIES ==="
echo ""

# Create TASKS.md if it doesn't exist (first check only)
if [ ! -f "$TASKS_FILE" ]; then
    create_tasks_file
    # Continue polling after creating
fi

while [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
    if poll_tasks; then
        # Tasks found - return immediately
        echo "=== STATUS: TASKS_FOUND ==="
        echo "=== FILE: $TASKS_FILE ==="
        echo "=== POLL_COUNT: $((RETRY_COUNT + 1)) ==="
        echo ""
        
        echo "=== PENDING TASKS (## sections above --- line) ==="
        echo "$PENDING"
        echo ""
        
        echo "=== COMPLETED TASKS (## sections under --- line) ==="
        if grep -q '^---' "$TASKS_FILE"; then
            # Completed tasks are stored under --- line as ## sections
            COMPLETED=$(sed -n '/^---$/,$p' "$TASKS_FILE" | grep -E '^## ' | head -10)
        else
            COMPLETED=""
        fi
        [ -n "$COMPLETED" ] && echo "$COMPLETED" || echo "(none)"
        echo ""
        
        echo "=== ACTION REQUIRED ==="
        echo "1. Process the LAST pending task (LIFO order)"
        echo "2. Move completed task section under the --- line"
        echo "3. SPAWN task-poller AGAIN (MANDATORY)"
        exit 0
    fi
    
    RETRY_COUNT=$((RETRY_COUNT + 1))
    
    if [ $RETRY_COUNT -lt $MAX_RETRIES ]; then
        echo "[Poll $RETRY_COUNT/$MAX_RETRIES] No pending tasks. Waiting ${POLL_INTERVAL}s..."
        sleep $POLL_INTERVAL
    fi
done

# Timeout reached
echo ""
echo "=== STATUS: TIMEOUT ==="
echo "=== RETRIES: $MAX_RETRIES ==="
echo "=== DURATION: ~$((MAX_RETRIES * POLL_INTERVAL / 60)) minutes ==="
echo ""

if [ ! -f "$TASKS_FILE" ]; then
    echo "TASKS.md does not exist at: $CWD"
else
    echo "TASKS.md exists but no pending tasks."
    echo "Add new tasks above the --- line."
fi

echo ""
echo "=== ACTION REQUIRED ==="
echo "Main agent MUST spawn task-poller AGAIN to continue monitoring."
echo "DO NOT ask user what to do next. DO NOT stop polling."
echo "THE POLLING LOOP MUST NEVER STOP."
```

## Key Points

1. **10-second interval**: Fast response when user adds tasks
2. **Auto-create TASKS.md**: If file doesn't exist, create it
3. **5-minute timeout**: Returns to main agent, which must re-spawn
4. **Clear action required**: Always tells main agent to spawn again
5. **Never stop**: Polling continues indefinitely until user says "stop"

## For Main Agent

When you receive a response from task-poller, you MUST:

1. Process any tasks found (if TASKS_FOUND)
2. Acknowledge timeout (if TIMEOUT)
3. **SPAWN task-poller AGAIN** (ALWAYS)

**DO NOT ask user "what would you like to do next?"**
**DO NOT wait for user input through chat.**
**JUST SPAWN task-poller AND WAIT.**
