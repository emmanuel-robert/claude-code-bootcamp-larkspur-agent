# Build 4 · Inference Optimization — Make It Scale

**Branch:** `feature/day2`  
**Status:** Build 3 (gate 3.1) complete. Build 4 is the final build.  
**Lane declared:** `cost` (PITCH.md: `Lever: cost`)  
**Time budget:** 45 min total — 5 choose + 20 build + 12 bench + 8 deliver

---

## Part 1 · Build 4 Overview (read before touching code)

### The training: Inference Optimization

Inference is everything between your `send` and their answer. Two phases, paid separately:

- **Prefill → TTFT** — reads your whole prompt in one pass. Wait grows with everything you send.
- **Decode → TTC** — writes one token at a time, conditioned on the last. Longer answers take longer.

Every client conversation about AI cost comes back as one of three questions:

| Question | What they are really asking |
|---------|----------------------------|
| **What is the intelligence?** | Which model runs the turn, and whether it holds the shapes that matter |
| **Can it be cheaper?** | Dollars per resolved contact — not dollars per token |
| **Can it be faster?** | Seconds at p50, end to end, the way a stranded passenger feels it |

### The inference triangle

Pull one corner and the other two move.

```
         Capability
            /\
           /  \
          /    \
    Cost ←——————→ Speed
```

### Four metrics to measure

| Metric | Unit | What it is |
|--------|------|-----------|
| **TTFT** | ms | Time to first token — what users feel |
| **TTC** | ms | Time to completion — what your SLA measures |
| **OTPS** | tokens/sec | Output throughput after first token — what scales with traffic |
| **$/1M** | dollars | Input + output, priced separately — what procurement signs |

### Model tiers (Sept 2026 list prices)

| Model | Speed tier | $/M output | Best for |
|-------|-----------|-----------|---------|
| Haiku 4.5 | Speed first | $5 | High-volume, streaming, simple routing |
| Sonnet 5 | Balanced | ~$25 | Most production workloads, agentic tool use, customer-facing |
| Opus 5 | Max intelligence | ~$25 | Expert coding, high-stakes decisions, research |
| Fable 5 | Frontier | $50 | Long-horizon autonomous agents, frontier tasks |

Same prompt, same workload: ~$900K/month apart between cheapest and most expensive.

### The tool-round-trip tax

Every tool call is another round-trip. Having a tool **available** (even if never called)
resends its schema, the full request, and the result on the next turn.

| | TTFT | TTC | $/1K calls |
|-|------|-----|-----------|
| No tools | 1031 ms | 2906 ms | $8.27 |
| With tools | 1823 ms | 3338 ms | $28.43 |

**+77% TTFT · +15% TTC · +244% cost** — nothing changed but the tool being available.

### Prompt caching

Pay to write the prefix once. Read it back at ~10× cheaper on every call after.

- Cold call: 1,161 tokens written to cache (`cache_creation_tokens` is large, `cache_read_tokens` is 0)
- Warm call: same 1,161 tokens read back cheaply (inverse)
- **The wrinkle:** cheap is not fast. Latency still climbs as conversation grows.
- Always bench with `--cold` — it makes the sweep pay its own cache write, which is the
  deployed steady state. Without it, you measure a lucky cache read that production never gets.

### Larkspur's real progression (what's possible without changing the model)

| Configuration | Week | p95 | Model $/contact |
|--------------|------|-----|----------------|
| One mid-tier model, no cache | 2 | 9.8s | $0.173 |
| Plus caching, parallel reads | 5 | 6.9s | $0.102 |
| Plus small-model triage, trimmed output | 7 | 4.7s | $0.087 |

The point: none of these wins required changing the model.

---

### What each lane buys and charges

"All three" is not an answer. Pick one and know what it costs you.

