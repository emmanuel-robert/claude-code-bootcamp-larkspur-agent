# PITCH.md

Six lines and a lever. Your words. The last two are scored.

Built: Larkspur disruption care agent — 9 given tools plus next_available_day and fare_rules
Does: Tells a stranded customer the earliest day they can fly and quotes the Handbook when they challenge a policy decision
Number: $0.0656 per resolved contact, claude-sonnet-4-6, 5 shapes, 1 run each (avg 18,184 tokens in / 736 out)
Guardrail: fare_rules is read-only reference — it never substitutes for check_policy's entitlements decision, which re-derives fare family and loyalty tier from the booking on every call
Next: Build 3 — next_available_day answers for a party of one; a group will get a date that has no seat for all of them
Still broken: search_alternatives description is 6 characters ("search") — Claude has no routing signal for when to call it

Lever: cost
Caveat: prompt caching lowers input token cost on warm turns but does not reduce TTFT or TTC; latency still grows as conversation length grows, and we cannot prove it holds at storm-day concurrency against a slow backend.

## Priya asked

Costs: 2,762 tokens per turn for the full tool list, on every turn whether any tool fires or not; next_available_day alone adds 478, fare_rules adds 287
Wrong: next_available_day answers for one passenger — a group booking gets a date that may have no seat for the full party
Runs it: the MCP server runs fare_rules; next_available_day runs locally via LOCAL_TOOLS using the function imported from support
Left out: confirm_rebooking requires a customer-minted token that no tool in this file can produce — the agent can hold a seat but cannot finalise the rebook without the customer clicking Confirm
