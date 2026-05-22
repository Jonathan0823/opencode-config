---
description: Discover and load relevant project context files
agent: build
subtask: true
---

Load relevant context files for the current task.

## User Request

$ARGUMENTS

## Process

1. **Detect task type**: Based on the request, determine what context is relevant:
   - Code/writing -> code-quality standards
   - Tests -> test-coverage standards
   - Documentation -> documentation standards
   - Review -> code-review workflow
   - Architecture -> patterns and conventions

2. **Discover**: Search context directories for matching files.

3. **Load**: Read and present the most relevant context.

4. **Report**: List what was loaded and why.
