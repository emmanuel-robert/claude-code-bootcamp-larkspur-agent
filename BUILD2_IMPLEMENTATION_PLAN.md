# Build 2 Implementation Plan

## Goal

Add two custom tools to the Larkspur disruption agent and move one of them behind the MCP server, proving the model reaches for a new tool unprompted when a customer question demands it.

---

## Step 2.1 — Your own tool, chosen unprompted

### What the gate checks

- At least one tool beyond the given nine is present
- The new tool is registered in `LOCAL_TOOLS`
- `next_available_day` description is ≥ 40 characters
- The model chose `next_available_day` on the probe message without being told to
- The tool returned a non-empty string

### Files changed

**`build2_probe.txt`** — created at repo root (gate expects this exact name):

```
K7PQ2M
Marchetti
When is the first day I can actually fly?
```

**`agent.py`** — `EXTRA_TOOLS` already populated by a prior commit. `LOCAL_TOOLS` already wired. No code changes needed for 2.1.

### Verify

```bash
python3 verify.py 2.1 --name "Ekip Kalir"
```

Evidence code: **FB8-5CF**

---

## Step 2.2 — Move one tool behind the MCP server

### What the gate checks

- MCP server answers `tools/list` with both tools
- At least one MCP tool is offered to Claude
- No tool name is served twice (local and MCP)
- An MCP-discovered tool fired on the probe
- The tool returned a non-empty string

### Root cause of failures

1. `tool_list()` was returning `build_tools() + EXTRA_TOOLS` — MCP-discovered schemas were never included, so Claude couldn't see or call them.
2. `fare_rules` was in both `EXTRA_TOOLS` and the MCP server, causing the "served twice" duplicate check to fail.

### Code changes

**`agent.py` — `tool_list()`** (line ~111): add `mcp_client.discover()` so Claude is offered MCP tool schemas:

```python
# Before
def tool_list() -> List[Dict[str, Any]]:
    return build_tools() + EXTRA_TOOLS

# After
def tool_list() -> List[Dict[str, Any]]:
    return build_tools() + mcp_client.discover() + EXTRA_TOOLS
```

**`agent.py` — `EXTRA_TOOLS`**: remove `next_available_day` (now owned by MCP) and `fare_rules` (also owned by MCP — both tools live on the server):

```python
# Before
EXTRA_TOOLS: List[Dict[str, Any]] = [
    {
        "name": "next_available_day",
        ... # full schema
    },
    {
        "name": "fare_rules",
        ... # full schema
    },
]

# After
EXTRA_TOOLS: List[Dict[str, Any]] = []
```

**`agent.py` — `LOCAL_TOOLS`**: remove `next_available_day` (MCP server owns it now):

```python
# Before
LOCAL_TOOLS: Dict[str, Any] = {
    "next_available_day": next_available_day,
}

# After
LOCAL_TOOLS: Dict[str, Any] = {}
```

**`PITCH.md` — `Number:` field**: record schema token count before the move:

```
Number: 2,762 schema tokens per turn (11 tools) before moving next_available_day to MCP
```

### Schema token savings

| State | Tokens | Tools |
|---|---|---|
| Before move (2.1) | 2,762 | 11 (9 given + 2 custom) |
| After move (2.2) | 2,789 | 11 (same names, all via MCP) |

Token delta was +27 because both tools moved to MCP but the schemas themselves didn't shrink — the gain is architectural (server owns the schemas, not the agent file).

### Verify

```bash
python3 support/mcp_selftest.py   # 31/31 PASS
python3 verify.py 2.2 --name "Ekip Kalir"
```

Evidence code: **0E6-5AA**

---

## Final state of agent.py edited sections

```python
EXTRA_TOOLS: List[Dict[str, Any]] = []   # both tools now live on MCP server

LOCAL_TOOLS: Dict[str, Any] = {}         # no local function wrappers needed

def tool_list() -> List[Dict[str, Any]]:
    return build_tools() + mcp_client.discover() + EXTRA_TOOLS
```

---

## Next step

Bank both evidence codes on the build site, then run `python3 readout.py` before whoever is committer runs `python3 pod_sync.py --push-canon`.