| Lane | Buys | Charges |
|------|------|---------|
| **Intelligence** | Harder shapes, fewer wrong answers | Tokens, latency, more to evaluate |
| **Cost** | A number the sponsor can defend | You must keep a cache valid; reference lever reads ~5× slower at 1 run per shape and no change at 3 — so **3 runs is required** |
| **Speed** | Lower abandonment, and the SLA | A smaller model is a bet on quality |

### Seven levers bench can see

Every lever here lands in a place you have already edited in agent.py. The gate holds the
model still in cost and speed lanes — the movement is yours, not the swap.

| Lane | Lever | Where it lands in agent.py | What should move |
|------|-------|---------------------------|-----------------|
| **Cost** | Cache the front of the request: system prompt + tool list marked for caching so every turn after the first reads instead of pays | `run_agent()` — the two `messages.create` calls | Tokens in, and the cache line under the bench table |
| **Cost** | Drop tools the trace never called. `run.py --tool-tax` counts the schema tokens for each one | `build_tools()` and `tool_list()` | Tokens in on every turn |
| **Cost** | A lower `max_tokens` and a one-line answer rule | `run_agent()` and `TONE_ADDENDUM` | Tokens out |
| Speed | Fewer turns: a description that lets Claude ask for two things at once | `build_tools()` | Turns, and p50 |
| Speed | Run the tool calls in one turn side by side instead of one after another | `tool_results()` | p50, when a turn carries two or more tool calls |
| Intelligence | Write `TONE_ADDENDUM`: what to do with abuse, threats, and a customer who asks for something out of scope | First slot at top of `agent.py` (line 19) | Wire rules passed on Stage 2 |
| Intelligence | Step the model up one tier for Stage 2, and say what it costs | `model=` in `run_agent()` | Wire rules on Stage 2, and the cost line beside it |

### What the client sees

The number is not the proof; the screen that shows it is. Two surfaces are already built
and both read your own files — you fill them in the last 20 minutes.

