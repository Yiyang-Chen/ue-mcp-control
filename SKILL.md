---
name: ue-mcp-control
description: Control Unreal Engine editor via MCP tools for this UE5 project. Use when the user asks to operate the UE editor (open levels, spawn actors, edit blueprints, run PIE, manage assets, etc.), or when any Unreal Engine operation is requested.
disable-model-invocation: true
---

# When and Why Using MCP

UE projects are **not plain text codebases**. Assets like levels, Blueprints, and materials are stored in binary `.uasset` files — directly editing them corrupts the file. The editor also maintains in-memory state (loaded actors, dirty packages, active PIE sessions) that file edits cannot reach.

The MCP bridge solves this by routing all operations through UE's own C++ subsystems at runtime, the same APIs the editor uses internally. This means:

- **Safe**: changes go through UE's transaction system, supporting undo
- **Live**: affects the running editor immediately, no restart needed
- **Correct**: Blueprint graphs, actor transforms, asset references stay valid

**Default to MCP for any UE operation.** Only fall back to direct file editing for plain-text files (`.ini`, `.uproject`, C++ source).

# How to Install

See https://github.com/ChiR24/Unreal_mcp and follow the instructions.

# UE MCP Control

## Project Info

- **UE version**: UE5
- **MCP plugin**: McpAutomationBridge (installed at `Plugins/McpAutomationBridge`)

## MCP Connection

The `unreal-engine` MCP server is configured in `.cursor/mcp.json` and connects to the plugin at `http://localhost:3000/mcp`.

**Prerequisite**: UE editor must be open and the MCP Automation Bridge plugin enabled. If tools fail, ask the user to check the editor status bar for `● MCP :3000`.

## Available Tools

Call `manage_tools` with `action: list_tools` to get the current tool list at runtime:

```json
{
  "name": "manage_tools",
  "arguments": { "action": "list_tools" }
}
```

This returns all enabled tools with their names, descriptions, and supported actions. Use it at the start of any session to confirm what's available before calling other tools.

Before calling any tool, check if a file named after that tool exists in `.cursor/skills/ue-mcp-control/`. If it does, read and follow its instructions first.

## C++ File Editing

C++ source files (`.h` / `.cpp`) are plain text and edited directly by the agent. After editing, **always trigger a build via MCP** instead of asking the user to compile manually:

```json
{
  "name": "system_control",
  "arguments": { "action": "run_ubt" }
}
```

**Before using any UE C++ method, macro, or UFUNCTION/UPROPERTY specifier**, search the engine source headers to confirm the exact signature — do not rely on memory or training data. UE has gone through many iterations and the agent may hallucinate outdated or nonexistent APIs.

Use `Grep` to search for the class or method name in the UE engine source directory. The header file is the ground truth — it shows the exact signature, supported specifiers, and inline comments.

## Config File Editing

**Never directly edit config files** (`.ini`, `.uproject`, etc.). Instead:

1. Write a Python script that performs the modification
2. Execute it via `system_control` with `action: execute_python`, passing the script as inline code or a temp file path
3. Delete the temp file after execution

This ensures config changes are auditable, repeatable, and don't risk partial writes from direct file edits.