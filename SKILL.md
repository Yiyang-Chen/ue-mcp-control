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

## Session Initialization (Required Before Any Tool Call)

This MCP server uses the StreamableHTTP protocol and requires a `Mcp-Session-Id` header on every request. You **must** initialize a session first to obtain this ID.

On Windows PowerShell, `curl -H` does not work. Always use `[System.Net.WebRequest]` directly.

**Step 1 — Initialize session and capture the Session ID:**

```powershell
$req = [System.Net.WebRequest]::Create("http://localhost:3000/mcp")
$req.Method = "POST"
$req.ContentType = "application/json"
$req.Accept = "application/json, text/event-stream"
$body = '{"jsonrpc":"2.0","id":0,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"cursor","version":"1.0"}}}'
$bytes = [System.Text.Encoding]::UTF8.GetBytes($body)
$req.ContentLength = $bytes.Length
$stream = $req.GetRequestStream()
$stream.Write($bytes, 0, $bytes.Length)
$stream.Close()
$resp = $req.GetResponse()
$sessionId = $resp.Headers["Mcp-Session-Id"]
(New-Object System.IO.StreamReader($resp.GetResponseStream())).ReadToEnd() | Out-Null
Write-Host "Session ID: $sessionId"
```

**Step 2 — Define a reusable helper function for all subsequent calls:**

```powershell
function Invoke-UEMcp($id, $rawJsonBody) {
    $req = [System.Net.WebRequest]::Create("http://localhost:3000/mcp")
    $req.Method = "POST"
    $req.ContentType = "application/json"
    $req.Accept = "application/json, text/event-stream"
    $req.Headers.Add("Mcp-Session-Id", $sessionId)
    $bytes = [System.Text.Encoding]::UTF8.GetBytes($rawJsonBody)
    $req.ContentLength = $bytes.Length
    $stream = $req.GetRequestStream()
    $stream.Write($bytes, 0, $bytes.Length)
    $stream.Close()
    $resp = $req.GetResponse()
    return (New-Object System.IO.StreamReader($resp.GetResponseStream())).ReadToEnd()
}
```

Always pass the JSON body as a **raw string literal** — do NOT use `ConvertTo-Json` on the arguments object, as PowerShell may alter the structure.

## Available Tools

After initializing the session, call `manage_tools` with `action: list_tools` to get the current tool list:

```powershell
Invoke-UEMcp 1 '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"manage_tools","arguments":{"action":"list_tools"}}}'
```

This returns all enabled tools with their names and categories. Use it at the start of any session to confirm what's available before calling other tools.

## Tool Schema Discovery (Required Before First Call to Any Tool)

`list_tools` only returns tool names and categories — it does **not** include actions, parameters, or usage details. Before calling a tool for the first time, you **must** fetch its full input schema via the MCP standard `tools/list` method:

```powershell
Invoke-UEMcp 2 '{"jsonrpc":"2.0","id":2,"method":"tools/list"}'
```

This returns the complete JSON Schema for every tool, including all available `action` values, parameter names, types, and which parameters are required. **Read and understand the schema for the target tool before making any call.** Do not guess action names or parameter names.

If the schema for a specific tool is too large, you can also read its source definition directly at `Plugins/McpAutomationBridge/Source/McpAutomationBridge/Private/MCP/Tools/McpTool_<ToolName>.cpp`.

Before calling any tool, also check if a file named after that tool exists in `.cursor/skills/ue-mcp-control/`. If it does, read and follow its instructions first.

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