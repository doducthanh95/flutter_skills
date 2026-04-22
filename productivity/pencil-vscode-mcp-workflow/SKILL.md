---
name: pencil-vscode-mcp-workflow
description: Use the Pencil VS Code MCP server to inspect and edit .pen design files via HTTP/MCP, including session setup, tool discovery, batch_get, and batch_design.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [pencil, vscode, mcp, design, pen-file, http, batch_design, batch_get]
---

# Pencil VS Code MCP Workflow

Use this skill when working with the Pencil design editor inside VS Code via the Pencil MCP server. This workflow is for reading and editing `.pen` files, inspecting the active canvas state, and applying batched design changes.

## Key discovery
The Pencil server does not expose design actions as plain JSON-RPC methods like `get_editor_state` or `get_guidelines`.
Instead:
1. Call `initialize`
2. Read the `Mcp-Session-Id` response header
3. Use `tools/call` with the tool name in the payload

Direct calls like `{"method":"get_editor_state"}` return `Method not found`.

## Starting the server
A reliable local HTTP mode launch is:
```bash
/Users/<user>/.pencil/mcp/visual_studio_code/out/mcp-server-darwin-arm64 --http --http-port 8099 -app visual_studio_code
```

The server logs typically show:
- HTTP endpoint: `http://localhost:8099/mcp`
- a connected Pencil socket under `~/.pencil/socket/...`

## Session bootstrap
Initialize first:
```python
import requests
base = 'http://127.0.0.1:8099/mcp'
init = {
  'jsonrpc': '2.0',
  'id': 1,
  'method': 'initialize',
  'params': {
    'protocolVersion': '2024-11-05',
    'capabilities': {},
    'clientInfo': {'name': 'client', 'version': '1.0'}
  }
}
r = requests.post(base, json=init)
session_id = r.headers['Mcp-Session-Id']
```

Always include the header in later calls:
```python
headers = {'Mcp-Session-Id': session_id}
```

## Listing tools
Use `tools/list` via the MCP endpoint:
```python
payload = {'jsonrpc':'2.0','id':2,'method':'tools/list','params':{}}
r = requests.post(base, headers=headers, json=payload)
```

Typical tools include:
- `batch_get`
- `batch_design`
- `export_nodes`
- `find_empty_space_on_canvas`
- `get_editor_state`
- `get_guidelines`
- `get_screenshot`
- `get_variables`
- `open_document`
- `replace_all_matching_properties`
- `search_all_unique_properties`
- `set_variables`
- `snapshot_layout`

## Reading editor state
Use `tools/call` with `get_editor_state`:
```python
payload = {
  'jsonrpc': '2.0',
  'id': 3,
  'method': 'tools/call',
  'params': {
    'name': 'get_editor_state',
    'arguments': {'include_schema': True}
  }
}
```

This returns:
- current `.pen` file name
- selected nodes
- top-level node IDs
- the `.pen` schema text

## Reading nodes
Use `batch_get` through `tools/call` to inspect frames/nodes:
```python
payload = {
  'jsonrpc': '2.0',
  'id': 4,
  'method': 'tools/call',
  'params': {
    'name': 'batch_get',
    'arguments': {
      'filePath': 'new',
      'nodeIds': ['FRAME_ID_1', 'FRAME_ID_2'],
      'readDepth': 1,
      'searchDepth': 1
    }
  }
}
```

Useful when you need:
- current frame structure
- direct child nodes
- names, layout, fills, text, sizes

## Editing nodes
Use `batch_design` through `tools/call`.
The `operations` argument is a string containing one operation per line.
Keep each call to 25 operations or fewer.

Example shape:
```python
payload = {
  'jsonrpc': '2.0',
  'id': 5,
  'method': 'tools/call',
  'params': {
    'name': 'batch_design',
    'arguments': {
      'filePath': 'new',
      'operations': '\n'.join([
        'home=I("document",{type:"frame",layout:"vertical",padding:20,gap:14,width:390,height:844})',
        'title=I(home,{type:"text",content:"Photo Guide",fontSize:24,fontWeight:"700"})'
      ])
    }
  }
}
```

## Practical design workflow
1. Read the current editor state.
2. Inspect top-level frames.
3. Decide which screens to redesign.
4. Apply one logical batch at a time:
   - screen structure first
   - then text/content
   - then styling cleanup
5. Verify with `snapshot_layout` or `get_screenshot` after each major batch.

## Verification tools
- `snapshot_layout` for structure and layout problems
- `get_screenshot` for visual confirmation
- `export_nodes` when you need PNG/PDF output
- `get_variables` / `set_variables` for theme and global styling

## Pitfalls
- Do not call design methods as direct JSON-RPC methods; use `tools/call`.
- Always keep the `Mcp-Session-Id` header after initialization.
- `GET /mcp` returns SSE and is not the right place to send design commands.
- `tools/list` may succeed only after `initialize`.
- `batch_design` failures roll back the whole batch.
- Keep operation batches small and logical.
- If the server says a method is missing, verify that you are calling `tools/call` with the tool name, not the tool name as the JSON-RPC method.

## Example end-to-end
```python
import requests

base = 'http://127.0.0.1:8099/mcp'
init = {
  'jsonrpc': '2.0',
  'id': 1,
  'method': 'initialize',
  'params': {
    'protocolVersion': '2024-11-05',
    'capabilities': {},
    'clientInfo': {'name': 'client', 'version': '1.0'}
  }
}
r = requests.post(base, json=init)
session_id = r.headers['Mcp-Session-Id']
headers = {'Mcp-Session-Id': session_id}

# Inspect current file state
state = requests.post(base, headers=headers, json={
  'jsonrpc': '2.0',
  'id': 2,
  'method': 'tools/call',
  'params': {
    'name': 'get_editor_state',
    'arguments': {'include_schema': True}
  }
})

# Edit content
edit = requests.post(base, headers=headers, json={
  'jsonrpc': '2.0',
  'id': 3,
  'method': 'tools/call',
  'params': {
    'name': 'batch_design',
    'arguments': {
      'filePath': 'new',
      'operations': '...'
    }
  }
})
```

---
This skill captures the reusable Pencil-in-VS-Code MCP workflow discovered through trial and error: initialize the server, preserve the session id, call all actions through `tools/call`, and use batched design edits for reliable UI work.
