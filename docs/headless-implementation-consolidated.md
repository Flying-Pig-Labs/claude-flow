# Headless Swarm Implementation - Consolidated Documentation

## Current System State

### The Decision: Using `-p` Flag Approach

After extensive analysis and testing, we've decided to implement headless mode using Claude CLI's `-p` (print) flag. This approach:

1. **Maintains 100% backward compatibility**
2. **Preserves all swarm infrastructure** (tracking directories, database integration, etc.)
3. **Enables non-interactive execution** for CI/CD, batch processing, and automation
4. **Works with existing MCP tools** without modification

### Implementation Status

**Current State**: The workaround is documented and working. The actual implementation is pending.

**What Works Today**:
```bash
# Direct Claude CLI invocation (bypasses claude-flow)
claude -p --mcp-config claude-flow-mcp.json < swarm-prompt.txt
```

**What Will Work After Implementation**:
```bash
# Native claude-flow support
npx claude-flow@alpha swarm "task" --executor
npx claude-flow@alpha swarm "task" --output-format json
npx claude-flow@alpha swarm "task" --background
```

## Technical Implementation Plan

### Core Changes Required

1. **Detect Headless Mode** in `src/cli/simple-commands/swarm.js`:
   ```javascript
   const isHeadless = flags.executor || 
                      flags['output-format'] === 'json' || 
                      flags.background ||
                      process.env.CI === 'true';
   ```

2. **Add `-p` Flag When Headless**:
   ```javascript
   if (isHeadless) {
     claudeArgs.push('-p');
     claudeArgs.push('--output-format', 'stream-json');
     claudeArgs.push('--max-turns', '50');
   }
   ```

3. **Ensure MCP Configuration**:
   ```javascript
   if (isHeadless && !flags['mcp-config']) {
     const mcpConfig = await ensureMCPConfig();
     claudeArgs.push('--mcp-config', mcpConfig);
   }
   ```

4. **Handle Headless Output**:
   - Capture stream-json output
   - Parse tool calls and results
   - Save to `./swarm-runs/{swarmId}/output.json`
   - Update process tracking files

### Files to Modify

1. **`src/cli/simple-commands/swarm.js`**:
   - Add `ensureMCPConfig()` function
   - Add `handleHeadlessOutput()` function
   - Modify claude spawning logic (lines 568-599)
   - Remove references to undefined `basicSwarmNew`

### Expected Behavior After Implementation

1. **Headless Execution**:
   - Automatically detects non-interactive scenarios
   - Adds `-p` flag to claude invocation
   - Captures structured output
   - Maintains all tracking infrastructure

2. **Backward Compatibility**:
   - Interactive mode unchanged
   - All existing flags work as before
   - Swarm tracking directories preserved
   - Database integration maintained

3. **New Capabilities**:
   - CI/CD pipeline integration
   - Batch processing support
   - JSON output for automation
   - Background execution tracking

## Usage Examples

### Current Workaround (Available Now)

```bash
#!/bin/bash
# Create MCP config
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

# Execute headless swarm
claude -p \
  --mcp-config claude-flow-mcp.json \
  --dangerously-skip-permissions \
  --output-format stream-json \
  --max-turns 50 \
  < swarm-task.txt > output.json
```

### Future Native Support (After Implementation)

```bash
# JSON output mode (forces headless)
npx claude-flow@alpha swarm "Build REST API" --output-format json

# Background execution
npx claude-flow@alpha swarm "Long running task" --background

# Explicit executor mode
npx claude-flow@alpha swarm "Complex task" --executor

# CI/CD integration
CI=true npx claude-flow@alpha swarm "Automated task"
```

## Key Decisions

1. **Why `-p` flag?**: It's the official Claude CLI flag for non-interactive print mode
2. **Why not fix basicSwarmNew?**: The `-p` approach is simpler and maintains all existing infrastructure
3. **Why preserve tracking dirs?**: Enables `swarm status`, `swarm list`, and monitoring tools to work
4. **Why stream-json output?**: Provides structured data for parsing and automation

## Next Steps

1. Implement the changes in `src/cli/simple-commands/swarm.js`
2. Add integration tests for headless scenarios
3. Update documentation with native examples
4. Remove temporary workaround documentation after release

## Summary

The headless swarm implementation using the `-p` flag is a minimal, surgical fix that:
- Solves the immediate problem (ReferenceError: basicSwarmNew)
- Maintains 100% backward compatibility
- Preserves all swarm infrastructure
- Enables critical automation use cases
- Requires minimal code changes

This consolidated document represents the complete system state and implementation plan for headless swarm execution in Claude Flow.