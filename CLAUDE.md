# Larkspur disruption agent: how Claude Code behaves in this repo

This file is part of the participant's kit. It is the contract Claude Code
reads when a pod member opens `claude` here, and it is what makes `/build`
coach instead of solve. You are pairing with a participant. Read this before
you help.

## What this repo is

A Partner Basecamp build-along wrapped around one continuous case: Larkspur
Airlines, a disruption-care chat agent. The participant is building a
multi-tool agent against the Claude Messages API: what Claude is told about each
tool, and the loop that drives them, tested against real airline policy data.

**Working repo:** `claude-code-bootcamp-larkspur-agent-main`
**First-time setup:** `cp .env.example .env` (add your ANTHROPIC_API_KEY),
then `python3 setup.py --fix` to build `.venv`.

**This repo belongs to a pod, not to one person.** Several people share one
GitHub repo and every one of them has a clone of it on their own laptop.
`TEAM.md` (the pod's name and a typed roster), `PITCH.md` and
`evals/cases.json` are the pod's shared record.

**`agent.py` is not one of those.** Everyone builds their own `agent.py`
locally, and nobody commits it mid-build. One person pushes the canon at the
end of the build with `python3 pod_sync.py --push-canon`. The pod agrees who
before the clock runs out. Everyone else takes it with `--take-canon`. So the person
you are helping is building their own file, in their own words, alongside
several other people doing the same thing. Their whole job is in `agent.py`.
Everything in `support/` is given and should not be edited.

**`agent.py` is one file with six editable places and no prose.** The loop is on
top, three empty labelled slots sit above it (`TONE_ADDENDUM`, `EXTRA_TOOLS`,
`LOCAL_TOOLS`), and the nine tool schemas are below a fold. Every editable place
carries a `✏️` mark naming its build and step, and `grep -n '✏' agent.py` lists
all six. Nothing new ever arrives in the file: every later step is an edit to
something they have looked at since the first minute. The instructions live on
the build site, not in the file, so do not go looking in `agent.py` for what a
step wants.

They verify with `python3 verify.py <step>`, which checks behavior on the
wire, not code shape.

## How to help

This is an implementation session, not a training workshop. Help the team
build, fix, and verify the agent. Write code when asked, explain what you
changed and why, and paste trace output verbatim after every run.

Rules that still apply from the workshop:
- Never edit `support/`, `verify.py`, `setup.py`, `pod_sync.py`, `readout.py`,
  `bench.py`, or `eval_harness.py`.
- Never weaken `.gitignore` or `.gitattributes`.
- Never put a key in a file you would commit.
- Paste `run.py` and `verify.py` output verbatim — never summarize a trace.

## Environment notes

- Python is pinned via `.venv`; `python3 setup.py --fix` builds it.
- Keys live in `.env` (gitignored) or the environment. One key per person,
  never a pod key. Never put a key in a file you would commit, and never echo a
  key back into the transcript.
- No network means a raised hand and a podmate's screen. There is no offline
  mode in this pack, and building one on the clock is not the work.
- Commands, by the moment they belong to:
  - **Day one, before anything:** `python3 setup.py`, until it says READY.
  - **Every build:** `python3 run.py <PNR> --trace`, then
    `python3 verify.py <step>`. The steps are `1.2`, `1.3`, `1.4` (Build 1),
    `2.1`, `2.2` (Build 2), `3.1` (Build 3), `4.1` (Build 4).
  - **End of a build:** `python3 readout.py`, then the one person the pod
    agreed on runs `python3 pod_sync.py --push-canon` and everyone else runs
    `python3 pod_sync.py --take-canon`.
  - **Next session:** `python3 bench.py --label before` / `--label after`
    around the lever, and `python3 eval_harness.py` for the cases.
