# Remote and Headless Execution Guide

## Overview

This document explains how to run Claude Flow swarms in headless or remote execution environments. While the `claude-flow swarm` command has a critical bug, we have a **working solution** using Claude CLI's print mode.

## Current Status: Working Solution Available

**✅ SOLUTION**: Use Claude CLI directly with MCP configuration to bypass the broken swarm command and achieve full headless execution.

## The Core Issue

The swarm command fails immediately when attempting to use executor mode (required for headless operation) due to an undefined function reference:

```
ReferenceError: basicSwarmNew is not defined
    at Object.swarmCommand [as handler] (file:///src/cli/simple-commands/swarm.js:807:7)
```

This error occurs in ALL non-interactive execution scenarios.

## Failed Execution Methods

### 1. Executor Mode with JSON Output
```bash
# DOES NOT WORK - fails with basicSwarmNew error
npx claude-flow swarm "Build API" --executor --output-format json --output-file results.json
```
**Result**: Immediate failure, no files created

### 2. Background Mode
```bash
# DOES NOT WORK - hangs or fails
npx claude-flow swarm "Build API" --background
```
**Result**: Process hangs due to argument parsing bugs

### 3. Direct nohup Execution
```bash
# DOES NOT WORK - fails with basicSwarmNew error
nohup npx claude-flow swarm "Build API" --executor --output-format json > log.txt 2>&1 &
```
**Result**: Process exits immediately after error

### 4. SPARC Non-Interactive Mode
```bash
# Limited functionality - only works for SPARC workflows
npx claude-flow sparc run code "task" --non-interactive
```
**Result**: Works only for SPARC-specific workflows, not general swarm tasks

## What Happens When You Try

1. **No Output Files**: No JSON, MD, or any other files are created
2. **No Task Execution**: The swarm never initializes or executes any tasks
3. **Immediate Exit**: Process fails and exits within seconds
4. **Misleading Exit Codes**: May return exit code 0 despite failure

## Architecture Limitations

### Design Issues
1. **Swarm as Launcher**: The swarm command is designed to spawn interactive Claude Code CLI sessions
2. **No Native Headless Mode**: System lacks built-in support for batch/headless execution
3. **Executor Mode Incomplete**: The fallback executor has missing function references

### Technical Blockers
- `basicSwarmNew` function is referenced but never defined
- JSON output mode forces broken executor path
- Background script has fundamental argument parsing bugs
- No bidirectional communication with spawned Claude Code process

## Impact on Use Cases

### ❌ Cannot Use For:
- CI/CD pipelines
- Automated batch processing
- Remote server execution
- Scheduled/cron jobs
- Containerized environments
- Serverless functions
- Background task queues

### ✅ Only Works In:
- Interactive terminal sessions with Claude Code installed
- Manual execution with user present
- SPARC-specific workflows with `--non-interactive` flag

## Working Solution: Claude CLI Print Mode

### Quick Start

1. **Create MCP Configuration**:
```bash
cat > claude-flow-mcp.json << 'EOF'
{
  "mcpServers": {
    "claude-flow": {
      "command": "npx",
      "args": ["claude-flow@alpha", "mcp", "start"],
      "type": "stdio"
    }
  }
}
EOF
```

2. **Execute Headless Swarm**:
```bash
claude -p \
  --mcp-config claude-flow-mcp.json \
  --dangerously-skip-permissions \
  --output-format stream-json \
  --max-turns 50 \
  < swarm-prompt.txt > output.json
```

This approach:
- ✅ Bypasses the broken `basicSwarmNew` error
- ✅ Provides full MCP tool access
- ✅ Works in CI/CD pipelines
- ✅ Supports remote execution
- ✅ Enables batch processing

## Required Fixes

To enable remote/headless execution, the following fixes are needed:

1. **Define `basicSwarmNew` function** or correct the import
2. **Implement proper executor mode** without undefined references
3. **Fix background mode** argument parsing
4. **Add dedicated batch mode** for non-interactive execution
5. **Improve error handling** and exit codes

## Complete Working Example

```bash
#!/bin/bash
# headless-swarm-execution.sh

# Create swarm prompt
cat > swarm-task.txt << 'EOF'
You are orchestrating a Claude Flow swarm. Use these MCP tools:
- mcp__claude-flow__agent_spawn - Create agents
- mcp__claude-flow__task_create - Create tasks
- mcp__claude-flow__memory_store - Store data

Please:
1. Spawn a coordinator agent
2. Create a task to build a REST API
3. Store the project plan in memory
4. Execute the development workflow
EOF

# Run headless swarm
SESSION_ID=$(uuidgen)
claude -p \
  --mcp-config claude-flow-mcp.json \
  --dangerously-skip-permissions \
  --output-format stream-json \
  --max-turns 50 \
  --session-id $SESSION_ID \
  < swarm-task.txt \
  > "results-$SESSION_ID.json" \
  2> "errors-$SESSION_ID.log"

echo "Execution complete. Check results-$SESSION_ID.json"
```

## Use Cases Now Enabled

### ✅ Can Now Use For:
- CI/CD pipelines
- Automated batch processing  
- Remote server execution
- Scheduled/cron jobs
- Containerized environments
- Serverless functions
- Background task queues

### Key Benefits:
- No interactive terminal required
- Full swarm orchestration capabilities
- Structured JSON output for parsing
- Complete automation support

## Important Notes

### What Changed:
- **Problem**: The `claude-flow swarm` command has a `basicSwarmNew` undefined error
- **Solution**: Use `claude -p` with MCP configuration instead
- **Result**: Full headless execution capability restored

### Requirements:
- Claude CLI installed (`claude --version`)
- Claude Flow installed (`npm list -g claude-flow`)
- Valid MCP configuration file
- Swarm prompt file with instructions

## Summary

While the Claude Flow swarm command has a critical bug in v2.0.0-alpha.78, headless execution is **fully achievable** using Claude CLI's print mode with MCP configuration. This solution enables all automation use cases including CI/CD, remote execution, and batch processing.

For detailed implementation guidance, see the [Headless Execution Guide](./headless-execution-guide.md).

---

*Last updated: August 2025*  
*Based on claude-flow version: 2.0.0-alpha.78*  
*Solution verified and working*