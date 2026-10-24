# designing-new-systems

An agent skill for designing a new system before writing code. It's plain markdown with no scripts and no dependencies.

It's tested in Claude Code and OpenAI Codex. It should work in Grok CLI, which reads the same skill format, but that hasn't been tested yet.

## What it does

Ask your agent to "design X." With this skill loaded, the agent goes through four steps.

1. **It asks before designing.** You get up to 10 questions, each with a suggested default, and only questions whose answers change the architecture, data model or cost. Everything else is listed as an assumption. Reply `defaults` to accept them all.
2. **It puts numbers on the load.** It shows the arithmetic for peak rates, fan-out, storage per data type and monthly cost, then sizes the design to fit. Often that's one server and Postgres.
3. **It writes `DESIGN.md`.** The file has 9 sections:
   - requirements and numbers
   - architecture
   - data model
   - key flows
   - data protection (RPO per data type, availability target)
   - decisions (each flagged reversible or not)
   - failure walk (domain expiry, certs and silent cron jobs included)
   - not building yet (each with a numeric trigger)
   - open risks
4. **It hands off a short chat message.** That's a 5-line summary plus the 3–5 decisions that are hardest to undo, for you to confirm before any code is written.

## Why it exists

Without the skill, current models already know a lot about system design. In our baseline runs they right-sized, used UNIQUE constraints for idempotency, and moved slow work to background jobs.

What they skipped was **process**:

- They designed without asking anything.
- Their estimates had no arithmetic ("a few thousand a day at most").
- One called photos "irreplaceable" and then chose nightly backups.
- "Add later" items had no number saying when.
- Nobody mentioned the domain expiring.

The skill targets those gaps, not general knowledge.

## Results

Scored on a 6-point rubric (see [`evals/`](evals/)):

| Scenarios | Without skill | With skill |
|---|---|---|
| Habit tracker, webhooks, photo sharing (draft 1) | 1 / 18 | 18 / 18 |
| Theater ticketing, greenhouse sensors, unseen while writing (draft 3) | ~2.5 / 12 (baseline from draft 2 run) | 12 / 12 |

**On OpenAI Codex** (theater scenario, draft 3):

| Run | Score | Output |
|---|---|---|
| CLI v0.164.0 (`gpt-6.1-sol`), **no skill** | **~0.5 / 6** | 1,167-word chat answer, no questions asked |
| CLI v0.164.0 (`gpt-6.1-sol`), with skill | **6 / 6** | 7 questions, then a 6,348-word `DESIGN.md` and a 181-word hand-off |
| CLI v0.155.1 (`gpt-6-astra`), with skill | **6 / 6** | 7 questions, then a 3,209-word `DESIGN.md` and a 154-word hand-off |

- With the skill installed, Codex picked it up on its own from the description in both versions.
- Without it, the same model's design was solid on correctness (seat locking, payment races, idempotent webhooks), but it skipped the same process steps the Claude baselines did.
- Details: [baseline](evals/codex-baseline-2026-10-24.md), [v0.164.0](evals/codex-v2-2026-10-24.md), [v0.155.1](evals/codex-2026-10-24.md).

**Limits:**
- n = 1 per scenario.
- Scored by the author, not an independent grader.
- Raw outputs for the later runs are in [`evals/runs/`](evals/runs/), so you can check the scoring yourself.

## Install

Clone the repo once:

```bash
git clone https://github.com/GcdZ03/system-design-skill.git ~/system-design-skill
```

Then link the skill folder into each agent you use.

### Claude Code

```bash
mkdir -p ~/.claude/skills
ln -s ~/system-design-skill/skills/designing-new-systems ~/.claude/skills/designing-new-systems
```

For a single project, link it into `<project>/.claude/skills/` instead. Restart Claude Code, then check with `/skills`.

### OpenAI Codex

```bash
mkdir -p ~/.agents/skills
ln -s ~/system-design-skill/skills/designing-new-systems ~/.agents/skills/designing-new-systems
```

For a single project, use `<project>/.agents/skills/`.

### Grok CLI

Grok CLI reads `~/.agents/skills/`, so the Codex link above covers it. It also reads `~/.grok/skills/` if you prefer that location.

**Not tested by the author yet.**

### Anything else (chat apps, other agents)

Paste the contents of `skills/designing-new-systems/SKILL.md` into the system prompt or project instructions. If the agent can't write files, it puts the full design in chat after the summary.

### Updating

```bash
cd ~/system-design-skill && git pull
```

The symlinks pick up the change automatically.

## Using it

Just describe what you want to build:

> Design a ticketing system for our 300-seat community theater. Shows sell out within an hour.

The skill triggers on requests to design, architect or plan a new system, or to choose databases, queues or hosting for a new project. Answer the questions, or reply `defaults`. Then review `DESIGN.md` and confirm the hard-to-reverse decisions.

## Repo layout

```
skills/designing-new-systems/
  SKILL.md                 the skill (~930 words)
  references/checklist.md  deeper review checklist, read on demand
evals/                     scenarios, rubric, scored runs, raw outputs
notes/                     the learning sessions this skill was distilled from
```

## How it was made

The author worked through five short design sessions (link shortener, reliability, async analytics, real-time chat, bulk email jobs) and distilled 25 rules from them. The rules are in [`notes/`](notes/).

From there:
1. Baseline runs showed which rules models already follow.
2. The skill was written for the rest.
3. Each draft was tested against baselines, including scenarios that weren't used to write it.