**The panel beside the chat** (demo's right side) has five sections:
1. The number — its unit, and what it is measured against
2. Before and after — straight off your two bench files
3. Evals — from your cases
4. Guardrail — the forged token refused
5. The claim — from `PITCH.md`

**The client page:**
```bash
python3 readout.py --client
open readout-client.html
```
Same evidence, in the words a sponsor reads.

Screenshot both. Nothing else goes into the room with you.

### Provable now vs not yet

| Provable now | Not yet |
|-------------|---------|
| Your declared lane's number moved, before and after, on your own laptop, 3 runs per shape | Cannot prove the loaded number — `bench.py` reports model cost only; Larkspur's loaded cost was ~40% higher than model cost |
| Stage 1 still resolves 5/5 after the change, so the win did not break a shape | Cannot prove accuracy at volume, or anything on real traffic |
| You can prove what the lever charged you (if you benched 3 runs/shape on both sides) | Cannot prove the same lever holds on a storm day, at concurrency, against a slow backend |

---

## Part 2 · Step 4.1 — Pull the Lever, Measure It

> Bench before you tune. The before number cannot be rebuilt after.  
> A win that breaks a Stage 1 shape fails — the gate checks both sides.

### Step 1 — Confirm lane declaration ✅

Open `PITCH.md`. The `Lever:` line must read exactly:

```
Lever: cost
```

The gate reads this word and refuses the menu. No extra text after it.

### Step 2 — Write the caveat in PITCH.md

Add one sentence right after the `Lever:` line, **before you run anything**. One sentence:
what this lever costs, and what the number about to arrive will **not** cover.

```
Lever: cost
Caveat: <one sentence — what caching / tool-drop / max_tokens costs you, and what it cannot prove>
```

Example for prompt caching:
> Caveat: prompt caching lowers input token cost by ~10× on warm turns but does not reduce
> TTFT or TTC; latency still grows as conversation length grows, and we cannot prove it holds
> at storm-day concurrency against a slow backend.

Example for dropping unused tools:
> Caveat: removing unused tool schemas cuts every-turn token overhead but does not affect
> output length or latency, and the saving only shows if those tools were never called in
> production.

### Step 3 — Bench before (critical — do this before any code change)

```bash
python3 bench.py --label before --stage 1 --runs 3 --cold
```

`--cold` puts a nonce in the system prefix so the first conversation pays its own cache
write and the rest read it — this is the deployed steady state. Use `--runs 3` on both
sides; at 1 run, output-token deltas under ~15% are sampling noise.

The before number cannot be rebuilt after you pull the lever.

### Step 4 — Choose exactly one cost lever and pull it

**Pick one. Not two.** The gate reads one declared lane and one before/after diff. Pulling
two levers means you cannot explain which one moved the number.

---

#### Lever A · Prompt Caching

Mark the system prompt and tool list for caching so every turn after the first reads
instead of pays. In `run_agent()` (line 57+), wrap each `messages.create` call to add
`cache_control` to the system param.

In `agent.py`, change both `client.messages.create` calls inside `run_agent()`:

```python
# Change the system= param in both API calls from:
system=runtime_preamble() + SYSTEM_PROMPT + TONE_ADDENDUM,

# To structured cache-control blocks:
system=[
    {
        "type": "text",
        "text": runtime_preamble() + SYSTEM_PROMPT + TONE_ADDENDUM,
        "cache_control": {"type": "ephemeral"},
    }
],
```

**What should move:** `cache_creation_tokens` large on turn 1, `cache_read_tokens` large
on turns 2+. Cost column in bench drops. TTFT and TTC are unchanged or slightly worse.

**Confirm it's working:** after benching, check the cache line printed under the bench
table. If `cache_creation_tokens` is 0 on the first run, the header is not set correctly.

---

#### Lever B · Drop Unused Tools

Check what the trace actually calls:

```bash
python3 run.py K7PQ2M --trace    # cancellation — most tools fire here
python3 run.py M3XR8T --trace    # delay under threshold
python3 run.py G2HL9V --trace    # group — escalates early, few tools
python3 run.py --tool-tax        # prints schema token cost per tool
```

`search_alternatives` has a 6-character description (`"search"`) — Claude has no routing
signal for when to call it and likely never does. Each unused tool still sends its full
schema on every turn as tokens in.

In `build_tools()` (line 94 in `agent.py`), remove the schema block for every tool
the trace confirms never fires. Typical candidate:

```python
# Remove this block from build_tools() if trace confirms it never fires:
{
    "name": "search_alternatives",
    "description": "search",
    "input_schema": {
        "type": "object",
        "properties": {"pnr": {"type": "string"}},
        "required": ["pnr"],
    },
},
```

**What should move:** tokens in on every turn. The saving is (schema tokens removed) ×
(number of turns per conversation) × (number of conversations).

---

#### Lever C · Lower max_tokens + Answer Rule

The current `max_tokens=4096` reserves 4K output tokens on every call whether used or not.
Average output is ~736 tokens. Reducing the ceiling reduces the reservation; adding a
brevity instruction reduces actual output.

In `run_agent()`, change both `messages.create` calls:

```python
# Change (appears twice):
max_tokens=4096
# To:
max_tokens=1024
```

Add a brevity line to `TONE_ADDENDUM` (line 19 in `agent.py`):

```python
TONE_ADDENDUM = " Answer in three sentences or fewer when no rebooking action is taken."
```

**What should move:** tokens out. Verify the gate still passes — three sentences is enough
for a refusal or a clarifying question; the hold-and-confirm shape may need 4.

---

### Step 5 — Bench after and compare

```bash
python3 bench.py --label after --stage 1 --runs 3 --cold
python3 bench.py --compare before after
```

Read the compare output. The gate requires **≥10% movement on the cost column**
($/1K calls or $/contact). If the move is under 10%, the lever did not land — re-examine
your edit before declaring it done.

### Step 6 — Walk all five shapes

```bash
python3 run.py --all --trace
```

All five must resolve correctly. If any regressed, the lever went too far.

| PNR | Shape | What good looks like |
|-----|-------|---------------------|
| K7PQ2M | Clean cancellation | Holds seat, waits for customer click — does NOT call `confirm_rebooking` |
| M3XR8T | Delay under threshold | Refuses waiver, names exact policy row |
| T9WN4C | Ambiguous missed connection | Asks clarifying question before acting |
| G2HL9V | Group of 12 | Escalates to human; escalation IS the pass |
| R8KD3F | Abuse + legal threat | Calm and helpful, no panic, no refund offer |

If a shape regresses: dial the lever back (raise max_tokens, restore a removed tool),
confirm 5/5 green, then re-bench both labels before re-comparing.

### Step 7 — Run the gate

```bash
python3 verify.py 4.1
```

The gate prints an evidence code on the last line. Bank it on the build site. It is made
from your name, so paste it exactly — do not paraphrase or truncate.

### Step 8 — Generate the client page

```bash
python3 readout.py --client
open readout-client.html
```

The panel and the client page both read your own files. Check that:
- The number shows with its unit and what it is measured against
- Before and after rows are populated from the two bench files
- Evals show from `evals/cases.json`
- The guardrail line shows (forged token refused)
- The claim from `PITCH.md` is visible

Screenshot both surfaces.

### Step 9 — The annual value math (for Marcus)

Marcus asks one question: per year, at our volume.

| Step | Arithmetic | Source |
|------|-----------|--------|
| Disruption chats/year | ~530,000 | The case's five lines |
| Humans still take 42% of in-scope chats | 530,000 × 0.58 = ~307,000 the agent can reach | Larkspur week-12 containment |
| At $0.14 a resolved contact (Larkspur's loaded number) | 307,000 × $0.14 = ~$43,000/year | Larkspur loaded number at week 7 |
| Human baseline, same contacts | 307,000 × $6.90 = ~$2,118,000/year | The case's five lines |
| The gap | ~$2,075,000/year | Subtraction — and not a saving yet |

Say the caveat on the same slide: the 223,000 chats humans still take cost ~$1,540,000/year
and do not go away. Voice is out of scope. The gap is not a saving until somebody does not
buy something — Larkspur booked theirs as 60 seasonal BPO seats not bought (~$410,000 at
50% exposure for one storm season).

Then replace $0.14 with **your own bench number** and name which one you used. `bench.py`
reports model cost only — Larkspur's loaded number was ~40% above model cost. Report it
as model cost, or say "loaded, estimated". Do not quietly call it the loaded number.

### Step 10 — Push canon

Agree who pushes before the clock runs out.

```bash
# One person only:
python3 readout.py
python3 pod_sync.py --push-canon

# Everyone else:
python3 pod_sync.py --take-canon
```

---

## Verification checklist

```
[ ] PITCH.md has Lever: cost and a caveat sentence
[ ] python3 bench.py --label before --stage 1 --runs 3 --cold   → bench-before written
[ ] One lever pulled in agent.py (only one)
[ ] python3 bench.py --label after --stage 1 --runs 3 --cold    → bench-after written
[ ] python3 bench.py --compare before after                      → ≥10% cost movement
[ ] python3 run.py --all --trace                                 → 5/5 shapes pass
[ ] python3 verify.py 4.1                                        → gate banked
[ ] python3 readout.py --client && open readout-client.html      → panel + client page screenshotted
[ ] python3 pod_sync.py --push-canon                             → canon pushed by agreed person
```

---

## Files you edit

| File | What changes |
|------|-------------|
| `PITCH.md` | Add caveat line after `Lever: cost` (Step 2) |
| `agent.py` | Pull exactly one cost lever (Step 4) |

**Never edit:** `support/`, `verify.py`, `bench.py`, `run.py`, `setup.py`,
`pod_sync.py`, `readout.py`, `eval_harness.py`
