# Build 1 Code Changes — feature/day1-ekip

## Files Modified

### 1. `agent.py` (new file)

---

### Change 1 — Fault 1: Echo full content list (line 71)

**File:** `agent.py`
**Line:** 71
**Function:** `run_agent()`

```python
# BEFORE (broken):
messages.append({"role": "assistant", "content": text_of(response)})

# AFTER (fixed):
messages.append({"role": "assistant", "content": response.content})
```

**Why:** The Claude API requires the full `response.content` list — which includes both `text` and `tool_use` blocks — to be echoed back as the assistant turn. Without this, the API cannot match tool results to the tool_use block IDs and will error. Passing only `text_of(response)` (a plain string) drops the tool_use blocks entirely.

---

### Change 2 — Fault 2: Remove stale answer capture inside loop (line 73)

**File:** `agent.py`
**Line:** 73 (line deleted)
**Function:** `run_agent()`

```python
# BEFORE (broken — inside the while loop):
answer = text_of(response)

# AFTER (fixed):
# line removed entirely
```

**Why:** This line was capturing the text from a mid-loop response (one that returned `stop_reason == "tool_use"`, so it has no final answer) into `answer`. That stale value would then be returned at the end instead of the actual final reply.

---

### Change 3 — Fault 3: Return from final response (line 80)

**File:** `agent.py`
**Line:** 80
**Function:** `run_agent()`

```python
# BEFORE (broken):
return answer

# AFTER (fixed):
return text_of(response)
```

**Why:** After the loop exits, `response` holds the model's final reply (the one where `stop_reason != "tool_use"`). Reading `text_of(response)` at this point gives the correct answer. The old `answer` variable was either an empty string (initialised as `""` on line 68) or set from a mid-loop intermediate response — neither is correct.

---

### Change 4 — Tool schemas: `build_tools()` (lines 93–230)

**File:** `agent.py`
**Lines:** 93–230
**Function:** `build_tools()`

All 9 tool schemas written from scratch. Each follows the Anthropic tool format with `name`, `description`, and `input_schema`.

| Line | Tool | Key fields |
|------|------|-----------|
| 97–110 | `lookup_booking` | `pnr`, `last_name` (both required) |
| 111–126 | `get_flight_status` | `flight_no`, `date` (YYYY-MM-DD) |
| 127–140 | `search_alternatives` | `pnr` |
| 141–164 | `check_policy` | `pnr`, `cause_code`, `delay_minutes`, `status` |
| 165–173 | `hold_seat` | `option_id`, `pnr` |
| 174–186 | `confirm_rebooking` | `hold_id`, `confirmation_token` |
| 187–204 | `issue_voucher` | `voucher_type`, `amount_usd`, `pnr`, `policy_row_id` |
| 205–220 | `escalate_to_human` | `pnr`, `reason`, `summary_for_human` |
| 221–229 | `send_confirmation` | `pnr`, `message` |

Full schema for each tool:

