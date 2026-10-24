---
name: designing-new-systems
description: Use when asked to design, architect, or plan a new system, service, or app backend before code exists, including "design X for me", "how should I structure this", or choosing databases, queues, caches, or hosting for a new project.
---

# Designing New Systems

Good designs come from confirmed requirements and real numbers, not from a list of components. Get the requirements confirmed, put numbers on the load, and pick the smallest design that meets them. Then say exactly what would make it need to change.

**Not for:** changes inside an existing codebase whose architecture is settled, or visual/UI design.

## Step 1: Ask before you design

Propose architecture, components, or a data model only after the user has answered your questions or accepted your defaults.

Send one message with **at most 10 questions**, each with your proposed default in brackets. Only ask a question if a different answer would change the architecture, the data model, or the cost. These almost always qualify:

- Scale: users who are active at the same time, and the main actions per day
- Which is worse for this system: downtime or losing data
- Monthly budget, and who builds and runs it

Add the questions this particular system turns on, such as fan-out, file sizes, ordering, or compliance. List everything else you're assuming under **Assumptions** at the end of the message.

End with: *"Reply with changes, or 'defaults' to accept all."* Leave out questions the request already answers.

If the user says to skip questions, list your defaults as **Assumptions** at the top of the design and continue.

| Thought | Reality |
|---|---|
| "The request is clear enough" | Scale and loss tolerance are almost never in the request |
| "I'll list open questions at the end" | Their answers change the design, so ask before designing |
| "I'll just assume a small scale" | Assumptions the user never saw can't be corrected |

## Step 2: Put numbers on it

Show the arithmetic for each of these:

- Average and peak rate of each main operation (reads and writes separately)
- Fan-out, if one action is delivered to many users
- Storage per year, per data type (estimate large media separately, since it usually dominates)
- Monthly cost in dollars

Shortcuts:
- 1M per month ≈ 0.4/s
- 1 day ≈ 86,400 s ≈ 100k s
- Peak ≈ 2–10× average (use 10× for bursty load)

Then **right-size**. One server with Postgres handles thousands of simple queries per second. If the load fits on that, the design starts there.

## Step 3: Write the design in this shape

Write the full design to `DESIGN.md` in the project root, or to the path the user names. If you can't write files, put the full design in your reply after the Step 4 message.

The design opens with a **summary of at most 5 lines**: the architecture in one sentence, the monthly cost, the top risk, and the decisions to confirm.

1. **Requirements and numbers.** What was confirmed, plus the arithmetic.
2. **Architecture.** The smallest design that fits, with a diagram.
3. **Data model.** Include the hot-path query and the index that serves it.
4. **Key flows.** Hot path first. Anything not needed for the user's response moves to a background worker.
5. **Data protection.**
   - A table with columns: data type | cost to lose it | RPO | mechanism.
   - An availability target in nines, and an RTO.
   - Irreplaceable data needs continuous backup (point-in-time recovery), not nightly backups.
6. **Decisions.**
   - A table with columns: choice | alternative | why | hard to reverse?
   - Mark a decision hard to reverse only if changing it later needs a data migration or breaks people outside your code. Typical examples are ID schemes, URL formats, public API or payload shapes, caching and redirect semantics, and retention promises.
   - Config values (timeouts, limits, intervals) are never hard to reverse.
7. **Failure walk.** Cover each of these:
   - A crash in the middle of an operation
   - Retries and double-submits
   - An external dependency that's down or slow
   - Single points of failure, including non-code ones: domain expiry, TLS certificates, DNS, a scheduled job that silently stops
8. **Not building yet.** Each item with the **numeric trigger** that would justify adding it.
9. **Open risks.**

Prefer tables over prose. Keep each section short.

## Step 4: Hand off

Your chat reply contains only these parts, in this order:

1. The 5-line summary
2. Where the full design is saved
3. The **3–5 most consequential** hard-to-reverse decisions, each with one line on what it would cost to change later
4. Any other hard-to-reverse decisions, as a one-line list
5. A request to confirm them all in one reply, or to say what to change, before any code is written

## Common mistakes

| Mistake | Fix |
|---|---|
| "A few thousand a day at most" with no math | Show the arithmetic |
| Nightly backups for data called irreplaceable | Set RPO by data type |
| "Revisit later" or "add when needed" | Give a number: "add a queue at more than 500 jobs/s" |
| Questions asked after the design | Step 1 |
| 15+ questions, so the user accepts defaults without reading them | At most 10. The rest go under Assumptions |
| Full design pasted into chat | Chat gets the Step 4 message. The design goes in `DESIGN.md` |

For a deeper review, walk through `references/checklist.md`.