```python
# lookup_booking — lines 97–110
{
    "name": "lookup_booking",
    "description": (
        "Retrieve a Larkspur reservation from Altura by confirmation code (PNR) "
        "and the passenger's last name. Both are required to prevent a lookup on "
        "a guessed PNR. Returns fare family, loyalty tier, the segment that needs "
        "attention, and any group/partner/minor/SSR flags relevant to scope."
    ),
    "input_schema": {
        "type": "object",
        "properties": {"pnr": {"type": "string"}, "last_name": {"type": "string"}},
        "required": ["pnr", "last_name"],
    },
}

# get_flight_status — lines 111–126
{
    "name": "get_flight_status",
    "description": (
        "Look up a Larkspur or Larkspur Link flight's current OpsFeed status for "
        "one local date: status, delay minutes, and cause. Use this before telling "
        "a customer anything about a flight's timing; never state it from memory."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "flight_no": {"type": "string"},
            "date": {"type": "string", "description": "YYYY-MM-DD"},
        },
        "required": ["flight_no", "date"],
    },
}

# search_alternatives — lines 127–140
{
    "name": "search_alternatives",
    "description": (
        "Search for available Larkspur rebooking alternatives for a disrupted PNR. "
        "Returns option_ids, flight details, cabin, and seat availability from the "
        "alternatives cache. Call this before hold_seat or check_policy so you have "
        "real option_ids to work with — never invent them."
    ),
    "input_schema": {
        "type": "object",
        "properties": {"pnr": {"type": "string"}},
        "required": ["pnr"],
    },
}

# check_policy — lines 141–164
{
    "name": "check_policy",
    "description": (
        "Resolve what Larkspur owes this customer for the disruption: rebooking "
        "waiver, refund path, meal/hotel/ground care, goodwill eligibility and cap, "
        "and any escalation triggers. cause_code, delay_minutes and status describe "
        "what get_flight_status told you; fare_family, loyalty_tier and whether this "
        "is overnight are looked up from the booking, not asked of you. Every "
        "response carries a policy_row_id. Cite it if you reference this decision again."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "pnr": {"type": "string"},
            "cause_code": {"type": "string", "enum": ["WX", "ATC", "MX", "CREW", "SEC"]},
            "delay_minutes": {"type": "integer"},
            "status": {"type": "string", "enum": ["ON_TIME", "DELAYED", "CANCELLED", "DIVERTED"]},
            "wait_minutes_for_alternative": {"type": "integer"},
            "chosen_option_id": {"type": "string"},
        },
        "required": ["pnr", "cause_code", "delay_minutes", "status"],
    },
}

# hold_seat — lines 165–173
{
    "name": "hold_seat",
    "description": "Place a 15-minute hold on one alternative. Reversible. It simply expires.",
    "input_schema": {
        "type": "object",
        "properties": {"option_id": {"type": "string"}, "pnr": {"type": "string"}},
        "required": ["option_id", "pnr"],
    },
}

# confirm_rebooking — lines 174–186
{
    "name": "confirm_rebooking",
    "description": (
        "Finalize a held seat. Irreversible. Requires a confirmation_token that "
        "only the customer's own Confirm-click can produce. You cannot supply it "
        "yourself, and 'the customer said yes' in chat does not substitute for it."
    ),
    "input_schema": {
        "type": "object",
        "properties": {"hold_id": {"type": "string"}, "confirmation_token": {"type": "string"}},
        "required": ["hold_id", "confirmation_token"],
    },
}

# issue_voucher — lines 187–204
{
    "name": "issue_voucher",
    "description": (
        "Issue a meal, ground, hotel, or goodwill voucher. Auto-approves within the "
        "policy's threshold for that type; above it, returns a pending status for a "
        "human. It does not fail. Always pass the policy_row_id that made it eligible."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "voucher_type": {"type": "string", "enum": ["meal", "ground", "hotel", "goodwill"]},
            "amount_usd": {"type": "number"},
            "pnr": {"type": "string"},
            "policy_row_id": {"type": "string"},
        },
        "required": ["voucher_type", "amount_usd", "pnr", "policy_row_id"],
    },
}

# escalate_to_human — lines 205–220
{
    "name": "escalate_to_human",
    "description": (
        "Hand this conversation to a human, with your reasoning attached. Use for "
        "groups, partner segments, unaccompanied minors, refunds, or anything else "
        "out of scope. This is the correct outcome for those cases, not a failure."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "pnr": {"type": "string"}, "reason": {"type": "string"},
            "summary_for_human": {"type": "string"}, "queue": {"type": "string"},
        },
        "required": ["pnr", "reason", "summary_for_human"],
    },
}

# send_confirmation — lines 221–229
{
    "name": "send_confirmation",
    "description": "Send the customer a written confirmation of what was just done. Benign.",
    "input_schema": {
        "type": "object",
        "properties": {"pnr": {"type": "string"}, "message": {"type": "string"}},
        "required": ["pnr", "message"],
    },
}
```

---

### 2. `CLAUDE.md` (new file)

**File:** `CLAUDE.md`
**Lines:** 1–74

Added the Claude Code coaching contract for this repo. Defines:
- What the repo is and how `agent.py` works
- How Claude Code should behave (pair alongside, never solve for the participant)
- Which files are off-limits (`support/`, `verify.py`, `setup.py`, `pod_sync.py`, etc.)
- Rules: never weaken `.gitignore`, never put a key in a committed file, paste traces verbatim
- Environment setup and command reference by build phase
